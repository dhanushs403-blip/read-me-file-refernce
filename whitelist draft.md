# Phase 3 — Mentor Discussion: Cloudflare & CloudFront L3/L4 Security Whitelisting

**Date:** 2026-07-06
**Author:** Dhanush
**Status:** Pre-implementation — awaiting mentor review before coding begins

---

## 1. What Are We Building and Why

### The Problem

When a customer routes traffic through Cloudflare or CloudFront, the CDN sits in front of our cluster's Application Load Balancer (ALB). From the ALB's perspective, all requests arrive from CDN IP addresses — not from end-user IPs. Without whitelisting, the ALB accepts connections from *any* IP, including direct bypass attackers.

Two risks if we do nothing:

1. **DDoS bypass:** An attacker who discovers the ALB's real IP address can send volumetric traffic directly, bypassing Cloudflare/CloudFront's edge protection entirely.
2. **Unauthorized direct access:** A request that should go through the CDN (and its WAF rules, rate limiting, caching) can reach the origin directly.

### The Solution

Restrict the ALB's accepted source IPs to *only* the published CIDR ranges of Cloudflare and/or CloudFront. Traffic from any other IP is dropped at the kernel level before Envoy processes a single byte.

This is opt-in — customers who don't use a CDN are unaffected.

---

## 2. Where Enforcement Actually Happens (Critical Correction)

### What We Originally Assumed (Wrong)

The initial design assumed enforcement happened via the **Cloud Controller Manager (CCM)** — i.e., a cloud provider firewall or security group configured by CCM when `loadBalancerSourceRanges` is set on a Kubernetes Service.

### What the Cluster Actually Uses (Correct)

Our cluster has **no CCM**. There is no `cloud-controller-manager` pod and nodes have no `providerID`. We verified this directly:

```bash
kubectl get pods -A | grep cloud-controller   # no results
kubectl get nodes -o yaml | grep providerID   # empty
```

Enforcement is performed by **Cilium eBPF** — specifically the `cilium_lb4_source_range` LPM (Longest Prefix Match) trie map inside the kernel on each host node. When `loadBalancerSourceRanges` is set, Cilium populates this map. Packets from non-matching source IPs are dropped at the **XDP/tc ingress hook**, in the kernel, in nanoseconds — Envoy never sees them.

Current state of the map (confirmed from cluster):

```
cilium_lb4_source_range → 0 entries   # no whitelisting applied yet
```

Phase 3 will populate this map for opted-in ALBs.

---

## 3. ALB vs NLB — Phase 3 Scope Boundary

Our platform exposes two load balancer types. Phase 3 only touches one of them.

| | NLB (Network Load Balancer) | ALB (Application Load Balancer) |
|---|---|---|
| **K8s resource** | `Service` (type: `LoadBalancer`) | Gateway API `Gateway` resource |
| **Gateway class** | — | `gatewayClassName: "cilium"` |
| **Proxy** | None — raw TCP/UDP pass-through | Envoy pod (managed by Envoy Gateway) |
| **TLS** | Not terminated | Terminated at Envoy (cert-manager) |
| **Protocol** | L4 — raw TCP/UDP | L7 — HTTP/1.1 or **HTTP/2 via ALPN** |
| **Connection to backend** | Direct pod (Cilium L4 IPAM) | Envoy → separate connection → pod |
| **Phase 3 in scope?** | **No** | **Yes** |

**Why this matters:**

- Phase 3 patches `loadBalancerSourceRanges` on the **EnvoyProxy** CRD. This resource only exists for ALBs. NLBs have no EnvoyProxy resource.
- A customer using Cloudflare in front of an **NLB** gets no protection from Phase 3. If they are attacked directly, packets hit the backend pod. This is a gap to acknowledge.

**Confirmed: HTTP/2 on ALBs.** The ALB uses HTTP/2 via ALPN negotiation, confirmed in the OpenAPI spec and cluster setup. CloudFront also uses HTTP/2 for origin connections by default. This means the CDN edge → ALB connection uses HTTP/2 multiplexed streams. CloudFront recycles origin connections approximately every 60 seconds. This is important for the drain window design (Section 7).

---

## 4. Component Architecture

### Four Components

```
┌─────────────────────────────────────────────────────────────┐
│  VSO (Control Plane — OVH/OpenStack host)                   │
│                                                              │
│  orch_api_server (FastAPI)                                   │
│    └─ on toggle enable/disable → inline call to iprecon     │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ Kubernetes API
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  VSN (Tenant Cluster — runs on OVH/OpenStack)               │
│                                                              │
│  Cloudflare CronJob (Go, runs every 24h)                    │
│    └─ polls api.cloudflare.com/client/v4/ips                │
│    └─ updates cloudflare-ip-cache ConfigMap                 │
│    └─ calls iprecon for all opted-in EnvoyProxies           │
│                                                              │
│  CloudFront Webhook (Go, Deployment)                        │
│    └─ receives AmazonIpSpaceChanged SNS notification        │
│    └─ fetches ip-ranges.amazonaws.com/ip-ranges.json        │
│    └─ calls iprecon for all opted-in EnvoyProxies           │
│                                                              │
│  internal/iprecon (shared Go package)                       │
│    └─ discovery via LabelSelector                           │
│    └─ ConfigMap cache read                                  │
│    └─ union computation (active ∪ pending_removal)          │
│    └─ Hubble telemetry query                                │
│    └─ JSON Merge Patch on EnvoyProxy                        │
│    └─ Prometheus metrics                                    │
└─────────────────────────────────────────────────────────────┘
```

### Deployment

Mirrors the existing `dns_controller` Ansible role pattern:

```
Ansible role: ip_whitelist_controller
  ├── tasks/main.yml
  ├── tasks/remove.yml
  └── templates/
      ├── cloudflare-cronjob.yaml.j2
      └── cloudfront-webhook.yaml.j2
```

Controlled by `ip_whitelist_enabled` flag in `deploy.yml`.

---

## 5. Opt-In Signal Design — Two Labels, One EnvoyProxy

Whitelisting uses **two independent Kubernetes labels** on the EnvoyProxy resource.

### Label 1 — Provider (routing intent, set automatically)

```
vishanti.io/provider=cloudflare
vishanti.io/provider=cloudfront
```

Set by Orchestrator when customer selects a CDN provider. Indicates which CDN is in front of this ALB. Does **not** enable whitelisting.

### Label 2 — Whitelist (enforcement intent, opt-in only)

```
vishanti.io/whitelist-cloudflare=enabled
vishanti.io/whitelist-cloudfront=enabled
```

Set only when customer explicitly enables L3/L4 whitelisting. Both can coexist on one EnvoyProxy (customer uses both CDNs simultaneously).

**Why labels and not annotations?** The CronJob and Webhook use `LabelSelector` in `ListOptions` for server-side filtering. Annotations cannot be filtered by the Kubernetes API server — filtering would require fetching all resources and filtering client-side, which is expensive and fragile.

### Patch Target

```
Resource GVR: gateway.envoyproxy.io/v1alpha1 / envoyproxies
Patch path:   spec.provider.kubernetes.envoyService.loadBalancerSourceRanges
Patch type:   JSON Merge Patch via dynamic Kubernetes client
```

---

## 6. IP Cache — ConfigMap Schema

Each CDN has its own ConfigMap. Example for Cloudflare:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflare-ip-cache
  namespace: vishanti-system
data:
  ipv4_cidrs: '["103.21.244.0/22","103.22.200.0/22",...]'
  ipv6_cidrs: '["2400:cb00::/32","2606:4700::/32",...]'
  last_successful_fetch: "2026-07-06T10:15:00Z"
  last_fetch_attempt:    "2026-07-06T10:15:00Z"
  source_etag:           'W/"abc123"'
  fetch_source:          "cloudflare-api-v4"
  pending_removal: |
    [
      {
        "cidr":       "20.21.22.0/24",
        "origin":     "cloudfront",
        "removed_at": "2026-07-06T09:00:00Z",
        "extensions": 2
      }
    ]
```

### `pending_removal` — Origin-Tagged (Multi-CDN Safety)

The `pending_removal` list is keyed by `(origin, cidr)` — not just CIDR. This prevents cross-contamination when a customer has **both** `whitelist-cloudflare=enabled` and `whitelist-cloudfront=enabled`:

- Cloudflare CronJob only reads and writes entries where `origin = "cloudflare"`.
- CloudFront Webhook only reads and writes entries where `origin = "cloudfront"`.

Without origin-tagging, a Cloudflare CronJob could prematurely remove a CloudFront CIDR that was still in its drain window.

---

## 7. Zero-Disruption IP Rotation — End-to-End Flow

This is the most complex part of Phase 3. Let me walk through it step by step.

### What Triggers a Rotation

Cloudflare publishes a new IP range list. The CronJob (or CloudFront SNS webhook) detects a diff between the cached list and the new list. Some CIDRs were in the old list but are absent from the new list.

### Step 1 — Union Write (Immediate)

Removed CIDRs do not get deleted immediately. Instead:

```
new_list         = [A, B, C]         (from CDN's new published list)
old_list         = [A, B, C, D, E]   (from ConfigMap cache)
removed          = [D, E]            (old_list minus new_list)

pending_removal  = D, E              (added with timestamp now+15m)
active_ranges    = new_list ∪ pending_removal = [A, B, C, D, E]
```

The full union `[A, B, C, D, E]` is written to `loadBalancerSourceRanges` on all opted-in EnvoyProxies. Nothing is removed. The CDN's new IPs are now also active. From this point, both old and new CDN IPs can reach the ALB.

### Step 2 — Initial Drain Window (15 Minutes)

Why 15 minutes before checking if the old CIDRs can be removed?

**DNS TTL analysis:**
- Cloudflare CDN edge-to-origin TTL: 60–300 seconds
- CloudFront CDN edge-to-origin TTL: 60–900 seconds
- Worst-case TTL: 900 seconds = 15 minutes

When a CDN rotates its edge IP (e.g., from `1.2.3.4` to `5.6.7.8`):
1. Old edge IP `1.2.3.4` stops receiving new user sessions.
2. DNS TTL governs how long DNS caches point to the old edge IP — this only affects new connection establishment.
3. During TTL propagation, some new users may still resolve to `1.2.3.4` and connect through it.
4. After TTL expires, all new connections go through `5.6.7.8`.

The 15-minute initial delay covers the full worst-case TTL propagation window. Once it passes, all new user connections are guaranteed to flow through new CDN IPs. Old CDN IPs (`1.2.3.4`, represented by CIDR `D`) may still carry **existing** connections established before the rotation.

**Note on the original design:** The CONTEXT.md shows a 2-hour initial delay and 30-minute extensions. These values were re-analyzed during this design review and corrected. See Section 10 (Design Corrections) for details.

### Step 3 — Telemetry Check via Hubble (Every 5 Minutes After Delay)

After the 15-minute initial delay, on each reconcile cycle, the reconciler queries Hubble for active flows from the removed CIDR:

```go
req := &observer.GetFlowsRequest{
    Whitelist: []*flow.FlowFilter{{
        Source:  []string{"20.21.22.0/24"},   // the removed CIDR
        Verdict: []flow.Verdict{flow.Verdict_FORWARDED},
    }},
    Since: timestamppb.New(time.Now().Add(-5 * time.Minute)),
}
```

**Why `verdict=FORWARDED` only?** If we filtered for all verdicts including `DROPPED`, we would see packets that are already being dropped (from the old IP that no longer belongs to the CDN). Those dropped packets are not active connections — they are likely probes or retries. Counting them would trigger false extensions. We only care about packets that are actually being accepted and forwarded.

**Why 5-minute lookback?** Hubble gRPC is a direct in-cluster query to `hubble-relay`. It is sub-second latency, not a Prometheus scrape pipeline. A 5-minute window is a sufficient lookback for determining whether active flows exist.

### Step 4 — Extension or Removal Decision

| Condition | Action |
|---|---|
| Active FORWARDED flows found for removed CIDR | Extend by 15 minutes (connection still active) |
| `LostEvents > 0` in Hubble response | Extend by 15 minutes (ring buffer overflow — cannot confirm no flows) |
| No flows found | Safe to remove — delete from `pending_removal` and `loadBalancerSourceRanges` |
| Extension count reaches `MaxExtensions` | Force-remove; fire alert webhook |

### Step 5 — Force Removal at MaxExtensions Ceiling

A hard ceiling prevents unbounded CIDR accumulation, which would exhaust Cilium's source range rule capacity. When `MaxExtensions` is reached, the CIDR is removed regardless of Hubble findings. An alert is fired.

**Open question for mentor (Section 9):** What should `MaxExtensions` be set to? This depends on whether ALBs can serve WebSocket connections. See Section 9.

### Hubble Fallback — Three Modes

```
HUBBLE_DISABLED=false (env var set to disable Hubble intentionally)
  → Strategy: fixed 15-minute removal (no extensions)
  → Use when: Hubble not installed, dev/test clusters

Hubble unavailable (gRPC dial fails after 3 retries)
  → Strategy: conservative fixed 30-minute retention
  → Use when: Hubble pod temporarily down, network issue

Hubble enabled and reachable (default)
  → Strategy: telemetry-aware extensions up to MaxExtensions ceiling
```

---

## 8. Staleness SLO — Cache Freshness Enforcement

The ConfigMap cache can go stale if the Cloudflare API or AWS SNS delivery is disrupted for an extended period. Writing a stale IP list as a whitelist could block legitimate CDN traffic.

### Freshness Gates (on every reconciliation)

| Check | Threshold | Action |
|---|---|---|
| Cache age warning | > 36 hours | Prometheus alert fires; reconciliation proceeds |
| Cache age critical | > 48 hours | Prometheus alert fires; reconciliation proceeds with warning log |
| Cache age fail-closed | > 72 hours | Refuse to remove any CIDRs; only additions allowed |

**Sanity gate — reject write if:**
- New list is empty
- Overlap between new list and cached list is < 50% (suggests a corrupt or truncated fetch)
- New list is missing known floor CIDRs (e.g., `103.21.244.0/22` for Cloudflare)

### Prometheus Metric

```
cdn_ip_cache_age_seconds{provider="cloudflare"}
cdn_ip_cache_age_seconds{provider="cloudfront"}
```

---

## 9. Open Questions for Mentor

These are blockers or decisions that require your input before implementation begins.

---

### Q1 — WebSocket Connections on ALBs: Confirmed or Not?

**Context:**

Our platform supports two load balancer types:
- **NLB** — raw TCP pass-through, no Envoy, WebSocket flows transparently (L4 byte stream)
- **ALB** — Envoy terminates TLS, speaks HTTP/2 via ALPN, separate connection to backend pod

Phase 3 only patches ALBs (EnvoyProxy CRD). The drain window concern — long-lived connections outlasting the retention window — only applies to connections through ALBs.

We confirmed that `SocketIoDemo.js` connects to port `8080`, which is a raw NLB endpoint (not an ALB). This suggests current WebSocket usage routes through NLBs, not ALBs.

**However,** Envoy does support WebSocket upgrades at L7. An HTTP `Upgrade: websocket` request on port 443 through an ALB would establish a long-lived WebSocket session through Envoy.

**Question:** Do any customers currently, or are they expected to, run WebSocket applications behind ALBs (port 443, TLS-terminated via Envoy)?

- **If NO** — HTTP/2 multiplexed streams (CloudFront recycles every ~60s) are the longest connection type. MaxExtensions = 1 or 2 is sufficient. Total retention: 30–45 minutes.
- **If YES** — WebSocket sessions have no timeout configured anywhere (no `idle_timeout`, no `stream_idle_timeout`, no `ClientTrafficPolicy` on Envoy Gateway). A session can remain open indefinitely. MaxExtensions = 8 (current design) is needed, but even then, an extremely long-lived WebSocket could be disrupted at the ceiling.

**Follow-up if YES:** Should we configure a `ClientTrafficPolicy` idle timeout on the ALB as a Phase 3 prerequisite? This would bound WebSocket lifetime and make MaxExtensions a predictable ceiling.

---

### Q2 — MaxExtensions Value

**Context:**

Current CONTEXT.md has `MaxExtensions = 8` which was originally designed assuming WebSocket support on ALBs.

With HTTP/2 ALPN confirmed on ALBs and CloudFront recycling origin connections every ~60 seconds, the longest residual connection through a retired CDN edge IP to our ALB is bounded — unless WebSocket sessions on ALBs exist (see Q1).

**Question:** What should MaxExtensions be?

| Scenario | Recommended MaxExtensions | Total Retention |
|---|---|---|
| No WebSocket on ALBs | 2 | ~45 minutes |
| WebSocket on ALBs, with ClientTrafficPolicy timeout | 4–6 | 1h15m – 1h45m |
| WebSocket on ALBs, no timeout (current state) | 8 (arbitrary ceiling) | 2h15m (but may still disrupt) |

---

### Q3 — CloudFront IP Rotation: No Drain Guarantee

**Context:**

Cloudflare's published FAQ states that IP changes are communicated to customers before going into production — customers are notified before a new IP goes live. This means the new Cloudflare edge IP is added to our whitelist before it starts serving traffic.

**CloudFront has no equivalent documented guarantee.** The `AmazonIpSpaceChanged` SNS topic notifies us when the change has already happened — after new IPs are already live. There is no documented cooling period between announcement and production use.

This creates a risk window where:
1. CloudFront rotates an edge IP.
2. SNS notification arrives in our webhook.
3. We add the new CIDR to `loadBalancerSourceRanges`.
4. **But between step 1 and step 3**, the new CloudFront IP is already serving traffic to our ALB — and our whitelist does not yet include it, so Cilium drops those packets.

The window depends on SNS-to-Webhook delivery latency (~seconds to minutes) plus our reconciliation time.

**Additionally:** The default CloudFront approach for origin identification uses **path prefixes on CloudFront distributions** (not external load balancers). This allows CloudFront to route to a specific origin behind a shared load balancer. Since our cluster uses bare-metal OVH servers with external load balancers, we cannot use CloudFront path prefix routing — we have a dedicated external IP per ALB.

**Question:** Is the CloudFront race window acceptable? If not, what is the remediation strategy?

Options:
- Accept the window (seconds to minutes of potential drops during a CloudFront rotation, which is rare).
- Pre-fetch and maintain a superset of all known CloudFront CIDRs (AWS publishes the full range in advance — we can fetch `ip-ranges.amazonaws.com` proactively on a schedule, not just on SNS).

---

### Q4 — Secret Header Pattern: Complementary or Alternative?

**Context:**

An alternative/complementary security pattern exists: **Cloudflare Transform Rules** or **CloudFront Custom Origin Headers** inject a shared secret into every request to the origin. The secret is validated at Envoy using a `SecurityPolicy` or `EnvoyFilter`. Any request missing the secret header is rejected at L7.

This pattern addresses **CNAME bypass attacks** — where an attacker discovers your origin domain (via CNAME lookup) and sends requests directly. IP whitelisting does not protect against this if the attacker controls an IP in the CDN's published range (via shared hosting on Cloudflare, for example).

**However, it does NOT replace L3/L4 whitelisting for DDoS protection:**

```
Without L3/L4 whitelisting:
  Attacker sends DDoS → reaches Envoy → Envoy does TLS handshake per packet
  → Envoy CPU spikes → pod OOMkilled or throttled
  → even if 99% of requests are rejected by secret header check,
    Envoy still touches every packet

With L3/L4 whitelisting:
  DDoS from non-CDN IP → Cilium eBPF drops in kernel (nanoseconds)
  → Envoy sees nothing
  → Envoy CPU flat
```

Both layers together provide defense-in-depth:
- L3/L4 whitelisting: DDoS protection, kernel-level drop
- Secret header: CNAME bypass prevention, L7 validation

**Question:** Should secret header validation be in scope for Phase 3, or as a follow-on phase? Phase 3 currently focuses on L3/L4 only per the CONTEXT.md scope boundary.

---

## 10. Design Corrections Made During This Review

The following values in CONTEXT.md are now known to be incorrect and should be updated before implementation:

| Field | CONTEXT.md Value | Corrected Value | Reason |
|---|---|---|---|
| Initial drain delay | 2 hours | **15 minutes** | Max CDN TTL is 900s. 2h assumed incorrect 60min corporate DNS proxy overrides which don't apply to CDN edge TTLs. |
| Extension interval | 30 minutes | **15 minutes** | Over-engineered. CDN connection recycling is faster. |
| MaxExtensions | 8 (→ 6h max) | **TBD pending Q1** | Depends on WebSocket-on-ALB confirmation. |
| Hubble lookback | 10 minutes | **5 minutes** | Hubble gRPC is sub-second direct query, not a Prometheus scrape pipeline. 10min was based on wrong latency assumption. |
| Enforcement layer | CCM / cloud SG | **Cilium eBPF** (`cilium_lb4_source_range` LPM trie) | Cluster has no CCM. Verified by `kubectl get pods -A` and node `providerID` check. |

---

## 11. End-to-End Flow Summary (One Page)

```
TENANT ENABLES WHITELIST TOGGLE (via Orchestrator UI)
        │
        ▼
orch_api_server detects toggle → reads cloudflare-ip-cache ConfigMap
        │
        ▼
iprecon patches EnvoyProxy: loadBalancerSourceRanges = Cloudflare CIDRs
        │
        ▼
Cilium reads EnvoyProxy spec → populates cilium_lb4_source_range eBPF map
        │
        ▼
[NORMAL OPERATION — all CDN traffic flows through]
User → Cloudflare Edge → ALB (Cilium allows Cloudflare CIDR) → Envoy → Pod

═══════════════════════════════════════════════════════

CDN IP ROTATION EVENT
        │
        ▼
Cloudflare publishes new IP list (CronJob detects diff)
OR AWS SNS fires AmazonIpSpaceChanged (Webhook receives)
        │
        ▼
iprecon computes:
  removed_cidrs = old_list - new_list
  union = new_list ∪ removed_cidrs
        │
        ▼
Patches EnvoyProxy with UNION (both old and new CIDRs active)
Adds removed_cidrs to pending_removal with origin tag + timestamp
        │
        ▼
[WAIT 15 MINUTES — DNS TTL propagation window]
During this window: old and new CDN IPs both work
        │
        ▼
Every 5 minutes: query Hubble for active FORWARDED flows
from removed CIDR
        │
        ├── Flows found? → Extend 15 minutes (up to MaxExtensions)
        │
        ├── LostEvents > 0? → Extend 15 minutes (ring buffer overflow)
        │
        ├── Hubble unavailable? → Fixed 30-minute conservative hold
        │
        └── No flows? → Remove CIDR from pending_removal
                          Patch EnvoyProxy (old CIDR removed)
                          Cilium eBPF map updated
                          Old CDN IP now rejected at kernel

═══════════════════════════════════════════════════════

STALENESS GUARD (runs on every reconciliation)
        │
        ├── Cache age > 36h → Alert
        ├── Cache age > 72h → Refuse CIDR removals (fail-closed)
        ├── New list empty → Reject write
        └── Overlap < 50% → Reject write (sanity gate)
```

---

## 12. What Is Not in Scope for Phase 3

| Item | Status |
|---|---|
| NLB whitelisting | Out of scope — no EnvoyProxy resource. NLBs receive raw TCP, no Envoy. |
| L7 secret header validation | Out of scope per CONTEXT.md (D-scope boundary). Deferred as future phase. |
| mTLS authenticated origin pulls | Deferred — noted in CONTEXT.md as future hardening. |
| Cloudflare Tunnel (`cloudflared`) | Deferred — requires ingress architecture change. |
| WebSocket timeout configuration (`ClientTrafficPolicy`) | TBD — depends on Q1 answer from mentor. |

---

*Prepared for mentor review — 2026-07-06*
*All technical findings verified against cluster state and codebase during design review session.*
