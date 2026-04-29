# Kubernetes deployment notes

This repo ships a **minimal Kustomize stack** under `deploy/` (PVC, Deployment, Service). DataKnife runs it on RKE2 with **Harbor** images and **Envoy Gateway + kube-vip** for player-facing IPs.

## Apply the default manifest

```bash
kubectl apply -k deploy/
```

Create an optional **`ConfigMap`** from `.env` (see `.env.example`) and name it `satisfactory-server-env` if you use the same keys as `deploy/deployment.yaml`.

## Image registry and pulls

- Images default to **`harbor.dataknife.net/library/satisfactory-server-k8s`** with a **pinned short SHA** in `deploy/kustomization.yaml` (not `latest`).
- **`imagePullPolicy: Always`** on the pod so a fixed tag still picks up a **new digest** after CI pushes.
- Use a **`kubernetes.io/dockerconfigjson`** secret for Harbor if the project is private (`imagePullSecrets`).

## Service type: **ClusterIP** (not LoadBalancer)

The primary Service **`satisfactory-server`** is **`ClusterIP`**. A plain **`LoadBalancer`** on this Service often stays **`<pending>`** on bare metal without MetalLB; instead, expose ports through **Envoy Gateway** with a **kube-vip** VIP.

Traffic path:

```text
Player → DNS/VIP (Envoy) → TCPRoute/UDPRoute → ClusterIP *-envoy Service → Pod
```

## How players connect

| What players use | Value (DataKnife example) |
|------------------|---------------------------|
| DNS **A** record | `satisfactory.dataknife.net` → Envoy VIP (e.g. `192.168.14.185`) |
| Game **UDP** | **7777** |
| Game **TCP** | **7777** |
| Beacon **TCP** | **7778** |

Some clients only accept a raw **IP**: use the **same VIP** as the DNS target. Ensure firewalls allow **UDP and TCP 7777** and **TCP 7778** to that address.

## Envoy Gateway manifests (`deploy/envoy/`)

**Gateway API** exposure (ClusterIP backend, **EnvoyProxy** for kube-vip, `Gateway`, routes) lives under **`deploy/envoy/`**. It is **not** part of the default `kubectl apply -k deploy/` stack; apply when your cluster has Envoy Gateway + kube-vip:

```bash
kubectl apply -k deploy/envoy/
```

Edit **`deploy/envoy/envoyproxy.yaml`** and **`deploy/envoy/gateway.yaml`** if your VIP or namespace differs (`kustomization.yaml` sets `namespace: game-servers`).

## kube-vip: VIP must appear on `loadBalancer`, not only `externalIPs`

With **Envoy Gateway** and a **`Gateway`** that has **both TCP and UDP** listeners plus **`spec.addresses`**, the generated Envoy **`Service`** can end up with **`spec.externalIPs`** populated while **`status.loadBalancer` stays empty**. **kube-vip** (service mode) programs **L2 bind + ARP** from **`status.loadBalancer.ingress`** / **`spec.loadBalancerIP`**, not from `externalIPs` alone, so the VIP may **never show on the node `eth0`** and clients see no path to the address.

**Fix:** add a namespaced **`EnvoyProxy`** (`gateway.envoyproxy.io/v1alpha1`) referenced from the **`Gateway`** via **`spec.infrastructure.parametersRef`**, with the same IP in `spec.provider.kubernetes.envoyService.loadBalancerIP` as in `Gateway.spec.addresses`, and set **`externalTrafficPolicy: Cluster`** on that Envoy service so traffic can reach Envoy **pods on workers** while kube-vip holds the VIP on **control-plane** nodes.

See **`docs/examples/envoyproxy-kube-vip.example.yaml`** and **`docs/examples/gateway-tcp-udp.example.yaml`** for the same pattern with placeholders (`EXTERNAL_VIP`, `LOAD_BALANCER_IP`). Production-style values for DataKnife are in **`deploy/envoy/`**.

## Examples in this repo

- **`deploy/envoy/`** — apply-ready Kustomize for Envoy + kube-vip (edit VIPs as needed).
- **`docs/examples/`** — shorter reference copies with placeholders for forks or other clusters.

## Observability (Envoy path)

**`kubectl logs` / `kubectl exec` → `502` / proxy error dialing node `:10250`**  
The control plane cannot talk to that worker’s **kubelet**. Pods can still be **Running**; check the node, or read container logs via **SSH + `crictl`** on the host until kubelet access is fixed.

**Envoy → backend `Connection_refused` on TCP game ports**  
For **Satisfactory**, the dedicated server is expected to accept **TCP/UDP** on the published ports when up; if Envoy sees refused while the pod is ready, check **process health**, **port names** (`game-tcp` / `beacon`), and that the **`*-envoy` Service** endpoints match the pod. **Windrose** can refuse **TCP :7777** when **`USE_DIRECT_CONNECTION=false`** (invite-only) while Envoy still runs a **`TCPRoute`** — see the Windrose **`docs/KUBERNETES.md`** section *Envoy TCPRoute and invite-only*.
