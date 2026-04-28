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

## Examples in this repo

See **`docs/examples/`** for copy-paste patterns (Envoy backend `Service`, sample Gateway + routes). Treat them as **reference**; wire VIPs from your kube-vip pool and keep routes in sync with `deploy/` Service port names.
