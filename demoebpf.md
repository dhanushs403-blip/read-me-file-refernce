

Below is a **demo playbook** you can present and execute live.

---

# 1. What to say in the demo

## A. Problem statement
- We want **TCP rate limiting in kernel space**, before traffic reaches Envoy.
- We support **two-tier policy**:
  - **Provider ceiling**: hard upper bound set by platform admin
  - **Tenant limit**: tenant-specific value, must stay within provider ceiling

## B. Data path
- Traffic hits node NIC `bond0`
- **XDP eBPF program** runs first
- It enforces:
  - **new TCP connections/sec** via SYN token bucket
  - **max concurrent connections** via connection counter
- If packet is over limit, it is **dropped in kernel**
- Allowed traffic continues to Cilium → Envoy → backend

## C. Why eBPF
- Faster than app-level filtering
- Protects Envoy before overload
- No restart needed to change limits; just update maps

---

# 2. Files used in this project

## eBPF code
- `src/xdp_vishanti_combined.c`
  - XDP hook
  - SYN rate limiting
  - connection count limiting
- `src/tc_vishanti_egress.c`
  - intended for close tracking on egress
  - in practice, Cilium ordering limits its usefulness

## Compiled objects
- `obj/xdp_vishanti_combined.o`
- `obj/tc_vishanti_egress.o`

## Scripts
- `/root/node-bootstrap.sh`
  - loads programs onto node
- `/root/configure-provider.sh`
  - set provider ceiling
- `/root/configure-tenant.sh`
  - set tenant limit
- `/root/tenant-onboard.sh`
  - convenience wrapper
- `/root/monitor.sh`
  - live monitor for selected tenant/LB
- `/root/send-attack.sh`
  - reset / attack / restore helper

## Cleanup
- `conn-cleanup.sh`
- `systemd: vishanti-cleanup.service`

---

# 3. Compiler / Tooling used

- **clang 14**
- **llvm**
- **bpftool**
- **libbpf-dev**
- **kernel 5.15**
- **Cilium 1.18**
- **BPF filesystem** mounted at `/sys/fs/bpf`

### Compile commands used
```bash
clang -O2 -g -target bpf -D__TARGET_ARCH_x86 \
  -I/usr/include/x86_64-linux-gnu \
  -c src/xdp_vishanti_combined.c -o obj/xdp_vishanti_combined.o

clang -O2 -g -target bpf -D__TARGET_ARCH_x86 \
  -I/usr/include/x86_64-linux-gnu \
  -c src/tc_vishanti_egress.c -o obj/tc_vishanti_egress.o
```

---

# 4. Maps used in the project

Pinned under:
```bash
/sys/fs/bpf/vishanti/
```

## Rate maps
- `syn_provider_cfg`
  - key = LB IP
  - value = `{rate_per_sec, burst}`
- `syn_tenant_cfg`
  - key = LB IP
  - value = `{rate_per_sec, burst}`
- `syn_rate_state`
  - token bucket runtime state
- `syn_drop_count`
  - per-CPU drop counters

## Connection maps
- `conn_provider_max`
  - provider max concurrent connections
- `conn_tenant_max`
  - tenant max concurrent connections
- `conn_count`
  - current active count seen by XDP logic
- `conn_drop_count`
  - connection-limit drops

## Control map
- `managed_lb_ips`
  - which LB IPs are under control

---

# 5. Important demo note about “full TCP”
For your demo audience, say this clearly:

> We test with **raw SYN packets** from `nping` because traffic generated from the same node using full socket connections can bypass the XDP path due to Cilium socket hooks.  
> But from the rate limiter’s perspective, **blocking the SYN means blocking the full TCP connection**, because no TCP connection can complete without the initial SYN.

So your four test scenarios are conceptually **full TCP connection tests**, but practically on this node we inject **raw SYN traffic** to validate the kernel enforcement.

---

# 6. Demo script you can read and execute

Save this as:

```bash
/root/demo-walkthrough.sh
```

```bash
cat > /root/demo-walkthrough.sh << 'SCRIPT'
#!/bin/bash

echo "============================================================"
echo " VISHANTI eBPF DEMO WALKTHROUGH"
echo "============================================================"
echo ""
echo "1. This project implements two-tier TCP rate limiting"
echo "   in kernel space using eBPF."
echo ""
echo "2. Provider sets a hard ceiling."
echo "3. Tenant sets a limit within that ceiling."
echo "4. XDP program on bond0 enforces:"
echo "   - SYN rate per second"
echo "   - Max concurrent connections"
echo ""
echo "5. Programs used:"
echo "   - xdp_vishanti_combined.c"
echo "   - tc_vishanti_egress.c"
echo ""
echo "6. Compiled with clang for BPF target."
echo ""
echo "7. Maps used:"
echo "   - syn_provider_cfg"
echo "   - syn_tenant_cfg"
echo "   - syn_rate_state"
echo "   - syn_drop_count"
echo "   - conn_provider_max"
echo "   - conn_tenant_max"
echo "   - conn_count"
echo "   - conn_drop_count"
echo "   - managed_lb_ips"
echo ""
echo "8. Scripts used:"
echo "   - /root/node-bootstrap.sh"
echo "   - /root/configure-provider.sh"
echo "   - /root/configure-tenant.sh"
echo "   - /root/monitor.sh"
echo "   - /root/send-attack.sh"
echo ""
echo "9. For testing on this node we use raw SYN packets."
echo "   Blocking SYN = blocking full TCP connection establishment."
echo ""
echo "============================================================"
SCRIPT

chmod +x /root/demo-walkthrough.sh
```

Run:
```bash
bash /root/demo-walkthrough.sh
```

---

# 7. Demo execution flow for NEW LB

Assume new LB IP is:
- `23.111.176.43`
- tenant namespace: `vishanti-demo2`
- hostname: `albdemo2.incubera.xyz`

---

## Step 1: Show program is attached

```bash
ip link show bond0 | grep xdp
bpftool prog show pinned /sys/fs/bpf/vishanti/xdp_prog
ls /sys/fs/bpf/vishanti/
```

Explain:
- XDP program is attached on `bond0`
- maps are pinned in BPF filesystem
- configuration changes update maps, not code

---

## Step 2: Configure provider ceiling for new LB

Run:
```bash
bash /root/configure-provider.sh
```

Set:
- SYN/sec = `10000`
- burst = `3000`
- max connections = `30000`

Explain:
- provider ceiling is the hard cap
- tenant cannot exceed it

---

## Step 3: Configure tenant limit for new LB

Run:
```bash
bash /root/configure-tenant.sh
```

Set:
- SYN/sec = `500`
- burst = `600`
- max connections = `1000`

Explain:
- this is the effective limit for testing
- it is validated against provider values

---

## Step 4: Start monitor in terminal 1

```bash
bash /root/monitor.sh
```

Select:
- `vishanti-demo2`

Explain fields:
- **Active Connections** = current tracked count
- **SYN Drops** = packets dropped by rate limiter
- **Conn Drops** = packets dropped due to max connection limit
- **Provider/Tenant tokens** = token bucket fullness

---

# 8. Manual test plan for your 4 cases

Because same-node full TCP connect can bypass XDP, use raw SYN testing to demonstrate the TCP connection establishment limiting.

For each case:
1. reset counters
2. send raw SYNs
3. observe monitor
4. read result

---

## Common reset command

Before every test:
```bash
bash /root/send-attack.sh
```

Choose:
- tenant = `vishanti-demo2`
- option `4` reset

---

## Test Case 1
### Target behavior
- configured:
  - SYN/sec = 500
  - burst = 600
  - max connections = 1000
- send equivalent of:
  - 300/sec
  - total 800

### Command
```bash
nping --tcp -p 443 --flags SYN -c 800 --rate 300 23.111.176.43
```

### Expected result
- **SYN drops = 0**
- **Conn drops = 0**
- because:
  - 300/sec < 500/sec
  - 800 total < 1000 connections

### What to say
> This is normal traffic within both rate and connection thresholds, so everything passes.

---

## Test Case 2
### Target behavior
- send:
  - 700/sec
  - total 800

### Command
```bash
nping --tcp -p 443 --flags SYN -c 800 --rate 700 23.111.176.43
```

### Expected result
- **SYN drops > 0**
- **Conn drops = 0**
- because:
  - rate exceeds 500/sec
  - total admitted traffic likely stays below 1000 connections

### What to say
> The rate limiter triggers first and drops excess SYNs before the connection counter even becomes the bottleneck.

---

## Test Case 3
### Target behavior
- send:
  - 700/sec
  - total 2000

### Command
```bash
nping --tcp -p 443 --flags SYN -c 2000 --rate 700 23.111.176.43
```

### Expected result
- heavy **SYN drops**
- connection drops may be low or zero
- because rate limiter removes most traffic before it reaches the connection counter

### What to say
> This shows that the first protection layer, SYN rate limiting, is effective enough to absorb the attack before the connection pool fills.

---

## Test Case 4
### Target behavior
- send:
  - 300/sec
  - total 2000

### Command
```bash
nping --tcp -p 443 --flags SYN -c 2000 --rate 300 23.111.176.43
```

### Expected result
- **SYN drops = 0**
- **Conn drops > 0**
- **Active connections ≈ 1000**
- because:
  - rate is within allowed threshold
  - but total connection attempts exceed max concurrent connection limit

### What to say
> Here the rate is acceptable, so packets pass the first gate. But once concurrent connections hit 1000, the second gate starts dropping new connections.

---

# 9. “Full TCP” explanation for the audience

Say this exactly:

> A full TCP connection always starts with a SYN packet.  
> Our eBPF program makes the allow/deny decision at the SYN stage.  
> So if the SYN is dropped, the full TCP connection never starts.  
> That is why testing raw SYNs is valid for demonstrating full TCP connection protection.

If they insist on full TCP:
- use external client machine/laptop/browser for real full connection attempts
- monitor counters on server
- but for deterministic rate generation, `nping --flags SYN` is the correct tool

---

# 10. Useful commands during demo

## Show tenant config
```bash
bpftool map lookup pinned /sys/fs/bpf/vishanti/syn_provider_cfg key hex 17 6f b0 2b
bpftool map lookup pinned /sys/fs/bpf/vishanti/syn_tenant_cfg key hex 17 6f b0 2b
bpftool map lookup pinned /sys/fs/bpf/vishanti/conn_provider_max key hex 17 6f b0 2b
bpftool map lookup pinned /sys/fs/bpf/vishanti/conn_tenant_max key hex 17 6f b0 2b
```

## Show counters
```bash
bpftool map lookup pinned /sys/fs/bpf/vishanti/conn_count key hex 17 6f b0 2b
bpftool map lookup pinned /sys/fs/bpf/vishanti/syn_drop_count key hex 17 6f b0 2b
bpftool map lookup pinned /sys/fs/bpf/vishanti/conn_drop_count key hex 17 6f b0 2b
```

## Restore limits after test
```bash
bash /root/configure-tenant.sh
```

Set back to normal values.

---

# 11. Recommended demo structure

## Terminal 1
```bash
bash /root/monitor.sh
```

## Terminal 2
```bash
bash /root/demo-walkthrough.sh
bash /root/configure-provider.sh
bash /root/configure-tenant.sh
bash /root/send-attack.sh   # reset
nping ...                   # run test case
```

---

# 12. Suggested enhancement

The easiest enhancement for your demo is a dedicated script:
- choose tenant
- apply test config
- reset counters
- run one chosen test
- print summary

If you want, I can generate **one final `run-test-case.sh` script** that supports all 4 scenarios automatically.


Below is a **presentation script / speaking script** you can use to explain `src/xdp_vishanti_combined.c` in a demo.

---

# Demo Explanation Script for `xdp_vishanti_combined.c`

## 1. What this file is

> This file contains the main eBPF program that runs at the **XDP hook** on the network interface `bond0`.  
> XDP is the earliest place in the Linux packet path where we can inspect and drop packets.  
> That means we can stop unwanted TCP traffic before it reaches Cilium, Envoy, or the application.

---

## 2. What this program does

> This single XDP program performs two protections for each managed LoadBalancer IP:
>
> 1. **SYN rate limiting**  
>    It controls how many new TCP connection attempts per second are allowed.
>
> 2. **Concurrent connection limiting**  
>    It controls how many TCP connections can exist at the same time.
>
> So this file is effectively the core of our kernel-level TCP protection.

---

## 3. Why this is called “combined”

> Originally, SYN rate limiting and connection counting were considered as separate hooks, but because of how Cilium attaches its own TC programs, we merged the important logic into one XDP program.
>
> So this file combines:
> - SYN token bucket logic
> - connection count logic
> - client-side FIN/RST decrement logic

---

## 4. High-level flow

You can say:

> For every packet arriving on `bond0`, this program does the following:
>
> - Parse Ethernet
> - Parse IPv4
> - Parse TCP
> - Check whether destination IP is one of our managed LB IPs
> - If not, the packet is ignored and passed through
> - If it is a managed LB IP:
>   - for SYN packets, enforce rate limit and max connection limit
>   - for FIN/RST packets from the client side, decrement the connection counter

---

## 5. Maps used by this program

> This program uses several BPF maps.  
> The maps are the configuration and state storage for the eBPF program.

### Configuration maps
- `syn_provider_cfg`
  - provider-level hard ceiling for SYN/sec and burst
- `syn_tenant_cfg`
  - tenant-level requested SYN/sec and burst
- `conn_provider_max`
  - provider hard ceiling for max concurrent connections
- `conn_tenant_max`
  - tenant requested max concurrent connections
- `managed_lb_ips`
  - list of LB IPs that this program should enforce

### Runtime state maps
- `syn_rate_state`
  - stores token bucket state for each LB IP
- `conn_count`
  - stores active connection count per LB IP

### Observability maps
- `syn_drop_count`
  - how many SYNs were dropped due to rate limiting
- `conn_drop_count`
  - how many SYNs were dropped due to max connection limit

---

## 6. Two-tier logic

> This program implements **two-tier policy enforcement**.
>
> The provider defines the maximum allowed value.  
> The tenant can define a value within that limit.  
> The program enforces the tenant value, but the tenant is never allowed to exceed the provider ceiling.
>
> In other words, effective enforcement is:
>
> ```text
> effective = min(provider, tenant)
> ```

---

## 7. SYN rate limiting logic

> For new TCP connections, the program looks only at packets with:
>
> - `SYN = 1`
> - `ACK = 0`
>
> These are the initial connection attempts.
>
> For each LB IP, the program maintains a **token bucket**:
>
> - tokens refill over time according to configured rate
> - burst defines the maximum bucket size
> - every new SYN consumes one token
> - if no tokens are available, the SYN is dropped immediately

### What this means in practice
> If tenant limit is 500 SYN/sec with burst 600:
> - normal traffic under 500/sec passes
> - brief spikes up to 600 are tolerated
> - sustained traffic above 500/sec starts getting dropped

---

## 8. Connection count limiting logic

> In addition to rate limiting, the program also counts concurrent TCP connections.
>
> When a SYN arrives and passes the rate check:
> - it atomically increments the connection counter
> - then checks whether the counter is above the allowed max
> - if above limit, it rolls back the increment and drops the packet

### Important implementation detail
> We use an **increment-first, rollback-if-over-limit** pattern.  
> This avoids race conditions between CPUs.
>
> So even under high parallel traffic, the limit is enforced safely.

---

## 9. FIN/RST handling

> When the client sends a TCP `FIN` or `RST`, the XDP program decrements the connection counter.
>
> That means client-side close is accounted for inside XDP.
>
> For server-side closes, we intended to use TC egress, but because of Cilium hook ordering, that path is limited, so we added a cleanup reconciliation mechanism in userspace.

---

## 10. Why XDP is important

> XDP is important because it runs before the normal Linux network stack.
>
> So if a packet is over limit:
> - it is dropped immediately
> - it never reaches Envoy
> - it never reaches the backend
> - CPU and memory usage are protected

### Short demo-friendly line
> “We are dropping attack traffic in the kernel, not in userspace.”

---

## 11. Why this protects full TCP connections

> Even though many of our tests use raw SYN packets, this is still valid for full TCP connections.
>
> A full TCP connection always begins with a SYN.  
> If we drop that SYN, the full TCP handshake never completes.
>
> So from the perspective of rate limiting:
> - blocking SYN = blocking the TCP connection establishment

---

## 12. What happens for a packet in this program

You can say this step-by-step:

> Example packet flow:
>
> 1. A SYN comes to LB IP `23.111.176.43`
> 2. Program checks whether `23.111.176.43` is a managed LB
> 3. It reads provider and tenant config from maps
> 4. It checks token bucket:
>    - if too many SYNs/sec → drop
> 5. If rate is OK, it checks connection count:
>    - if too many concurrent connections → drop
> 6. If both checks pass:
>    - connection is allowed
>    - packet continues to Cilium and Envoy

---

## 13. Example from your demo

You can give this concrete example:

> Suppose for a tenant we configure:
>
> - provider ceiling:
>   - 10000 SYN/sec
>   - 30000 max connections
>
> - tenant limit:
>   - 2000 SYN/sec
>   - 7500 max connections
>
> Then this XDP program enforces:
>
> - 2000 SYN/sec effective limit
> - 7500 concurrent connections effective limit
>
> If I send:
> - 300/sec with total 800 → no drops
> - 700/sec with total 800 → SYN drops occur
> - 300/sec with total 2000 → connection drops occur when 7500 is exceeded in a real case, or 1000 if configured lower for demo

---

## 14. Why maps are important in the demo

> The code itself stays the same.  
> We do not recompile the program when changing tenant values.
>
> Instead, we update the BPF maps:
> - provider map
> - tenant map
> - connection max map
>
> This means configuration changes are live, immediate, and do not require restarts.

---

## 15. One-line explanation of each code section

If someone asks “what are the major parts of this file?” say:

- **struct definitions** → describe config and runtime state
- **map definitions** → persistent kernel storage
- **refill_and_consume()** → token bucket refill and rate enforcement
- **get_effective_max()** → selects correct connection limit
- **main XDP handler** → parses packet and applies logic
- **drop counters** → record enforcement events for monitoring

---

## 16. Final summary line for this file

> `xdp_vishanti_combined.c` is the kernel-side enforcement engine of this project.  
> It sits on the NIC, identifies traffic for managed LoadBalancer IPs, applies two-tier TCP admission control, and drops excess SYNs or excess connections before they reach Envoy.

---

# Short 30-second version

If you need a very short explanation:

> This file is our main XDP eBPF program. It runs on the NIC at the earliest possible point in the kernel. It enforces two protections per LoadBalancer IP: maximum new TCP connections per second and maximum concurrent TCP connections. It reads provider and tenant limits from BPF maps, applies the effective minimum, and drops packets immediately if limits are exceeded.

---

If you want, I can next give you the same style **explanation script for `tc_vishanti_egress.c`**.
