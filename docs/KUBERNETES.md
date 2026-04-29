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

Full Envoy / kube-vip / DNS layout lives in **`DataKnifeAI/gitops-tools`**: see `docs/GAME_SERVERS_ENVOY.md` and bundle `game-servers-exposure/overlays/prd-apps/`.

## kube-vip: VIP must appear on `loadBalancer`, not only `externalIPs`

With **Envoy Gateway** and a **`Gateway`** that has **both TCP and UDP** listeners plus **`spec.addresses`**, the generated Envoy **`Service`** can end up with **`spec.externalIPs`** populated while **`status.loadBalancer` stays empty**. **kube-vip** (service mode) programs **L2 bind + ARP** from **`status.loadBalancer.ingress`** / **`spec.loadBalancerIP`**, not from `externalIPs` alone, so the VIP may **never show on the node `eth0`** and clients see no path to the address.

**Fix:** add a namespaced **`EnvoyProxy`** (`gateway.envoyproxy.io/v1alpha1`) referenced from the **`Gateway`** via **`spec.infrastructure.parametersRef`**, with the same IP in `spec.provider.kubernetes.envoyService.loadBalancerIP` as in `Gateway.spec.addresses`, and set **`externalTrafficPolicy: Cluster`** on that Envoy service so traffic can reach Envoy **pods on workers** while kube-vip holds the VIP on **control-plane** nodes.

See **`docs/examples/envoyproxy-kube-vip.example.yaml`** and the updated **`docs/examples/gateway-tcp-udp.example.yaml`** (`infrastructure.parametersRef` on the `Gateway`).

## Where manifests live (this repo vs gitops-tools)

| Location | Role |
|----------|------|
| **`deploy/`** in this repo | Portable app: PVC, Deployment, primary **ClusterIP** `Service`. No cluster-specific VIPs. |
| **`docs/examples/`** here | **Reference** snippets (placeholders, optional names) for platform teams; not tied to one Fleet path. |
| **`gitops-tools` `game-servers-exposure/`** | **DataKnife production** bundle: concrete names, VIPs, Fleet `paths`, lives next to other cluster add-ons. |

That split avoids coupling the open-source game chart to our Rancher Fleet repo while still documenting the integration.

## Examples in this repo

See **`docs/examples/`** for copy-paste patterns (**EnvoyProxy** for kube-vip, Envoy backend `Service`, `Gateway` + routes). Treat them as **reference**; wire VIPs from your kube-vip pool and keep routes in sync with `deploy/` Service port names.

**Duplicate content?** The same *ideas* appear in **`gitops-tools`** as apply-ready YAML (`game-servers-exposure/overlays/prd-apps/`). This repo keeps **lighter examples** with placeholders (`EXTERNAL_VIP`, `LOAD_BALANCER_IP`); gitops holds the **canonical prd-apps** copies. Maintain behavior in one place for production (**gitops-tools**); update examples here when the pattern changes.
