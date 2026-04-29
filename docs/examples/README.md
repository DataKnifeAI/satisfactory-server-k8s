# Kubernetes examples (reference)

| File | Purpose |
|------|---------|
| `clusterip-envoy-backend.example.yaml` | Extra **ClusterIP** `Service` with the **same selector** as the game Deployment; Envoy Gateway routes to this (not to a redundant `LoadBalancer` on the primary Service). |
| `envoyproxy-kube-vip.example.yaml` | **EnvoyProxy** so the Envoy `Service` gets **`loadBalancerIP`** / **`status.loadBalancer.ingress`** for **kube-vip** L2 bind (not `externalIPs`-only). Replace `LOAD_BALANCER_IP`; must match `Gateway.spec.addresses`. |
| `gateway-tcp-udp.example.yaml` | **Gateway** (with **`infrastructure.parametersRef`** → EnvoyProxy) + **TCPRoute** / **UDPRoute** for Satisfactory ports (**7777** TCP+UDP, **7778** TCP). Replace `EXTERNAL_VIP` with an IP from your kube-vip (or MetalLB) pool. |

Apply **`deploy/`** for the app. For **Envoy + kube-vip** with concrete VIPs (DataKnife-style), use **`../deploy/envoy/`** (`kubectl apply -k deploy/envoy/`). These **`docs/examples/`** files are **placeholder** copies for other clusters or docs.
