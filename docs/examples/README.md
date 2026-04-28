# Kubernetes examples (reference)

| File | Purpose |
|------|---------|
| `clusterip-envoy-backend.example.yaml` | Extra **ClusterIP** `Service` with the **same selector** as the game Deployment; Envoy Gateway routes to this (not to a redundant `LoadBalancer` on the primary Service). |
| `gateway-tcp-udp.example.yaml` | **Gateway** + **TCPRoute** / **UDPRoute** sketch for Satisfactory ports (**7777** TCP+UDP, **7778** TCP). Replace `EXTERNAL_VIP` with an IP from your kube-vip (or MetalLB) pool. |

Apply **`deploy/`** from the repo root for the app itself; these snippets are for **platform** teams integrating Envoy.

Production-ready manifests for the DataKnife cluster live in **`https://github.com/DataKnifeAI/gitops-tools`** (`game-servers-exposure/overlays/prd-apps`).
