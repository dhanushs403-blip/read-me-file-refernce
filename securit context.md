# Vishanti Cloud — Security Hardening Postflight Gate
## Design & Implementation Review

**Author:** Vishanti Engineering  
**Date:** 2026-05-26  
**Status:** 🟡 Awaiting Mentor Review  
**Phase:** 01 — Security Hardening Postflight Gate  

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Security Assessment Findings](#2-security-assessment-findings)
3. [Cluster Topology](#3-cluster-topology)
4. [Design Decisions](#4-design-decisions)
5. [System Architecture](#5-system-architecture)
6. [Implementation Plan — 4 Phases](#6-implementation-plan)
   - [Phase 01-01: Ansible Hardening Playbook](#phase-01-01-ansible-hardening-playbook-azsible)
   - [Phase 01-02: azsible Python Handler](#phase-01-02-azsible-python-hardening-handler)
   - [Phase 01-03: orch_api_server API](#phase-01-03-orch_api_server-security-status-api)
   - [Phase 01-04: orch-client UI](#phase-01-04-orch-client-pipeline-stage-ui)
7. [Port / Firewall Rules Reference](#7-port--firewall-rules-reference)
8. [Network Policy Reference](#8-network-policy-reference)
9. [Out of Scope (Deferred)](#9-out-of-scope-deferred)
10. [Open Questions for Mentor](#10-open-questions-for-mentor)

---

## 1. Problem Statement

The Vishanti Cloud platform currently deploys customer Kubernetes clusters via **Azsible** (an Ansible execution engine triggered by Redis pub/sub events). After a successful cluster provisioning, **no automated security hardening is applied** to the cluster nodes.

A manual script (`apply_security_hardening.sh`) exists and has been applied to the VSO management node, but:

- It is **not idempotent** (raw `iptables -I` commands run every time)
- It is **not applied automatically** as part of the deployment pipeline
- It is **not applied to all node types** — VSN control plane and worker nodes are not covered
- It is **not visible** to operators — there is no UI status for hardening state
- It is **a bash script**, not a proper Ansible playbook — making it unmaintainable and untestable

**Goal:** Convert the existing hardening logic into an automated, idempotent postflight step that runs on every cluster deploy across all 3 node types, with status visibility in orch-client.

---

## 2. Security Assessment Findings

The following publicly exposed services and missing controls were identified on the VSO node (`51.83.11.105`) and VSN nodes:

| Service | Port | Exposure Issue |
|---------|------|----------------|
| Kubernetes API | `6443` | Externally reachable — should be firewall-restricted |
| Kubelet API | `10250` | Publicly accessible — cluster-internal only |
| etcd client | `2379` | Publicly accessible — cluster-internal only |
| etcd peer | `2380` | Publicly accessible — cluster-internal only |
| node-exporter | `9100` | Publicly accessible — monitoring-internal only |
| Thanos Query | `31090` (NodePort) | Publicly accessible — should be ClusterIP only |
| Traefik dashboard | `31048` (NodePort) | Publicly accessible — must be blocked |
| MongoDB | `27017` | Accessible on public interface — must be blocked |
| Frontend app | `443/31437` (NodePort) | NodePort exposure — should move to Ingress-only |

**Actions required:**
- Block sensitive ports via host-level `iptables` rules (allow only from cluster subnet + localhost)
- Apply Kubernetes `NetworkPolicy` resources for service-to-service traffic isolation
- Convert `NodePort` services to `ClusterIP` where possible
- Disable Traefik insecure dashboard (`api.insecure=false`)

---

## 3. Cluster Topology

Every customer deployment consists of a minimum of **3 nodes**:

```
┌─────────────────────────────────────────────────────────────────┐
│   VSO (Vishanti Service Orchestrator)  — 51.83.11.105           │
│   Role: Vishanti management plane                                │
│   K8s: Single-node control-plane cluster                         │
│   CRI: containerd 1.7.23 / k8s v1.30.6                          │
│   Runs: Traefik, Keycloak, Thanos, Grafana, MongoDB, Redis       │
└──────────────────────────┬──────────────────────────────────────┘
                           │ manages / deploys to
          ┌────────────────┴──────────────────────┐
          │                                       │
┌─────────▼──────────────┐         ┌──────────────▼──────────────┐
│  server-1 (VSN CP)     │         │  server-2 (VSN Worker)       │
│  51.83.11.70           │         │  37.187.90.41                │
│  K8s: control-plane    │◄────────│  K8s: <none> (worker)        │
│  CRI: cri-o 1.30.10    │         │  CRI: cri-o 1.30.10          │
│  Runs: etcd, API server│         │  Runs: tenant workloads      │
└────────────────────────┘         └──────────────────────────────┘
```

Each node type requires a **different hardening profile**:

| Node | `node.role` in MongoDB | Profile Name | What Gets Applied |
|------|------------------------|--------------|-------------------|
| VSO | `vso_orchestrator` | `full` | iptables (5 port groups) + 12 NetworkPolicies + Thanos NodePort patch |
| server-1 | `vsn_control_plane` | `vsn_control_plane` | iptables (4 port groups: etcd + kubelet + node-exporter) |
| server-2 | `vsn_worker` | `vsn_worker` | iptables (2 port groups: kubelet + node-exporter only) |

---

## 4. Design Decisions

### Decision 1 — Postflight, not Preflight
Hardening runs **after** the cluster is fully deployed, not before. A cluster must exist before rules referencing its services can be applied. The Ansible playbook cannot patch Kubernetes NetworkPolicies on a cluster that doesn't exist yet.

**Flow:** `Inventory Upload → Provision → Deploy → Harden (postflight)`

### Decision 2 — Node Role from MongoDB, not Inventory Sheet
The hardening profile for each node is **auto-detected from MongoDB** at runtime (via `node.role` field). No changes to the inventory sheet format are required. Azsible queries MongoDB during the hardening step to determine which profile to apply per node.

This avoids: user error in inventory files, schema changes to existing inventory format, and extra configuration burden.

### Decision 3 — Convert bash script to proper Ansible playbook
`apply_security_hardening.sh` is converted to `playbooks/security-hardening.yml` using:
- `ansible.builtin.iptables` module (idempotent, not raw shell)
- `kubernetes.core.k8s` module for NetworkPolicies (idempotent, declarative)

**Why:** Idempotency (safe to re-run), testability (`ansible-lint`), maintainability, and consistent integration with existing Ansible infrastructure in azsible.

### Decision 4 — Harden stage inline in deploy pipeline UI
The hardening status surfaces as a **third stage** in the existing deploy pipeline in orch-client:

```
[ Provision ] ──── [ Deploy ] ──── [ Harden ]
     ✓ Passed          ✓ Passed      ⟳ Running
```

No separate Security tab is created in this phase. Status is visible where the operator already watches the deployment progress.

---

## 5. System Architecture

### End-to-End Flow

```
orch-client (React)
  │  User uploads inventory → triggers deploy
  ▼
orch_api_server
  │  Publishes to Redis channel "cluster-configs"
  │  action: "deploy"
  ▼
azsible (Python / Ansible Runner)
  │  Receives Redis message → runs deploy.yml
  │  On az_success == True:
  │    ├── Update inventory status → "Allocated"
  │    ├── Update AZ status → "Deployed"
  │    └── [NEW] Calls Handler.harden(inventory)
  │           │
  │           ├── Builds HardeningInventory (3 groups)
  │           ├── Runs security-hardening.yml via ansible_runner
  │           │     ├── VSO localhost → vso_orchestrator_hardening.yml
  │           │     ├── server-1     → vsn_control_plane_hardening.yml
  │           │     └── server-2     → vsn_worker_hardening.yml
  │           └── Reports per-node status → PUT /inventory/update_inventory/
  │                  "Hardened" or "HardenFailed"
  ▼
orch_api_server
  │  Stores security.status per node in MongoDB
  ▼
orch-client (HardenStage component)
  │  Polls GET /azs/security_status/{sc}/{region}/{az} every 3s
  │  Displays: pending → running → passed ✓ / failed ✗
  └── "Re-run Hardening" button → POST /azs/harden/{sc}/{region}/{az}
```

### Ansible Inventory Structure (Generated per Deploy)

```ini
[all:vars]
cluster_subnet=10.233.0.0/16
public_iface=eno1

[vso_orchestrator]
localhost  ansible_connection=local

[vsn_control_plane]
51.83.11.70  ansible_user=root  ansible_password=...

[vsn_worker]
37.187.90.41  ansible_user=root  ansible_password=...
               ansible_ssh_common_args='-o ProxyCommand="... @51.83.11.70"'
```

Workers are accessed via SSH ProxyCommand through the VSN control plane — identical to the existing deploy inventory pattern in `src/inventory.py`.

### MongoDB Schema Addition (per node document)

```json
{
  "inventory_name": "server-1",
  "status": "Hardened",
  "security": {
    "status": "passed",
    "profile": "vsn_control_plane",
    "checks": [
      { "name": "kubelet-9100-blocked", "passed": true, "error": null },
      { "name": "etcd-2379-blocked",    "passed": true, "error": null }
    ],
    "ran_at": "2026-05-26T14:30:00Z",
    "playbook_output": "..."
  }
}
```

Cluster-level `security.status` is an aggregate: `passed` only if **all 3 nodes** pass.

---

## 6. Implementation Plan

Execution order is wave-based. Wave 2 items run in parallel after Wave 1 completes.

```
Wave 1 ──────────────────────────────────────────────────────────────
  [01-01] Ansible Playbook (azsible/playbooks/)

Wave 2 ──────────────────────────────────────────────────────────────
  [01-02] azsible Python Handler     [01-03] orch_api_server API
  (azsible/src/)                     (orch_api_server/openapi/)
         │                                      │
         └──────────────────┬───────────────────┘
                            ▼
Wave 3 ──────────────────────────────────────────────────────────────
  [01-04] orch-client Pipeline UI
  (orch-client/src/)
```

---

### Phase 01-01: Ansible Hardening Playbook (azsible)

**Repo:** `vishanti/azsible`  
**Wave:** 1 (no dependencies)

#### Files Created

| File | Purpose |
|------|---------|
| `playbooks/security-hardening.yml` | Main entrypoint — 3 plays, one per node role |
| `playbooks/tasks/vso_orchestrator_hardening.yml` | Full profile: 5 iptables groups + 12 NetworkPolicies |
| `playbooks/tasks/vsn_control_plane_hardening.yml` | 4 iptables groups (etcd + kubelet + node-exporter) |
| `playbooks/tasks/vsn_worker_hardening.yml` | 2 iptables groups (kubelet + node-exporter) |
| `playbooks/group_vars/all.yml` | Default vars: `cluster_subnet`, `public_iface` |
| `playbooks/templates/hardening-inventory.ini.j2` | Jinja2 template for 3-group hardening inventory |

#### Playbook Structure

```yaml
# security-hardening.yml
- name: Security Hardening — VSO Orchestrator
  hosts: vso_orchestrator
  become: true
  gather_facts: true
  vars:
    host_ip: "{{ ansible_default_ipv4.address }}"
  tasks:
    - include_tasks: tasks/vso_orchestrator_hardening.yml

- name: Security Hardening — VSN Control Plane
  hosts: vsn_control_plane
  ...

- name: Security Hardening — VSN Worker
  hosts: vsn_worker
  ...
```

#### iptables Rule Translation (example)

```bash
# Original bash (apply_security_hardening.sh)
iptables -I INPUT -p tcp --dport 9100 -j DROP
iptables -I INPUT -p tcp --dport 9100 -s 10.233.0.0/16 -j ACCEPT
```

```yaml
# Converted to Ansible (idempotent)
- name: Block node-exporter from public
  ansible.builtin.iptables:
    chain: INPUT
    protocol: tcp
    destination_port: "9100"
    jump: DROP
    state: present

- name: Allow node-exporter from cluster subnet
  ansible.builtin.iptables:
    chain: INPUT
    protocol: tcp
    destination_port: "9100"
    source: "{{ cluster_subnet }}"
    jump: ACCEPT
    state: present
```

#### Acceptance Criteria
- `ansible-lint` passes on all 6 files
- `ansible-playbook --syntax-check` passes
- Idempotent: second run produces `changed=0`
- All 12 NetworkPolicy manifests present in VSO task file
- No hardcoded IPs or credentials

---

### Phase 01-02: azsible Python Hardening Handler

**Repo:** `vishanti/azsible`  
**Wave:** 2 (depends on 01-01)

#### Files Modified / Created

| File | Change |
|------|--------|
| `src/hardening_inventory.py` | NEW — `HardeningInventory` class |
| `src/handler.py` | ADD `harden_path` to `__init__`, ADD `harden()` method, CALL from `deploy()` |

#### HardeningInventory Class

New class mirrors the existing `Inventory` class pattern. Takes an existing `Inventory` object and renders `hardening-inventory.ini.j2` to a temp file.

#### Handler.harden() — Postflight Hook

```python
def harden(self, inventory):
    hardening_inv = HardeningInventory(inventory)
    inventory_file = hardening_inv.write_temp_inventory()
    try:
        az = AnsibleExecutor(inventory_file, self.harden_path)
        # Report per-node results
        for node in az.result["successful"]:
            inventory.update_inventory_status(node, "Hardened")
        for node in az.result["failed"] | az.result["unreachable"]:
            inventory.update_inventory_status(node, "HardenFailed")
    finally:
        hardening_inv.delete_temp_inventory()
```

#### Integration Point in deploy()

```python
# Existing code (handler.py ~line 107)
if az_success:
    inventory.update_az_status("Deployed", "Deployment completed")
    Mongo.update_status_config(inventory.config_id, "Deployed")
    # NEW: postflight hardening
    self.harden(inventory)
```

Hardening is only triggered on **successful deploys**. `update()` and `delete()` are not touched.

#### Acceptance Criteria
- `HardeningInventory` importable without errors
- `Handler.harden()` executes without crashing on valid inventory
- `deploy()` calls `harden()` inside `if az_success:` block only
- Per-node status reported to orch_api_server via existing HTTP pattern

---

### Phase 01-03: orch_api_server Security Status API

**Repo:** `vishanti/orch_api_server`  
**Wave:** 2 (depends on 01-01, runs parallel with 01-02)

#### Schema Change

Add optional `security` subdocument to the inventory node model:

```python
class SecurityStatus(BaseModel):
    status: Optional[str] = None   # pending | running | passed | failed
    profile: Optional[str] = None  # vso_orchestrator | vsn_control_plane | vsn_worker
    checks: Optional[List[dict]] = []
    ran_at: Optional[datetime] = None
    playbook_output: Optional[str] = None
```

Existing inventory documents without this field remain valid (optional field).

#### New API Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `PUT` | `/inventory/update_inventory/{sc_name}/{name}` | **Extended** — now accepts `"Hardened"` and `"HardenFailed"` as valid status values |
| `GET` | `/inventory/security_status/{sc_name}/{inventory_name}` | Returns per-node security state |
| `POST` | `/azs/harden/{sc_name}/{region_name}/{az_name}` | Manually trigger re-hardening (returns 202) |
| `GET` | `/azs/security_status/{sc_name}/{region_name}/{az_name}` | Returns per-node list + aggregate status |

#### Cluster Security Status Response

```json
{
  "az_name": "az-la",
  "aggregate_status": "passed",
  "nodes": [
    {
      "inventory_name": "vso",
      "ip": "51.83.11.105",
      "security": { "status": "passed", "profile": "vso_orchestrator", "ran_at": "..." }
    },
    {
      "inventory_name": "server-1",
      "ip": "51.83.11.70",
      "security": { "status": "passed", "profile": "vsn_control_plane", "ran_at": "..." }
    },
    {
      "inventory_name": "server-2",
      "ip": "37.187.90.41",
      "security": { "status": "passed", "profile": "vsn_worker", "ran_at": "..." }
    }
  ]
}
```

`aggregate_status` = `passed` only if all 3 nodes are `passed`. `failed` if any node is `HardenFailed`. `pending` otherwise.

#### Acceptance Criteria
- `PUT /inventory/update_inventory/` accepts `"Hardened"` status without 422 error
- `GET /azs/security_status/` returns correct per-node data
- `POST /azs/harden/` returns 202 and Redis message is published
- All endpoints require Bearer token auth (consistent with existing API)
- No regression on existing inventory CRUD

---

### Phase 01-04: orch-client Pipeline Stage UI

**Repo:** `yogesh_sandbox/orch-client`  
**Wave:** 3 (depends on 01-02 + 01-03)

#### New Component

`src/Components/HardenStage/index.js` — self-contained component that:

1. Renders the 3-stage pipeline: `Provision → Deploy → Harden`
2. Derives stage state from `azStatus` prop:

| `azStatus` | Provision | Deploy | Harden |
|---|---|---|---|
| `In Progress` | ⟳ Running | ⏳ Pending | ⏳ Pending |
| `Deployed` | ✓ Passed | ✓ Passed | ⟳ Polling... |
| `Hardened` | ✓ Passed | ✓ Passed | ✓ Passed |
| `HardenFailed` | ✓ Passed | ✓ Passed | ✗ Failed + Re-run button |
| `Failed` | ✓ Passed | ✗ Failed | — Skipped |

3. Polls `GET /azs/security_status/` every **3 seconds** when `azStatus == "Deployed"`
4. Stops polling when aggregate status is `passed` or `failed`
5. Shows per-node breakdown (collapsed by default):

```
[ Harden ] ✓ Passed  ▼ (expand)
  ├── vso      (VSO Orchestrator)     ✓
  ├── server-1 (VSN Control Plane)    ✓
  └── server-2 (VSN Worker)           ✓
```

#### New Redux Actions

```javascript
getAZSecurityStatus(sc_name, region_name, az_name)  // GET polling
triggerAZHarden(sc_name, region_name, az_name)       // POST re-run
```

#### Integration Point

Added to `src/pages/Availabilityzones/ViewAZ.js` (or equivalent AZ detail view):

```jsx
<HardenStage
  scName={azData.sc_name}
  regionName={azData.region_name}
  azName={azData.az_name}
  azStatus={azData.status}
  dispatch={dispatch}
/>
```

Uses `iconsax-react` icons (`TickCircle`, `CloseCircle`, `Clock`) and reactstrap `Spinner` — **no new UI library dependencies**.

#### Acceptance Criteria
- 3-stage pipeline visible in AZ detail view after any deploy
- Polling starts automatically when `azStatus == "Deployed"`
- Interval cleared on component unmount (no memory leaks)
- Re-run button visible only on `HardenFailed` state
- Per-node breakdown expandable
- No regression on existing AZ dashboard

---

## 7. Port / Firewall Rules Reference

### VSO Orchestrator (`vso_orchestrator` profile) — Full

| Port | Service | Rule | Allowed Sources |
|------|---------|------|----------------|
| 9100 | node-exporter | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |
| 2379 | etcd client | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |
| 2380 | etcd peer | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |
| 10250 | kubelet | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |
| 27017 | MongoDB | DROP on public iface `eno1` | None from public |

### VSN Control Plane (`vsn_control_plane` profile)

| Port | Service | Rule | Allowed Sources |
|------|---------|------|----------------|
| 9100 | node-exporter | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |
| 2379 | etcd client | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |
| 2380 | etcd peer | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |
| 10250 | kubelet | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |

### VSN Worker (`vsn_worker` profile)

| Port | Service | Rule | Allowed Sources |
|------|---------|------|----------------|
| 9100 | node-exporter | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |
| 10250 | kubelet | DROP all, ACCEPT from cluster/localhost | `10.233.0.0/16`, `127.0.0.1`, host IP |

---

## 8. Network Policy Reference

Applied on VSO only (manages the Vishanti management stack namespaces). VSN tenant NetworkPolicies are tenant-specific and out of scope for this phase.

### `monitoring` namespace

| Policy Name | Allows Traffic To | From |
|---|---|---|
| `default-deny-ingress` | All pods | Nobody (default deny) |
| `allow-prometheus-to-node-exporter` | `node-exporter:9100` | `prometheus` pods |
| `allow-thanos-query-internal` | `thanos-query:10902,10901` | `grafana`, `query-frontend`, `prometheus` |
| `allow-prometheus-internal` | `prometheus:9090` | `grafana`, `thanos-query`, `alertmanager` |
| `allow-grafana-from-ingress` | `grafana:3000` | Traefik pods in `ingress-system` |

### `vishanti-orchestrator` namespace

| Policy Name | Allows Traffic To | From |
|---|---|---|
| `default-deny-ingress` | All pods | Nobody (default deny) |
| `allow-orch-client-from-ingress` | `orch-client:3000` | Traefik in `ingress-system` |
| `allow-apiserver-internal` | `apiserver:8080` | Traefik, `syncer-manager`, `orch-syncer`, `azsible` |
| `allow-keycloak-internal` | `keycloak:8080` | Traefik, `apiserver` |
| `allow-mongodb-internal` | `mongodb:27017` | `apiserver`, `syncer-manager`, `orch-syncer` |
| `allow-redis-internal` | `redis:6379` | `apiserver`, `syncer-manager`, `orch-syncer` |
| `allow-mysql-internal` | `mysql:3306` | `keycloak` |

### Additional Patches (Traefik + Thanos)

- **Traefik dashboard:** `api.insecure=false` — dashboard no longer accessible on port `31048`
- **Thanos Query:** NodePort `31090` → `ClusterIP` on port `9090`

---

## 9. Out of Scope (Deferred)

These items were identified during design but are **intentionally excluded** from this phase:

| Item | Reason Deferred |
|------|----------------|
| Kubernetes API (`6443`) firewall restriction | Requires OVH private networking / VRack setup — infrastructure-level change |
| Traefik rate limiting (`IngressRoute` middleware) | Belongs in a dedicated Traefik configuration phase |
| CPU/memory resource limits for all workloads | Separate resource-management phase — needs per-service profiling |
| NodePort → Ingress migration for frontend | Requires DNS + TLS certificate changes across all tenants |
| Persistent Security compliance dashboard | Ongoing state (not just deploy-time) — future phase after this one ships |

---

## 10. Open Questions for Mentor

These items require review before execution begins:

1. **NetworkPolicy scope on VSN clusters** — Should VSN clusters also receive default-deny NetworkPolicies for their tenant workload namespaces? Currently this phase only applies NetworkPolicies on VSO (management stack). VSN tenant policies would need to know which namespaces exist per tenant.

2. **Hardening on update/delete** — Currently `Handler.harden()` is only called after a successful **deploy**. Should re-hardening also run when a worker node is **added to an existing cluster** (update action)? If a tenant scales out a worker, the new node would not be hardened until the next full deploy.

3. **iptables rule ordering** — The original script uses `iptables -I` (INSERT at position 1) which means rules are inserted in reverse order to work correctly. The Ansible `iptables` module with `state: present` appends rules. Does this create a different rule ordering that could affect existing iptables chains on production nodes?

4. **SSH key vs password auth** — The hardening inventory uses `ansible_password` (same as deploy inventory pattern). Is SSH key-based auth preferred for hardening, or is password auth acceptable given the credentials are already stored in MongoDB?

5. **Keycloak initContainer patch** — `apply_security_hardening.sh` line 375 patches a Keycloak `initContainer` for PV permission fix. Is this a security hardening concern or an operational fix? Should it be included in the hardening playbook or moved to the deploy playbook?

6. **Re-harden frequency** — Should there be a scheduled job that re-runs hardening periodically (e.g., weekly) to detect drift? Or is deploy-time + manual re-run sufficient for now?

---

*Page generated from planning artifacts in `.planning/phases/01-security-hardening-postflight/`*  
*Implementation plans: `01-01-PLAN.md` through `01-04-PLAN.md`*
