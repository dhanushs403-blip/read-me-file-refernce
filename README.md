# TCP Rate Limiting — Manual Testing Guide

> **Important:** These tests operate at the **TCP connection layer**, not HTTP.
> We use `openssl s_client` to open and **hold** TCP/TLS connections, and monitor
> Envoy's `downstream_cx_active` stat to see the connection count in real-time.

## Environment Setup

```bash
# ── Set variables (run once) ──
export LB_IP=23.111.176.42
export HOST=rl4.incubera.xyz
export ENVOY_NS=envoy-gateway-system
export TENANT_NS=vishanti-ratelimit4
export ENVOY_POD=$(kubectl get pods -n $ENVOY_NS \
  -l gateway.envoyproxy.io/owning-gateway-name=albpolicytest,\
gateway.envoyproxy.io/owning-gateway-namespace=$TENANT_NS \
  -o jsonpath='{.items[0].metadata.name}')
echo "Envoy pod: $ENVOY_POD"

# ── Port-forward Envoy admin (keep running in a separate terminal) ──
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000
```

> **Tip:** Keep 3 terminals open:
> - **Terminal 1** — Run load commands
> - **Terminal 2** — `watch` Envoy stats
> - **Terminal 3** — Try new connections (`curl`)

---

## Test 1: Global Max Connections (1000/pod)

**What:** Overload Manager limits total active TCP connections per Envoy pod.
**Config:** `envoy-gateway-tenant.yaml` → `max_active_downstream_connections: 1000`
**Effective limit:** ~2000 (1000 × 2 pods)

### Baseline

```bash
# Terminal 2: Watch active connection count on this pod
watch -n1 "curl -s localhost:19000/stats | grep listener.0.0.0.0_10443.downstream_cx_active"

# Terminal 3: Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
```

**Expected:** `downstream_cx_active: 0` (or very low), HTTP 200.

### During Load

```bash
# Terminal 1: Open 3000 TCP/TLS connections and hold them open
for j in $(seq 1 3000); do
  openssl s_client -connect $LB_IP:443 -servername $HOST -quiet &
  sleep 0.01
done

# Terminal 2: Watch the active connection count climb
# It should plateau around 1000 on this pod (other 1000 go to pod 2)
watch -n1 "curl -s localhost:19000/stats | grep listener.0.0.0.0_10443.downstream_cx_active"

# Terminal 3: Try a NEW connection once limit is reached
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n" --connect-timeout 5
```

**Expected:**
- `downstream_cx_active` climbs to ~1000, then new connections are refused
- `curl` in Terminal 3 gets **connection refused/timeout** or **HTTP 503**
- Overload stats show `stop_accepting_connections` triggered

### After Load

```bash
# Terminal 1: Kill all background openssl connections
pkill -f "openssl s_client"

# Terminal 2: Watch connection count drop back to 0
# Terminal 3: Verify recovery
sleep 5
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"

# Check overload manager stats
curl -s localhost:19000/stats | grep "overload.stop_accepting"
```

**Expected:** Connections drop to 0, HTTP 200 restored.

---

## Test 2: Connection Rate Limiting (50 conn/sec per pod)

**What:** `local_ratelimit` listener filter uses a token bucket to throttle new TCP connections.
**Config:** `tcp-rate-limits.yaml` → token bucket (50 tokens, 50/s refill)
**Effective limit:** ~100 new conn/s (50 × 2 pods)

### Baseline

```bash
# Terminal 2: Watch the rate limit stats
watch -n1 "curl -s localhost:19000/stats | grep local_ratelimit"

# Terminal 3: Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
```

**Expected:** HTTP 200, `local_ratelimit` counters at baseline.

### During Load

```bash
# Terminal 1: Open 200 connections as fast as possible (no sleep = burst)
for j in $(seq 1 200); do
  openssl s_client -connect $LB_IP:443 -servername $HOST -quiet &
done

# Terminal 2: Watch rate_limited counter increase
watch -n1 "curl -s localhost:19000/stats | grep local_ratelimit"

# Terminal 3: Try a connection during the burst
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n" --connect-timeout 3
```

**Expected:**
- Many connections get **refused** (rate exceeded 50/s per pod)
- `local_ratelimit.rate_limited` counter increases
- `curl` in Terminal 3 may return **429** or connection refused

### After Load

```bash
# Terminal 1: Kill all background openssl
pkill -f "openssl s_client"

# Wait for token bucket to refill (1 second)
sleep 2
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"

# Check how many were rate-limited
curl -s localhost:19000/stats | grep local_ratelimit
```

**Expected:** HTTP 200. Token bucket refills instantly. `rate_limited` counter shows total rejections.

---

## Test 3: Per-Listener Connection Limit (500/chain/pod)

**What:** `connection_limit` network filter caps active connections per filter chain.
**Config:** `tcp-rate-limits.yaml` → `max_connections: 500`
**Effective limit:** ~1000 (500 × 2 pods)

### Baseline

```bash
# Terminal 2: Watch connection limit stats
watch -n1 "curl -s localhost:19000/stats | grep connection_limit"

# Terminal 3: Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
```

**Expected:** HTTP 200, `connection_limit.limited_connections: 0`.

### During Load

```bash
# Terminal 1: Open 600 connections and hold them (exceeds 500/pod)
for j in $(seq 1 600); do
  openssl s_client -connect $LB_IP:443 -servername $HOST -quiet &
  sleep 0.01
done

# Terminal 2: Watch limited_connections counter
watch -n1 "curl -s localhost:19000/stats | grep connection_limit"

# Terminal 3: Try a new connection once at limit
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n" --connect-timeout 5
```

**Expected:**
- `active_connections` climbs to 500, then stops
- `limited_connections` counter increases (excess connections rejected)
- `curl` may fail with connection reset/refused

### After Load

```bash
# Terminal 1: Clean up
pkill -f "openssl s_client"
sleep 3

# Terminal 3: Verify recovery
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
curl -s localhost:19000/stats | grep connection_limit
```

**Expected:** HTTP 200. `limited_connections` shows total rejections during the test.

---

## Test 4: Per-IP Connection Rate (via RLS)

**What:** `ratelimit` network filter checks per-IP rates against external Rate Limit Service (Redis).
**Config:** `tcp-rate-limits.yaml` → domain `tcp_ratelimit4`, descriptor `remote_address`
**Depends on:** RLS ConfigMap having `tcp_ratelimit4` domain

### Baseline

```bash
# Terminal 2: Watch RLS-related stats
watch -n1 "curl -s localhost:19000/stats | grep tcp_conn_rate_per_ip"

# Verify RLS config has the TCP domain
kubectl get configmap -n $ENVOY_NS -l app=ratelimit -o yaml | grep -A5 "tcp_ratelimit4"

# Terminal 3: Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
```

**Expected:** HTTP 200. If `tcp_ratelimit4` domain is not in RLS config, this filter passes all connections.

### During Load

```bash
# Terminal 1: Open 50 rapid connections from this single IP
for j in $(seq 1 50); do
  openssl s_client -connect $LB_IP:443 -servername $HOST -quiet &
done

# Terminal 2: Watch for over_limit or cx_closed stats
watch -n1 "curl -s localhost:19000/stats | grep tcp_conn_rate_per_ip"

# Terminal 3: Try another connection
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n" --connect-timeout 3
```

**Expected (if RLS configured):**
- Some connections get **closed/refused** (per-IP rate exceeded)
- `over_limit` counter increases
- `curl` returns 429 or connection refused

**Expected (if RLS NOT configured):**
- All connections succeed — filter is installed but no matching domain in RLS.

### After Load

```bash
pkill -f "openssl s_client"
sleep 2
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
curl -s localhost:19000/stats | grep tcp_conn_rate_per_ip
```

---

## Test 5: TCP Keepalive / Half-Open Detection

**What:** `ClientTrafficPolicy` enables TCP keepalive probes on connections.
**Config:** `tcp-connection-limits.yaml` → probes=3, idleTime=60s, interval=10s
**Detection:** Dead connections closed within ~90s of going idle.

### Baseline (verify config)

```bash
# Verify the CTP has keepalive configured
kubectl get clienttrafficpolicy ratelimit4-tcp-limits -n $TENANT_NS \
  -o jsonpath='{.spec.tcpKeepalive}' && echo

# Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
```

**Expected:** `{"idleTime":"60s","interval":"10s","probes":3}`

### During Load (idle connection test)

```bash
# Terminal 1: Open a single connection and leave it completely idle
openssl s_client -connect $LB_IP:443 -servername $HOST -quiet &
OPENSSL_PID=$!
echo "Connection PID: $OPENSSL_PID"

# Terminal 2: Watch the connection in Envoy stats
watch -n1 "curl -s localhost:19000/stats | grep listener.0.0.0.0_10443.downstream_cx_active"

# Wait 15 seconds (less than idleTime=60s)
sleep 15

# Check if connection is still alive
kill -0 $OPENSSL_PID 2>/dev/null && echo "ALIVE after 15s (expected)" || echo "DEAD (unexpected)"

# For full keepalive test, wait ~100 seconds total:
# sleep 100
# kill -0 $OPENSSL_PID 2>/dev/null && echo "STILL ALIVE (unexpected)" || echo "CLOSED by keepalive (expected)"
```

**Expected:**
- Connection stays **alive** after 15s (keepalive probes start at 60s)
- After ~90s of complete idleness, the connection would be closed by Envoy

### After Load

```bash
kill $OPENSSL_PID 2>/dev/null
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
```

**Expected:** HTTP 200.

---

## Test 6: SYN Flood Protection (Kernel)

**What:** Linux kernel's TCP SYN cookies prevent SYN flood attacks from exhausting the connection table.
**Config:** `/etc/sysctl.conf` → `tcp_syncookies=1`, `tcp_max_syn_backlog=1024`
**Level:** Kernel — protects TCP handshake before Envoy even sees the connection.

### Baseline

```bash
# Verify sysctl settings
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.tcp_max_syn_backlog

# Check kernel log for existing SYN flood messages
dmesg | grep -i "SYN flood" | tail -3

# Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
```

**Expected:** `tcp_syncookies = 1`, `tcp_max_syn_backlog = 1024`, HTTP 200.

### During Load (SYN flood)

```bash
# Terminal 1: Launch a SYN flood (spoofed source IPs, max speed)
sudo hping3 -S -p 443 --flood --rand-source $LB_IP &
FLOOD_PID=$!

# Wait 5 seconds for the flood to build up
sleep 5

# Terminal 3: Try LEGITIMATE connections during the flood
for i in 1 2 3 4 5; do
  curl -sk https://$HOST/ -o /dev/null -w "Attempt $i: HTTP %{http_code} in %{time_total}s\n" --connect-timeout 5
  sleep 1
done

# Check if kernel detected the flood
dmesg | grep -i "SYN flood" | tail -5
```

**Expected:**
- Legitimate connections return **HTTP 200** despite the flood
- Kernel may log: `TCP: Possible SYN flooding on port 443. Sending cookies.`
- SYN cookies let real clients complete the handshake

### After Load

```bash
# Stop the flood
kill $FLOOD_PID 2>/dev/null

# Verify full recovery
sleep 2
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"

# Final kernel log check
dmesg | grep -i "SYN flood" | tail -5
```

**Expected:** HTTP 200, normal latency. Server fully recovered.

---

## Cleanup (after all tests)

```bash
# Kill any remaining background openssl connections
pkill -f "openssl s_client" 2>/dev/null

# Kill any port-forwards
pkill -f "kubectl port-forward" 2>/dev/null

# Verify everything is clean
curl -sk https://$HOST/ -o /dev/null -w "Final check: HTTP %{http_code}\n"
```

## Quick Reference

| # | Feature | Load Tool | Monitor Stat | Success Indicator |
|---|---------|-----------|-------------|-------------------|
| 1 | Global Max Conn | `openssl s_client` ×3000 | `downstream_cx_active` | Plateaus at ~1000/pod |
| 2 | Conn Rate | `openssl` burst (no sleep) | `local_ratelimit.rate_limited` | Counter increases |
| 3 | Listener Limit | `openssl s_client` ×600 | `connection_limit.limited_connections` | Counter increases |
| 4 | Per-IP Rate | `openssl` ×50 rapid | `tcp_conn_rate_per_ip.over_limit` | Connections refused |
| 5 | TCP Keepalive | Single idle `openssl` | `downstream_cx_active` | Drops after ~90s |
| 6 | SYN Flood | `hping3 --flood` | `dmesg` kernel log | `curl` still returns 200 |
