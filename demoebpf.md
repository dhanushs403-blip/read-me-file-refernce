

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
