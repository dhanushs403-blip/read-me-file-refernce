# Architecture Reconstruction: CloudFront + Cilium Gateway Automation

This document reconstructs the complete end-to-end flow of the Vishanti Cloud Platform's CloudFront whitelisting and routing system. It serves as a historical record and a technical blueprint for the current implementation.

## 1. The Core Objective
To deliver a **Sovereign Cloud Experience** where local VSN clusters are accelerated and secured by AWS CloudFront, while maintaining a single control plane (VSO) that manages dynamic firewall rules (Cilium) to block all non-CloudFront traffic.

---

## 2. End-to-End Traffic Flow

When a user accesses your application, the request travels through four distinct layers:

### Layer 1: Cloudflare (The Entry Point)
*   **Domain:** `cdn.incubera.xyz`
*   **Status:** **Grey Cloud (DNS-only)**. 
*   **Action:** Cloudflare acts as the DNS provider, resolving `cdn.incubera.xyz` to your AWS CloudFront distribution domain (e.g., `d123.cloudfront.net`). 
*   **Why Grey Cloud?** To avoid "Double Proxying" issues and allow CloudFront's own edge network and SSL termination to handle the traffic directly.

### Layer 2: Amazon CloudFront (The Accelerator)
*   **Requirement:** `cdn.incubera.xyz` is added as an **Alternate Domain Name (CNAME)** in the AWS console.
*   **SSL:** CloudFront handles the public-facing certificate for `cdn.incubera.xyz`.
*   **Origin:** Configured to point to `alb4.incubera.xyz`.
*   **Protocol:** CloudFront communicates with the Origin (VSN) over **HTTPS (Port 443)**.
*   **Action:** Forwards the `Host: cdn.incubera.xyz` header to the VSN Gateway.

### Layer 3: VSN Gateway (The Enforcement)
*   **ALB Endpoint:** `alb4.incubera.xyz` (mapped to VSN Public IP `51.83.11.70`).
*   **Cilium Firewall:** The `loadBalancerSourceRanges` on the Service are dynamically restricted to the ~100+ AWS CloudFront IP ranges.
*   **Action:** Cilium L3/L4 filtering rejects any packet not coming from a CloudFront Edge server.

### Layer 4: Kubernetes Application (The Backend)
*   **HTTPRoute:** Matches the header `Host: cdn.incubera.xyz`.
*   **Action:** Forwards the request to the `bp3` service pods.

---

## 3. Configuration & Domain Matrix

| Role | Domain Name | Points To | Logic Layer |
| :--- | :--- | :--- | :--- |
| **Public URL** | `cdn.incubera.xyz` | CloudFront Distribution | User Access |
| **Origin Endpoint** | `alb4.incubera.xyz` | `51.83.11.70` (VSN) | AWS -> VSN Bridge |
| **SSL Validation** | `*.incubera.xyz` | VSN IP (via DNS-01) | cert-manager |
| **Control Plane** | `webhook.incubera.xyz` | `51.83.11.105` (VSO) | SNS -> Webhook |

---

## 4. Automation Flow (The "Brain")

The whitelisting is not static; it is handled by the **VSO Webhook Orchestrator**:

1.  **Event:** AWS publishes an `AmazonIpSpaceChanged` signal via SNS.
2.  **Notification:** SNS triggers a POST request to `https://webhook.incubera.xyz/webhook/cloudfront-ip-update`.
3.  **Validation:** The Webhook (running FastAPI on VSO) verifies the AWS cryptographic signature.
4.  **Discovery:** The Webhook finds all VSN resources labeled with `vishanti.io/provider: cloudfront`.
5.  **Enforcement:** The Webhook uses the `vsn-kubeconfig` secret to remotely patch the `CiliumGatewayClassConfig` on the VSN, updating the firewall CIDRs immediately.

---

## 5. "Hidden" Steps & Hurdles Solved

During implementation, several non-obvious hurdles were overcome:

*   **HTTP-01 vs DNS-01:** Traditional Let's Encrypt validation failed because Traefik was redirecting Port 80, which CloudFront doesn't always handle gracefully during validation. We shifted to **DNS-01 via Cloudflare API** to allow certificate issuance even when the backend is locked down.
*   **Redundant Domain Collision:** The certificate `alb4-cert` initially failed because it requested both `alb4.incubera.xyz` and `*.incubera.xyz`. The solution was to unify them under a single wildcard or specific name request to avoid ACME "redundant domain" errors.
*   **SNS SSL Policy:** AWS SNS requires a valid, publicly trusted SSL certificate (no self-signed) to deliver notifications. This necessitated setting up `cert-manager` on the VSO *before* the webhook could receive real SNS signals.
*   **Origin Host Header:** We ensured CloudFront sends the custom `cdn.incubera.xyz` Host header to the Origin, allowing the Cilium `HTTPRoute` to match and route the traffic correctly.

---

## 6. Replication Guide (The "Recipe")

To replicate this setup for a new tenant or domain:

1.  **DNS:** Create a Grey Cloud CNAME in Cloudflare pointing to the CloudFront distribution.
2.  **AWS:** Add the domain to CloudFront's "Alternate Domain Names" and configure the Origin to point to your VSN's ALB DNS.
3.  **VSN Metadata:** Label your `CiliumGatewayClassConfig` with `vishanti.io/provider: cloudfront`.
4.  **VSN Routing:** Create an `HTTPRoute` matching the new domain name.
5.  **Validation:** Run the VSO `force-update` script to immediately sync the new tenant's whitelist.
