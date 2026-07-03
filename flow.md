## End-to-End Flow Script: CDN IP Rotation to Zero-Disruption Removal

---

### Scene 1: The Starting State

Before anything happens, here's what's running:

```
ACTORS:
  - User "Alice"        → browsing app.customer.com from her laptop
  - CloudFront Edge A   → IP: 20.21.22.5 (serving Alice's requests)
  - CloudFront Edge B   → IP: 9.8.7.6 (exists but not serving Alice yet)
  - Your ALB            → IP: 51.83.11.105 (Envoy on bare-metal node)
  - Cilium eBPF         → enforcing loadBalancerSourceRanges on the node

CURRENT STATE:
  loadBalancerSourceRanges: [1.2.3.0/24, 5.6.7.0/24, 9.10.11.0/24, 20.21.22.0/24]
  ConfigMap pending_removal: []  (empty, nothing being drained)

  Alice's DNS cache: app.customer.com → 20.21.22.5 (TTL: 60 seconds)

TRAFFIC FLOW:
  Alice (100.100.100.100) → Edge A (20.21.22.5) → ALB (51.83.11.105) → Envoy → Backend
  Cilium sees: src=20.21.22.5, verdict=FORWARDED ✓
```

---

### Scene 2: CloudFront Decides to Retire Edge A

**T = 0 minutes.** AWS internally decommissions Edge A. Two things happen simultaneously at CloudFront's end:

```
ACTION 1: CloudFront DNS stops returning 20.21.22.5 for app.customer.com.
          New DNS queries will get 9.8.7.6 (Edge B) instead.

ACTION 2: CloudFront publishes updated ip-ranges.json to SNS topic.
          20.21.22.0/24 is ABSENT from the new list.
          SNS sends notification to your webhook.
```

---

### Scene 3: Your Webhook Receives the Notification

**T = 0 minutes + a few seconds.** The CloudFront Webhook pod in `kube-system` receives the SNS notification.

```
WEBHOOK ACTIONS:

Step 1 — Fetch new IP list from ip-ranges.json:
  New list: [1.2.3.0/24, 5.6.7.0/24, 9.10.11.0/24, 99.98.97.0/24]

Step 2 — Read existing ConfigMap (cloudfront-ip-cache):
  Old list: [1.2.3.0/24, 5.6.7.0/24, 9.10.11.0/24, 20.21.22.0/24]

Step 3 — Compute diff:
  Added:     99.98.97.0/24     (new CloudFront edge range)
  Removed:   20.21.22.0/24     (Edge A's range — retired)
  Unchanged: 1.2.3.0/24, 5.6.7.0/24, 9.10.11.0/24

  Diff is NOT empty → proceed to sanity gate.
```

---

### Scene 4: Sanity Gate

**T = 0 minutes + a few seconds.** Before writing the new list, the webhook validates it.

```
SANITY CHECKS:

Check 1 — Overlap ≥ 50%?
  Old list: 4 CIDRs.  New list: 4 CIDRs.  Overlap: 3 CIDRs = 75%.
  75% ≥ 50% → ✓ PASS

Check 2 — New list non-empty?
  4 CIDRs → ✓ PASS

Check 3 — Contains known floor CIDRs?
  (CloudFront always has certain ranges, like 13.32.0.0/15)
  Present → ✓ PASS

All checks passed. Proceed to ConfigMap update.

  ❌ IF ANY CHECK FAILED:
     - Do NOT overwrite the ConfigMap
     - Update last_fetch_attempt only
     - Increment cdn_ip_cache_sanity_reject_total counter
     - Alert: possible API corruption or MITM
```

---

### Scene 5: ConfigMap Update

**T = 0 minutes + a few seconds.** The webhook writes the new state.

```
CONFIGMAP: cloudfront-ip-cache (AFTER UPDATE)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ipv4_cidrs: ["1.2.3.0/24","5.6.7.0/24","9.10.11.0/24","99.98.97.0/24"]
  ipv6_cidrs: [...]
  last_successful_fetch: "2026-07-03T14:00:05Z"
  last_fetch_attempt:    "2026-07-03T14:00:05Z"
  source_etag:           "W/\"def456\""
  fetch_source:          "aws-ip-ranges-v1"
  pending_removal:
    - cidr: "20.21.22.0/24"
      origin: "cloudfront"          ← origin tag prevents cross-contamination
      deprecated_at: "2026-07-03T14:00:05Z"
      remove_after:  "2026-07-03T16:00:05Z"    ← T + 2 hours
      extensions: 0
      last_flow_seen_at: null

KEY POINT: Only 20.21.22.0/24 went to pending_removal.
           The other 3 CIDRs are untouched.
           If this tenant also has whitelist-cloudflare=enabled,
           the Cloudflare entries are completely unaffected
           because of the origin tag.
```

---

### Scene 6: Reconciler Patches EnvoyProxy

**T = 0 minutes + a few seconds.** The webhook calls the shared reconciler.

```
RECONCILER (internal/iprecon):

Step 1 — List EnvoyProxy resources:
  LabelSelector: vishanti.io/whitelist-cloudfront=enabled
  Found: envoyproxy/tenant-abc, envoyproxy/tenant-xyz, ... (all opted-in tenants)

Step 2 — Compute UNION for each resource:
  active CIDRs:          [1.2.3.0/24, 5.6.7.0/24, 9.10.11.0/24, 99.98.97.0/24]
  + pending_removal:     [20.21.22.0/24]  (not expired yet)
  ─────────────────────────────────────
  = UNION:               [1.2.3.0/24, 5.6.7.0/24, 9.10.11.0/24, 99.98.97.0/24, 20.21.22.0/24]
                                                                                 ^^^^^^^^^^^^^^^
                                                               still allowed during 2h drain window

Step 3 — Idempotency check:
  Compare UNION with current loadBalancerSourceRanges on EnvoyProxy.
  If identical → skip patch (no-op).
  If different → proceed.

Step 4 — JSON Merge Patch:
  PATCH envoyproxies/tenant-abc
  {"spec":{"provider":{"kubernetes":{"envoyService":
    {"loadBalancerSourceRanges":["1.2.3.0/24","5.6.7.0/24","9.10.11.0/24","99.98.97.0/24","20.21.22.0/24"]}
  }}}}

Step 5 — Cilium detects Service change:
  eBPF LPM trie map updated in MILLISECONDS.
  20.21.22.0/24 is still in the map → packets from Edge A still FORWARDED.
```

---

### Scene 7: DNS Propagation (Happens in Parallel — Not Our Action)

**T = 0 to T = ~3 minutes.** This is happening on the internet side, completely independent of our reconciler.

```
T = 0 min 00s:
  Alice's browser has cached: app.customer.com → 20.21.22.5 (TTL: 55s remaining)
  Alice clicks a link → request goes to Edge A (20.21.22.5) → your ALB
  Cilium: src=20.21.22.5 in 20.21.22.0/24 → FORWARDED ✓
  Alice sees the page. No disruption.

T = 0 min 55s:
  Alice's DNS TTL expires.
  Browser discards cached entry for app.customer.com.

T = 1 min 00s:
  Alice clicks another link.
  Browser asks DNS: "What is app.customer.com?"
  DNS responds: "9.8.7.6"  ← Edge B (CloudFront no longer returns 20.21.22.5)

  Alice's browser opens NEW connection to Edge B (9.8.7.6).
  Edge B opens origin connection to your ALB (51.83.11.105).
  Cilium: src=9.8.7.6 in 9.10.11.0/24 → FORWARDED ✓
  Alice sees the page. No disruption.

  ┌──────────────────────────────────────────────────────┐
  │ Alice is NOW on Edge B. She will NEVER contact       │
  │ Edge A (20.21.22.5) again. The DNS TTL did the work. │
  └──────────────────────────────────────────────────────┘

T = ~3 min:
  Edge A (20.21.22.5) has no more users hitting it for this origin.
  Its idle origin connection to 51.83.11.105 times out and closes.
  No more packets from 20.21.22.5 → 51.83.11.105.

  Hubble flow log for 20.21.22.5 goes silent.
```

---

### Scene 8: The 2-Hour Wait

**T = 0 to T = 2 hours.** Nothing happens on our side. We're waiting.

```
WHY 2 HOURS?

  DNS TTL is 60 seconds for most resolvers.
  But the internet has billions of resolvers:
    - Some corporate proxies cache for 30 min
    - Some ISP resolvers cache for 1 hour
    - Some broken resolvers ignore TTL entirely

  2 hours covers the long tail of legitimate DNS stragglers.

DURING THIS TIME:
  - 20.21.22.0/24 remains in loadBalancerSourceRanges (union)
  - Any straggler resolver still pointing users at 20.21.22.5 works fine
  - The CIDR is in the eBPF map → packets are FORWARDED
  - No user sees any error

  Alice? She moved to Edge B at T = 1 minute. She's been fine for 2 hours.
```

---

### Scene 9: First Hubble Check

**T = 2 hours.** `remove_after` timestamp reached. Reconciler wakes up and checks.

```
HUBBLE CAPABILITY CHECK:
  Reconciler dials hubble-relay gRPC → connection succeeds → mode = "enabled"

  (If Hubble was down: mode = "unavailable" → skip Hubble, use fixed 4h drain, remove at T+4h)
  (If HUBBLE_DISABLED=true: mode = "disabled" → skip Hubble, use fixed 2h drain, remove now)

HUBBLE QUERY:
  GetFlows({
    source:  ["20.21.22.0/24"],
    dest:    ["51.83.11.105"],
    verdict: [FORWARDED],         ← CRITICAL FILTER
    since:   now - 5 minutes
  })

WHY verdict=FORWARDED?
  Cilium stamps every packet:
    - FORWARDED = allowed through to Envoy (real traffic)
    - DROPPED = blocked by eBPF (not reaching Envoy)

  Without filter:
    If someone is STILL trying to connect from 20.21.22.5 but we
    already removed the CIDR, their packets show as DROPPED.
    Hubble would say "flows found!" → extend timer → WRONG.
    We'd be extending the timer for traffic that's already being blocked.

  With verdict=FORWARDED filter:
    Only real, allowed traffic counts. Blocked traffic is invisible.

WHY 5 MINUTES (not 1 second)?
  HTTP traffic is bursty:
    |████  ██   ███     █  ████   ██       ████  █|
                              ^
                         3-min gap (normal user reading a page)

  A 1-second check during a natural gap → "no traffic" → premature removal.
  A 5-minute check → sees the surrounding bursts → "still active."
  If truly zero FORWARDED flows in 5 full minutes → no users, no keepalives,
  no active connections. Genuinely idle.

RESULT (at T = 2 hours):
  Hubble response: 0 FORWARDED flows in last 5 minutes.

  Last real traffic from 20.21.22.5 was at T = ~3 minutes (Scene 7).
  That was 1 hour 57 minutes ago. The edge has been completely silent.
```

---

### Scene 10: Decision — Remove

**T = 2 hours.** No traffic found. Safe to remove.

```
DECISION TABLE:
  ┌─────────────────────────┬──────────────────────┬─────────────┐
  │ Hubble Result           │ Action               │ Next Check  │
  ├─────────────────────────┼──────────────────────┼─────────────┤
  │ No FORWARDED flows      │ ✅ REMOVE CIDR        │ None (done) │ ← THIS ONE
  │ FORWARDED flows found   │ 🔴 Extend +30 min    │ In 30 min   │
  │ LostEvents > 0          │ 🟡 Extend +30 min    │ In 30 min   │
  │ Hubble error/down       │ 🟡 Extend +30 min    │ In 30 min   │
  └─────────────────────────┴──────────────────────┴─────────────┘

  Result: NO FORWARDED FLOWS → REMOVE.

REMOVAL STEPS:
  1. Delete 20.21.22.0/24 from pending_removal in ConfigMap
  2. Recompute UNION: [1.2.3.0/24, 5.6.7.0/24, 9.10.11.0/24, 99.98.97.0/24]
                       ← 20.21.22.0/24 is GONE
  3. Patch EnvoyProxy with updated loadBalancerSourceRanges
  4. Cilium eBPF map updated (milliseconds)
  5. Any future packet from 20.21.22.0/24 → DROPPED in kernel

  Metrics emitted:
    cdn_pending_removal_cidrs{origin="cloudfront"} = 0
```

---

### Scene 11: What If Traffic Was Still Flowing? (Alternate Path)

**If Hubble had found FORWARDED flows at T = 2 hours** (rare, but possible with a very slow resolver):

```
T = 2h 00min:
  Hubble: "3 FORWARDED flows from 20.21.22.5 in last 5 minutes"
  Reconciler: "Active traffic. Extend."

  ConfigMap updated:
    pending_removal:
      - cidr: "20.21.22.0/24"
        origin: "cloudfront"
        remove_after: "2026-07-03T16:30:05Z"    ← T + 2h 30min
        extensions: 1                            ← incremented
        last_flow_seen_at: "2026-07-03T16:00:05Z"

T = 2h 30min:
  Hubble check again. "0 FORWARDED flows." → REMOVE.

  OR if still active → extend to T + 3h 00min (extensions: 2)

  This repeats up to 8 times:
    T+2h 00min  Check 1 (extensions: 0 → 1)
    T+2h 30min  Check 2 (extensions: 1 → 2)
    T+3h 00min  Check 3 (extensions: 2 → 3)
    T+3h 30min  Check 4 (extensions: 3 → 4)
    T+4h 00min  Check 5 (extensions: 4 → 5)
    T+4h 30min  Check 6 (extensions: 5 → 6)
    T+5h 00min  Check 7 (extensions: 6 → 7)
    T+5h 30min  Check 8 (extensions: 7 → 8)
    T+6h 00min  MaxExtensions = 8 reached → FORCE REMOVE
```

---

### Scene 12: Force Removal (Worst Case Only)

**T = 6 hours.** MaxExtensions reached. This means traffic from 20.21.22.0/24 has been seen continuously for 6 hours after CloudFront retired it. Extremely unlikely.

```
FORCE REMOVAL:
  1. Remove 20.21.22.0/24 from pending_removal and loadBalancerSourceRanges
  2. Fire alert webhook: cdn_forced_removal_total{origin="cloudfront"} += 1
  3. Page on-call: "CIDR 20.21.22.0/24 force-removed after 8 extensions"

WHAT HAPPENS TO STRAGGLERS?
  If a broken resolver is STILL pointing a user at Edge A (20.21.22.5):
    - Edge A tries to open new origin connection to 51.83.11.105
    - Cilium eBPF: src=20.21.22.5 NOT in LPM map → DROPPED
    - CloudFront's health-check detects: origin unreachable from Edge A
    - CloudFront internally reroutes: user's next request goes via Edge B
    - User sees: one failed request, then everything works
    - Recovery time: seconds

  This only affects clients with BROKEN DNS resolvers that ignore TTLs
  after 6 full hours. Not a normal user. Force removal is correct.
```

---

### Scene 13: Final State

```
AFTER REMOVAL:

  loadBalancerSourceRanges: [1.2.3.0/24, 5.6.7.0/24, 9.10.11.0/24, 99.98.97.0/24]
  pending_removal: []  (empty again)

  Alice:     On Edge B (9.8.7.6) since T = 1 minute. Never noticed anything.
  All users: On current edges. No one is being sent to 20.21.22.5.
  20.21.22.5: Retired. Not in DNS. Not in eBPF map. Fully decommissioned.

  ┌────────────────────────────────────────────────────────────────┐
  │ KEY INSIGHT:                                                    │
  │                                                                 │
  │ The zero-disruption design does NOT keep the CIDR alive until  │
  │ the user comes back. It keeps it alive until the CDN's own     │
  │ DNS-driven traffic drain is complete. After that, no user is   │
  │ being sent to the retired edge. Whether the user returns in    │
  │ 5 minutes or 5 years, DNS will point them to whatever edge is  │
  │ current — and your eBPF map will match.                        │
  └────────────────────────────────────────────────────────────────┘
```

---

### Quick Reference: The Entire Flow in One View

```
CDN retires IP
    │
    ├─ Webhook/CronJob detects it
    ├─ Sanity gate validates new list (≥50% overlap, non-empty, floor CIDRs)
    ├─ ConfigMap updated (removed CIDRs → pending_removal with origin tag)
    ├─ Reconciler computes UNION (active + pending), patches EnvoyProxy
    ├─ Cilium eBPF map updated (milliseconds)
    │
    ├─ [Meanwhile on the internet] DNS TTL expires → users move to new edge (60s)
    │
    ├─ Wait 2 hours (DNS propagation tail)
    │
    ├─ Query Hubble: FORWARDED flows from deprecated CIDR in last 5 min?
    │   ├─ No traffic     → REMOVE (done)
    │   ├─ Traffic found  → extend +30min (max 8 times)
    │   └─ Hubble down    → fallback to fixed 4h drain
    │
    ├─ MaxExtensions (8) reached at T+6h → FORCE REMOVE + alert
    │
    └─ Result: zero disruption for all users with working DNS
```
