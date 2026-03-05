# TCP Rate Limiting — Manual Testing Guide

**Environment:**

| Item | Value |
|------|-------|
| Namespace | `vishanti-ratelimit4` |
| Gateway | `albpolicytest` |
| Domain | `rl4.incubera.xyz` |
| LB IP | `23.111.176.42` |
| Envoy Replicas | 2 |

**Prerequisites:**
```bash
# Tools needed
apt-get install -y hping3     # For SYN flood test
go install github.com/rakyll/hey@latest  # Or use pre-installed hey

# Set variables used throughout
export LB_IP=23.111.176.42
export HOST=rl4.incubera.xyz
export ENVOY_NS=envoy-gateway-system
export TENANT_NS=vishanti-ratelimit4
export ENVOY_POD=$(kubectl get pods -n $ENVOY_NS \
  -l gateway.envoyproxy.io/owning-gateway-name=albpolicytest,\
gateway.envoyproxy.io/owning-gateway-namespace=$TENANT_NS \
  -o jsonpath='{.items[0].metadata.name}')
echo "Using Envoy pod: $ENVOY_POD"
```

---

## Test 1: Global Max Connections

**What:** Overload Manager limits total active TCP connections per Envoy pod to **1000**.  
**Config:** `envoy-gateway-tenant.yaml` → `global_downstream_max_connections: 1000`  
**Effective limit:** ~2000 (1000 × 2 pods)

### Baseline (before load)

```bash
# 1. Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"

# 2. Check current active connections on each pod
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "downstream_cx_active" | grep -v worker | head -5
kill $PF_PID 2>/dev/null
```

**Expected:** HTTP 200, very few active connections.

### During Load

```bash
# 3. Open 1200 concurrent connections and hold them (limit is 1000/pod, 2000 effective)
hey -c 1200 -z 30s -q 1 -host "$HOST" https://$LB_IP/

# 4. While hey is running, in another terminal check active connections:
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "downstream_cx_active" | grep -v worker
curl -s http://localhost:19000/stats | grep "overload.stop_accepting"
kill $PF_PID 2>/dev/null

# 5. Try a new connection while at capacity:
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n" --connect-timeout 3
```

**Expected:**
- `hey` output shows some failed connections if >2000 attempted
- `downstream_cx_active` near 1000 per pod
- New `curl` may get refused or timeout once limit is hit

### After Load

```bash
# 6. Wait for hey to finish, then re-check
sleep 5
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"

# 7. Check overload stats
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "overload"
kill $PF_PID 2>/dev/null
```

**Expected:** HTTP 200 restored, overload counters may show non-zero values.

---

## Test 2: Connection Rate Limiting (Global)

**What:** `local_ratelimit` listener filter limits new TCP connections to **50/sec per pod**.  
**Config:** `tcp-rate-limits.yaml` → token bucket (50 tokens, refill 50/s)  
**Effective limit:** ~100 conn/s (50 × 2 pods)

### Baseline (before load)

```bash
# 1. Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"

# 2. Check current rate limit stats
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "local_ratelimit"
kill $PF_PID 2>/dev/null
```

**Expected:** HTTP 200, rate limit counters at baseline values.

### During Load

```bash
# 3. Burst 500 requests with 200 concurrent connections (far exceeds 50/s rate)
hey -c 200 -n 500 -host "$HOST" https://$LB_IP/

# 4. Look at hey output for:
#    - Status code distribution (expect mix of 200 and 429)
#    - Requests/sec achieved
```

**Expected:**
- Many `429 Too Many Requests` responses
- Actual throughput throttled to ~100 req/s (50/pod × 2)
- Some `200 OK` responses (the ones that got through)

### After Load

```bash
# 5. Wait 2 seconds (token bucket refills), then test again
sleep 2
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"

# 6. Check rate limit counters increased
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "local_ratelimit"
kill $PF_PID 2>/dev/null
```

**Expected:** HTTP 200 restored. Rate limit `rate_limited` counter shows total rejections.

---

## Test 3: Per-Listener Connection Limit

**What:** `connection_limit` network filter limits active connections to **500 per filter chain per pod**.  
**Config:** `tcp-rate-limits.yaml` → `max_connections: 500`  
**Effective limit:** ~1000 (500 × 2 pods)

### Baseline (before load)

```bash
# 1. Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"

# 2. Check connection limit stats
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "connection_limit"
kill $PF_PID 2>/dev/null
```

**Expected:** HTTP 200, `connection_limit.limited_connections: 0` or similar.

### During Load

```bash
# 3. Open 600 concurrent long-lived connections (exceeds 500/pod on at least one)
hey -c 600 -z 30s -q 1 -host "$HOST" https://$LB_IP/

# 4. While running, check another terminal:
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "connection_limit"
kill $PF_PID 2>/dev/null
```

**Expected:**
- `connection_limit.limited_connections` counter > 0
- Some connections in `hey` output get connection errors

### After Load

```bash
# 5. After hey finishes
sleep 3
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"

# 6. Final stats check
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "connection_limit"
kill $PF_PID 2>/dev/null
```

**Expected:** HTTP 200 restored. `limited_connections` counter shows how many were rejected.

---

## Test 4: Per-IP Connection Rate (via RLS)

**What:** `ratelimit` network filter sends `remote_address` descriptor to external Rate Limit Service.  
**Config:** `tcp-rate-limits.yaml` → domain `tcp_ratelimit4`  
**Depends on:** RLS ConfigMap having `tcp_ratelimit4` domain configured

### Baseline (before load)

```bash
# 1. Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"

# 2. Check if RLS is reachable and has the domain configured
kubectl get configmap -n $ENVOY_NS -l app=ratelimit -o yaml | grep -A5 "tcp_ratelimit4" || \
  echo "WARNING: tcp_ratelimit4 domain not found in RLS config"

# 3. Check ratelimit stats
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "ratelimit" | grep -v "local_ratelimit"
kill $PF_PID 2>/dev/null
```

**Expected:** HTTP 200. If RLS domain is not configured, all connections pass through.

### During Load

```bash
# 4. Send 100 rapid requests from this IP
hey -c 50 -n 100 -host "$HOST" https://$LB_IP/

# 5. Check status code distribution in hey output
```

**Expected (if RLS configured):**
- Mix of `200` and `429` responses
- `429` count increases as per-IP rate is exceeded

**Expected (if RLS NOT configured):**
- All `200` — the filter is present but RLS has no matching domain/descriptors

### After Load

```bash
# 6. Verify recovery
sleep 2
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"

# 7. Check ratelimit stats for this filter
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &
PF_PID=$!
sleep 1
curl -s http://localhost:19000/stats | grep "tcp_conn_rate_per_ip"
kill $PF_PID 2>/dev/null
```

**Expected:** HTTP 200. Stats show `cx_closed` or `over_limit` if RLS rejected connections.

> **Note:** To enable per-IP TCP rate limiting, add this to your RLS ConfigMap:
> ```yaml
> domain: tcp_ratelimit4
> descriptors:
>   - key: remote_address
>     rate_limit:
>       unit: second
>       requests_per_unit: 10
> ```

---

## Test 5: TCP Keepalive / Half-Open Detection

**What:** `ClientTrafficPolicy` configures TCP keepalive to detect and close dead/half-open connections.  
**Config:** `tcp-connection-limits.yaml` → `probes: 3, idleTime: 60s, interval: 10s`  
**Detection time:** ~90s for a dead connection (60s idle + 3×10s probes)

### Baseline (verify configuration)

```bash
# 1. Verify CTP is applied with keepalive settings
kubectl get clienttrafficpolicy ratelimit4-tcp-limits -n $TENANT_NS \
  -o jsonpath='{.spec.tcpKeepalive}' && echo

# 2. Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"
```

**Expected:**
```json
{"idleTime":"60s","interval":"10s","probes":3}
```

### During Load (idle connection test)

```bash
# 3. Open a connection using openssl and leave it idle
openssl s_client -connect $LB_IP:443 -servername $HOST -quiet &
OPENSSL_PID=$!

# 4. Wait 15 seconds (less than idleTime=60s)
sleep 15

# 5. Check if the connection is still alive
kill -0 $OPENSSL_PID 2>/dev/null && echo "Connection ALIVE (expected)" || echo "Connection DEAD"

# 6. Clean up
kill $OPENSSL_PID 2>/dev/null
```

**Expected:** Connection stays alive (keepalive probes only start after 60s idle).

### After Load

```bash
# 7. Verify normal connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code}\n"
```

**Expected:** HTTP 200. Keepalive is passive — it only activates on truly idle connections after 60s.

> **Full keepalive test (optional, takes ~2 minutes):**
> ```bash
> # Open connection, then block the network (simulating a dead client)
> openssl s_client -connect $LB_IP:443 -servername $HOST -quiet &
> OPENSSL_PID=$!
> # After 90s, Envoy should close the connection
> sleep 100
> kill -0 $OPENSSL_PID 2>/dev/null && echo "STILL ALIVE (unexpected)" || echo "CLOSED (expected)"
> ```

---

## Test 6: SYN Flood Protection (Kernel)

**What:** Linux kernel's `tcp_syncookies` and `tcp_max_syn_backlog` protect against SYN flood attacks.  
**Config:** `/etc/sysctl.conf` → `tcp_syncookies=1`, `tcp_max_syn_backlog=1024`  
**Level:** Kernel (below Envoy — protects the TCP handshake itself)

### Baseline (verify sysctl settings)

```bash
# 1. Check current kernel settings
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.tcp_max_syn_backlog

# 2. Check for any existing SYN flood messages in kernel log
dmesg | grep -i "SYN flood" | tail -3

# 3. Verify connectivity
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"
```

**Expected:**
```
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
```

### During Load (SYN flood)

```bash
# 4. Start a SYN flood in the background (needs root)
#    --flood = max speed, --rand-source = spoofed source IPs, -S = SYN packets
sudo hping3 -S -p 443 --flood --rand-source $LB_IP &
FLOOD_PID=$!

# 5. Wait 5 seconds for the flood to build up
sleep 5

# 6. Try a LEGITIMATE connection while the flood is active
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n" --connect-timeout 5

# 7. Try multiple times to confirm consistency
for i in 1 2 3 4 5; do
  curl -sk https://$HOST/ -o /dev/null -w "Attempt $i: HTTP %{http_code} in %{time_total}s\n" --connect-timeout 3
  sleep 0.5
done

# 8. Check kernel log for SYN flood detection
dmesg | grep -i "SYN flood" | tail -5
```

**Expected:**
- Legitimate `curl` requests return **HTTP 200** despite the flood  
- Kernel may log: `TCP: Possible SYN flooding on port 443. Sending cookies.`

### After Load (stop flood and verify)

```bash
# 9. Stop the flood
kill $FLOOD_PID 2>/dev/null
wait $FLOOD_PID 2>/dev/null

# 10. Verify the server is fully responsive
sleep 2
curl -sk https://$HOST/ -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n"

# 11. Final kernel log check
dmesg | grep -i "SYN flood" | tail -5
```

**Expected:** HTTP 200 with normal latency. Server fully recovered.

---

## Quick Reference: All Tests at a Glance

| # | Feature | Mechanism | Limit | Test Tool | Key Indicator |
|---|---------|-----------|-------|-----------|---------------|
| 1 | Global Max Conn | Overload Manager | 1000/pod | `hey -c 1200` | Connections refused |
| 2 | Conn Rate (Global) | `local_ratelimit` filter | 50/s/pod | `hey -c 200 -n 500` | HTTP 429 responses |
| 3 | Per-Listener Limit | `connection_limit` filter | 500/chain/pod | `hey -c 600` | Connection errors |
| 4 | Per-IP Rate (RLS) | `ratelimit` + Redis | configurable | `hey -c 50 -n 100` | HTTP 429 responses |
| 5 | TCP Keepalive | ClientTrafficPolicy | 60s idle | `openssl s_client` | Conn dies after ~90s |
| 6 | SYN Flood | Kernel sysctl | 1024 backlog | `hping3 --flood` | Legit conn still works |

## Useful Envoy Stats Commands

```bash
# Port-forward to Envoy admin (run once, use in subsequent commands)
kubectl port-forward -n $ENVOY_NS $ENVOY_POD 19000:19000 &

# All connection-related stats
curl -s localhost:19000/stats | grep -E "downstream_cx_(active|total)" | grep -v worker

# Rate limit counters
curl -s localhost:19000/stats | grep "local_ratelimit"

# Connection limit filter stats
curl -s localhost:19000/stats | grep "connection_limit"

# Per-IP RLS stats
curl -s localhost:19000/stats | grep "tcp_conn_rate_per_ip"

# Overload manager stats
curl -s localhost:19000/stats | grep "overload"

# Kill port-forward when done
kill %1 2>/dev/null
```
