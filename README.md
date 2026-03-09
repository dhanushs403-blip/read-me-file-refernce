# TCP Rate Limiting — Manual Testing Guide

> **Important:** Testing TCP limits (unlike HTTP limits) requires opening raw TCP/TLS connections and **holding them open** to build up concurrent connections.
> We use a simple Python script to do this cleanly without flooding your terminal.

## 1. Environment Setup

First, set up your variables in Terminal 1 (keep this terminal open for the tests):

```bash
# ── Set variables ──
export LB_IP=23.111.176.42
export HOST=rl4.incubera.xyz
export ENVOY_NS=envoy-gateway-system
export TENANT_NS=vishanti-tcprl
export ENVOY_POD=$(kubectl get pods -n $ENVOY_NS \
  -l gateway.envoyproxy.io/owning-gateway-name=albpolicytest,\
gateway.envoyproxy.io/owning-gateway-namespace=$TENANT_NS \
  -o jsonpath='{.items[0].metadata.name}')
echo "Envoy pod: $ENVOY_POD"
```

## 2. Create the TCP Load Tool

Instead of spamming `openssl` in the background (which causes terminal issues), run this block to create a clean `tcp-load.py` loading tool:

```bash
cat << 'EOF' > tcp-load.py
#!/usr/bin/env python3
import socket, ssl, time, threading, sys, argparse

parser = argparse.ArgumentParser(description="TCP Load Tester")
parser.add_argument("-c", "--connections", type=int, default=100, help="Concurrent connections")
parser.add_argument("-duration", type=int, default=30, help="Hold duration (seconds)")
parser.add_argument("-host", default="rl4.incubera.xyz", help="SNI Hostname")
parser.add_argument("ip", help="Target IP:Port")
args = parser.parse_args()

ip, port = args.ip.split(":"); port = int(port)
print(f"Opening {args.connections} connections to {ip}:{port} (SNI: {args.host})...")

ctx = ssl.create_default_context()
ctx.check_hostname = False; ctx.verify_mode = ssl.CERT_NONE
sockets, connected, failed, lock = [], 0, 0, threading.Lock()

def worker():
    global connected, failed
    try:
        s = socket.create_connection((ip, port), timeout=5)
        tls = ctx.wrap_socket(s, server_hostname=args.host)
        with lock: sockets.append(tls); connected += 1
    except:
        with lock: failed += 1

threads = [threading.Thread(target=worker) for _ in range(args.connections)]
for t in threads: t.start()
for t in threads: t.join()

print(f"Successfully opened {connected} active connections. ({failed} failed)")
print(f"Holding connections open for {args.duration} seconds...")
try: time.sleep(args.duration)
except KeyboardInterrupt: pass
print("Closing connections.")
for s in sockets: s.close()
EOF
chmod +x tcp-load.py
```

## 3. Enable Envoy Stats Monitoring

Open a **separate terminal (Terminal 2)**, set the `ENVOY_POD` variable again, and run port-forwarding so we can observe Envoy's metrics:

```bash
# In Terminal 2 (keep this running)
export ENVOY_POD=$(kubectl get pods -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=albpolicytest,gateway.envoyproxy.io/owning-gateway-namespace=$TENANT_NS -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n envoy-gateway-system $ENVOY_POD 19000:19000
```

> **Tip:** Open a **third terminal (Terminal 3)** to check stats while the load is running!

---

## Test 1: Global Max Connections (100/pod)

**What:** Tests the absolute hard ceiling for the entire proxy pod. 
**Target Config:** `max_active_downstream_connections: 100` in `envoy-gateway-tenant.yaml` (Overload Manager).

### Baseline (Terminal 3)
```bash
curl -s localhost:19000/stats | grep "listener.0.0.0.0_10443.downstream_cx_active" | grep -v worker
```
Expected: `0` (or very low)

### During Load (Terminal 1)
```bash
# Open 300 connections at 60 conns/sec (to stay under the 100 conns/sec global rate limit)
# The test should finish in ~5 seconds and immediately hit the 200 combined connection limit!
./tcp-load.py -c 300 -duration 10 -rate 60 23.111.176.42:443
```
Expected output: Notice some connections fail because the 200 combined connection limit is hit.

### Verify Limits Hit (Terminal 3 - Run while the python script above is running)
```bash
# Check active connections -- it should plateau EXACTLY at 100 (or 200, depending on load balancer pod affinity)
curl -s localhost:19000/stats | grep "listener.0.0.0.0_10443.downstream_cx_active" | grep -v worker

# Try another connection
curl -sk https://rl4.incubera.xyz/ -o /dev/null -w "HTTP %{http_code}\n" --connect-timeout 3
```
Expected: The `downstream_cx_active` gauge will abruptly stop at `100` and refuse to go any higher. The `curl` command will hang or fail (Unexpected EOF) because the global capacity has been reached and Envoy is aggressively slamming the door on any new TCP requests.

---

## Test 2: Connection Rate (Global): 50 conn/sec per Envoy pod
## Effective rate ≈ 100 conn/sec with 2 Envoy replicas

**What:** Targets the `local_ratelimit` Network Filter.

```bash
# In Terminal 1 (Blast 200 connections instantly, hold for 5s)
./tcp-load.py -c 500 -duration 1 23.111.176.42:443
```

```bash
# In Terminal 3 (Run immediately after the blast)
# Check for rate_limited stats indicating connections were rejected
curl -s localhost:19000/stats | grep "local_rate_limit.tcp_conn_rate_global_https.rate_limited"
```
Expected: You will see connections failing in the python script output, and the `rate_limited` counter in Envoy stats will increase.

---

## Test 3: Per-Listener Connection Limit (50/chain per pod)
## Effective limit ≈ 100 connections with 2 Envoy replicas

**What:** Tests the smaller "room capacity" limit applied only to a specific listener/port (HTTPS on 443).
**Target Config:** `envoy.filters.network.connection_limit` set to `max_connections: 50` in `tcp-rate-limits.yaml`.

```bash
# In Terminal 1 (Open 150 connections to the HTTPS port)
# We throttle the connection initiation to 30 conns/sec to avoid hitting the 50 conns/sec rate limit!
./tcp-load.py -c 150 -duration 10 -rate 30 23.111.176.42:443
```

```bash
# In Terminal 3 (Run while load is active)
# Check active connections
curl -s localhost:19000/stats | grep "downstream_cx_active" | grep -v worker

# Check limited_connections counter
curl -s localhost:19000/stats | grep "connection_limit.https-10443.limited_connections"
```
Expected: The Envoy counter `limited_connections` will increase as it rejects connections over 500 on this specific listener.

---

## Test 4: Per-IP Connection Rate (via RLS)

> **⚠️ CRITICAL WARNING (SNAT):** If your LoadBalancer uses `externalTrafficPolicy: Cluster`, the Kubernetes kube-proxy will perform Source NAT (SNAT). This means Envoy will rate-limit the **internal IP of your Kubernetes worker nodes**, not the actual end-user's public IP! Keep this in mind for production.

**What:** Targets the `ratelimit` network filter checking against Redis RLS.

```bash
# In Terminal 1 (Blast 50 connections from a single IP limit)
./tcp-load.py -c 50 -duration 5 23.111.176.42:443
```

```bash
# In Terminal 3 (Run immediately after)
# Check global rate limit stats
curl -s localhost:19000/stats | grep "ratelimit" | grep -v local
```
Expected: If your RLS ConfigMap is loaded with the IP limits, the `over_limit` counter will increase.

---

## Test 5: TCP Keepalive / Half-Open Detection

**What:** Verifies `ClientTrafficPolicy` `tcpKeepalive` probes close truly dead connections in ~90s.

> **Note:** A connection that is simply idle will not be closed if the client OS is healthy (it will reply to keepalive probes with ACKs). To test keepalive, we must simulate a "dead" client by forcibly dropping traffic.

```bash
# In Terminal 1 (Open a single connection)
./tcp-load.py -c 1 -duration 120 23.111.176.42:443 &

# Immediately drop all incoming packets from the LB to simulate a severed connection
sudo iptables -A INPUT -s 23.111.176.42 -j DROP
```

```bash
# In Terminal 3 (Monitor it occasionally)
curl -s localhost:19000/stats | grep "listener.0.0.0.0_10443.downstream_cx_active" | grep -v worker
```
Expected: The connection will show as `active_cx: 1` initially. After exactly 90s, Envoy's keepalive probes will realize the client is dead (because iptables dropped the ACKs) and forcefully close it, returning Envoy stats to `0`.

```bash
# IMPORTANT: Remove the firewall rule when done! (Terminal 1)
sudo iptables -D INPUT -s 23.111.176.42 -j DROP
```

---

## Test 6: SYN Flood Protection (Linux Kernel)

**What:** Tests kernel `sysctl` (`tcp_syncookies`) protection against SYN floods. Note: This test operates beneath Envoy.

```bash
# In Terminal 1 (Verify settings)
sysctl net.ipv4.tcp_syncookies
sysctl net.ipv4.tcp_max_syn_backlog
dmesg | grep -i "SYN flood" | tail -3
```

```bash
# In Terminal 1 (Launch a raw packet SYN flood)
sudo hping3 -S -p 443 --flood --rand-source 23.111.176.42 &
FLOOD_PID=$!
sleep 5

# In Terminal 3 (Try a legitimate connection)
curl -sk https://rl4.incubera.xyz/ -o /dev/null -w "HTTP %{http_code}\n" --connect-timeout 5

# Clean up (Terminal 1)
kill $FLOOD_PID 2>/dev/null
dmesg | grep -i "SYN flood" | tail -3
```
Expected: `curl` returns `HTTP 200` perfectly during the attack because `tcp_syncookies` handles the flood. The `dmesg` output will show `Possible SYN flooding on port 443. Sending cookies.`
