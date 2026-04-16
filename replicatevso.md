# CloudFront Webhook: Full Replication Guide

This guide reproduces the complete working CloudFront IP Whitelisting automation from an existing VSO node to a **new server**.

---

## Architecture Overview

```
AWS SNS (AmazonIpSpaceChanged)
       │ HTTPS POST
       ▼
[New Server: webhook.incubera.xyz]
  ┌────────────────────────────────────────┐
  │  Kubernetes (single-node)              │
  │  ┌──────────────┐   ┌───────────────┐  │
  │  │  Traefik      │──▶│  cloudfront-  │  │
  │  │  Ingress      │   │  webhook pod  │  │
  │  └──────────────┘   └──────┬────────┘  │
  │  ┌──────────────┐          │ kubectl   │
  │  │  Cert-Manager │          │ (via      │
  │  │  (DNS-01 TLS) │          │ kubeconfig│
  │  └──────────────┘          ▼           │
  └───────────────────────────────────────-┘
                               │
                    VSN Cluster (51.83.11.70)
                    Updates CiliumGatewayClassConfig
```

---

## Files You Need to Copy from the Existing Server

Before starting, gather these from your existing VSO node:

| File | Location on Existing VSO | Purpose |
|---|---|---|
| `vsn-webhook.kubeconfig` | `/root/vso-setup-payload/vsn-webhook.kubeconfig` | Credentials to connect to VSN |
| `main.py` | `/root/tenants/main.py` | Webhook application source |
| `requirements.txt` | `/root/tenants/requirements.txt` | Python dependencies |
| `Dockerfile` | `/root/tenants/Dockerfile` | Container build spec |

```bash
# Run on the EXISTING VSO to create a transfer bundle
tar -czf /tmp/webhook-payload.tar.gz \
  /root/vso-setup-payload/vsn-webhook.kubeconfig \
  /root/tenants/main.py \
  /root/tenants/requirements.txt \
  /root/tenants/Dockerfile

# Transfer to the new server (replace NEW_SERVER_IP)
scp /tmp/webhook-payload.tar.gz root@NEW_SERVER_IP:/root/
```

---

## Phase 1: New Server Prerequisites

### 1.1 — Verify Kubernetes is Running
```bash
kubectl get nodes
# Expected: single node in Ready state
kubectl get svc -n ingress-system | grep traefik
# Expected: ingress-traefik LoadBalancer with your PUBLIC IP
```

### 1.2 — Verify Helm is Installed
```bash
helm version
# If missing: curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 1.3 — Update DNS
In Cloudflare (or your DNS provider), update:
- **Record**: `webhook.incubera.xyz`
- **Type**: `A`
- **Value**: `<NEW_SERVER_PUBLIC_IP>`
- **Proxy**: Set to DNS-only (grey cloud) initially

---

## Phase 2: Extract Files on New Server

```bash
cd /root
tar -xzf webhook-payload.tar.gz --strip-components=1
# Organize them:
mkdir -p /root/tenants
mv vsn-webhook.kubeconfig /root/
mv main.py requirements.txt Dockerfile /root/tenants/
```

---

## Phase 3: Install Cert-Manager

### 3.1 — Install via Helm
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.2 \
  --set crds.enabled=true
```

### 3.2 — Wait for Cert-Manager to Be Ready
```bash
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=180s
```

### 3.3 — Create Cloudflare API Token Secret

> [!IMPORTANT]
> Use your **Cloudflare API Token** with `Zone:Zone:Read` and `Zone:DNS:Edit` permissions.
> You can view the token value from the existing server:
> `kubectl get secret cloudflare-api-token -n cert-manager -o jsonpath='{.data.api-token}' | base64 -d`

```bash
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token="<YOUR_CLOUDFLARE_API_TOKEN>" \
  -n cert-manager
```

### 3.4 — Create Let's Encrypt ClusterIssuer
```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: support@incubera.xyz
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        cloudflare:
          email: support@incubera.xyz
          apiTokenSecretRef:
            name: cloudflare-api-token
            key: api-token
EOF
```

> [!NOTE]
> We use **DNS-01** (not HTTP-01) because Traefik has a global HTTP→HTTPS redirect that breaks HTTP-01 challenges. DNS-01 works entirely through Cloudflare's API and does not need port 80 to be reachable.

---

## Phase 4: Build and Deploy the Webhook

### 4.1 — Build the Docker Image
```bash
cd /root/tenants
docker build -t cloudfront-webhook:v2.1 .
```

### 4.2 — Load Image into Containerd (for Kubernetes)
```bash
docker save cloudfront-webhook:v2.1 | ctr -n k8s.io images import -
```

### 4.3 — Create Namespace and VSN Kubeconfig Secret
```bash
kubectl create namespace cloudfront-webhook

kubectl create secret generic vsn-kubeconfig \
  --from-file=config=/root/vsn-webhook.kubeconfig \
  -n cloudfront-webhook
```

### 4.4 — Apply All Kubernetes Manifests
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudfront-webhook
  namespace: cloudfront-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cloudfront-webhook
  template:
    metadata:
      labels:
        app: cloudfront-webhook
    spec:
      containers:
      - name: cloudfront-webhook
        image: cloudfront-webhook:v2.1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
        env:
        - name: KUBECONFIG
          value: /etc/vsn-kubeconfig/config
        - name: GATEWAY_CONFIG_NAME
          value: cloudfront-alb-config
        volumeMounts:
        - name: vsn-kubeconfig
          mountPath: /etc/vsn-kubeconfig
          readOnly: true
      volumes:
      - name: vsn-kubeconfig
        secret:
          secretName: vsn-kubeconfig
---
apiVersion: v1
kind: Service
metadata:
  name: cloudfront-webhook
  namespace: cloudfront-webhook
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: cloudfront-webhook
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cloudfront-webhook
  namespace: cloudfront-webhook
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: ingress-traefik
  tls:
  - hosts:
    - webhook.incubera.xyz
    secretName: cloudfront-webhook-tls
  rules:
  - host: webhook.incubera.xyz
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cloudfront-webhook
            port:
              number: 80
EOF
```

---

## Phase 5: Configure VSN (Remote Kubernetes Cluster)

> [!NOTE]
> The VSN cluster endpoint is `https://51.83.11.70:6443`. These commands run from the **new server** using the kubeconfig credentials.

### 5.1 — Apply Cross-Namespace RBAC on VSN
```bash
cat <<EOF | kubectl --kubeconfig=/root/vsn-webhook.kubeconfig apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloudfront-webhook-global-role
rules:
  - apiGroups: ["cilium.io"]
    resources: ["ciliumgatewayclassconfigs"]
    verbs: ["get", "list", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cloudfront-webhook-global-binding
subjects:
  - kind: User
    name: kubernetes-admin
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cloudfront-webhook-global-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

> [!IMPORTANT]
> If this step fails, check that the kubeconfig file is correct by running:
> `kubectl --kubeconfig=/root/vsn-webhook.kubeconfig get nodes`

### 5.2 — Label the Tenant Gateway on VSN
```bash
kubectl --kubeconfig=/root/vsn-webhook.kubeconfig \
  label ciliumgatewayclassconfig cloudfront-alb-config \
  vishanti.io/provider=cloudfront \
  -n vishanti-alb3 --overwrite
```

---

## Phase 6: AWS SNS Subscription Update

### 6.1 — Add New Endpoint Subscription
```bash
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:806199016981:AmazonIpSpaceChanged \
  --protocol https \
  --notification-endpoint https://webhook.incubera.xyz/webhook/cloudfront-ip-update
```

### 6.2 — Remove Old Endpoint (Optional but Recommended)
In the AWS Console → SNS → Topics → `AmazonIpSpaceChanged` → Subscriptions:
- Find and delete any subscription pointing to the **old server's IP**.

> [!CAUTION]
> Do **not** delete the old subscription until the new server is confirmed healthy to avoid missing a CloudFront update event.

---

## Phase 7: Validation

Run these checks in order:

### ✅ Check 1: Pod is Running
```bash
kubectl get pods -n cloudfront-webhook
# Expected: STATUS = Running, READY = 1/1
```

### ✅ Check 2: TLS Certificate is Issued
```bash
kubectl get certificate -n cloudfront-webhook
# Expected: READY = True (may take 1-3 minutes after deployment)
```

### ✅ Check 3: Health Endpoint
```bash
curl https://webhook.incubera.xyz/health
# Expected: {"status": "ok"}
```

### ✅ Check 4: Force Update (End-to-End Test)
```bash
curl -X POST https://webhook.incubera.xyz/webhook/force-update \
     -H "Authorization: Bearer change-me-in-production"
# Expected: {"status": "success", "report": {"success": 1, "failed": 0, "errors": []}}
```

### ✅ Check 5: Verify on VSN
```bash
kubectl --kubeconfig=/root/vsn-webhook.kubeconfig \
  get ciliumgatewayclassconfig cloudfront-alb-config \
  -n vishanti-alb3 \
  -o jsonpath='{.spec.service.loadBalancerSourceRanges}' | python3 -m json.tool
# Expected: A large list of CloudFront CIDR ranges
```

---

## Common Pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Pod crashes with `ModuleNotFoundError: cryptography` | `requirements.txt` missing `cryptography` | Add `cryptography>=41.0.0` and rebuild |
| Certificate stuck in `pending` | DNS-01 solver cannot reach Cloudflare | Verify `cloudflare-api-token` secret is correct |
| `kubectl` from pod returns `Unauthorized` | kubeconfig credentials wrong or RBAC not applied on VSN | Re-check Phase 5 RBAC steps |
| Health check returns `503` | Pod is not ready yet or kubeconfig mount is wrong | `kubectl describe pod -n cloudfront-webhook` |
| SNS `SubscriptionConfirmation` not auto-confirmed | Wrong endpoint URL or Ingress not routed correctly | Confirm Ingress is pointing to the webhook service |
| Traefik 308 redirect breaking HTTP-01 cert challenge | Traefik global redirect | Always use DNS-01 (already configured in this guide) |

---

## Security Hardening (Recommended Post-Replication)

1. **Change the force-update token**: Set `FORCE_UPDATE_TOKEN` as a Kubernetes Secret instead of the default.
2. **Restrict kubeconfig permissions on VSN**: If possible, create a dedicated ServiceAccount on VSN for this webhook instead of using `kubernetes-admin`.
3. **Enable network policies**: Restrict the webhook pod to only communicate with the VSN API server IP.
