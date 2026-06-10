# Homelab

GitOps configuration for the homelab Kubernetes cluster, managed by **ArgoCD with native Helm**.

Structure inspired by the `fai-devops` repo: each app is a local Helm "umbrella" chart,
and ArgoCD renders Helm directly (no Kustomize, no `--enable-helm`).

## How it works

```
You push to Git (github.com/paul2126/homelab.git, branch main)
        │
        ▼
ArgoCD watches the repo
        │
        ▼
Two root "app of apps" Applications (applied once with kubectl):
   apps/templates/cluster-setup/cluster-setup-application.yaml   (platform)
   apps/templates/workloads/workloads-application.yaml           (workloads)
        │  each syncs its own folder of plain Application manifests
        ▼
   apps/templates/cluster-setup/gpu-operator.yaml ──► deployments/gpu-operator (Helm umbrella chart)
   apps/templates/cluster-setup/cilium-lb.yaml ► deployments/cilium-lb  (plain CR manifests)
   ...                                        ──► ...
        │
        ▼
ArgoCD runs `helm dependency build` + `helm template` and applies the result.
```

Two legible layers: **root → Application → Helm chart**. No Kustomize, no overlays.

> **The cluster and its CNI (Cilium) are provisioned by Kubespray, not this repo.** This
> repo manages applications on top. The only Cilium-related thing it owns is the
> LoadBalancer IP pool + L2 announcement policy CRs (`deployments/cilium-lb/`, plain
> manifests applied by a directory source) — Cilium itself is installed and configured via
> Kubespray.

## Layout

```
homelab/
├── apps/
│   └── templates/
│       ├── cluster-setup/                     # platform group
│       │   ├── cluster-setup-application.yaml  # root #1 (app of apps)
│       │   ├── argo-cd.yaml                    # Application: argocd (self-managing)
│       │   ├── cilium-lb.yaml                  # Application: cilium-lb (LB pool + L2 policy CRs)
│       │   ├── gpu-operator.yaml                # Application: gpu-operator (NVIDIA GPU stack)
│       │   ├── envoy-gateway.yaml               # Application: envoy-gateway (L7 ingress)
│       │   ├── monitoring.yaml                  # Application: monitoring (VictoriaMetrics + Grafana)
│       │   └── external-secrets.yaml            # Application: external-secrets (ESO + Infisical store)
│       └── workloads/                           # workloads group
│           ├── workloads-application.yaml        # root #2 (app of apps)
│           ├── n8n.yaml                           # Application: n8n
│           ├── vllm-llama.yaml                    # Application: vllm-llama (Llama 3.1 8B serving)
│           ├── infisical.yaml                     # Application: infisical (secrets manager)
│           └── immich.yaml                        # Application: immich (photos; library on TrueNAS)
└── deployments/                                 # umbrella charts + plain-manifest dirs
    ├── argo-cd/        { Chart.yaml, Chart.lock, values.yaml }
    ├── cilium-lb/      { loadbalancer-ippool.yaml, l2-announcement-policy.yaml }   # plain CRs
    ├── gpu-operator/   { Chart.yaml, Chart.lock, values.yaml }
    ├── envoy-gateway/  { Chart.yaml, Chart.lock, values.yaml, templates/ (GatewayClass, Gateway, EnvoyProxy) }
    ├── n8n/            { Chart.yaml, Chart.lock, values.yaml, templates/ (postgres, httproute) }
    ├── vllm-llama/     { deployment.yaml, service.yaml, pvc.yaml, httproute.yaml }   # plain manifests
    ├── monitoring/     { Chart.yaml, Chart.lock, values.yaml, dashboards/, templates/ (VMServiceScrape, HTTPRoute, dashboard CM) }
    ├── infisical/      { Chart.yaml, Chart.lock, values.yaml, templates/ (postgres, redis, httproute) }
    ├── immich/         { Chart.yaml, Chart.lock, values.yaml, templates/ (postgres, library-nfs, httproute) }
    ├── external-secrets/ { Chart.yaml, Chart.lock, values.yaml, templates/ (Infisical ClusterSecretStore) }
    ├── hami/                 # PARKED — chart kept, no Application (superseded by gpu-operator)
    └── nvidia-device-plugin/ # PARKED — chart kept, no Application (superseded by gpu-operator)
```

### The umbrella chart pattern

Each `deployments/<app>/` is a real Helm chart that declares the upstream chart as a
**dependency** in `Chart.yaml`:

```yaml
# deployments/gpu-operator/Chart.yaml
apiVersion: v2
name: gpu-operator
version: 0.1.0
dependencies:
  - name: gpu-operator
    version: v26.3.2
    repository: https://helm.ngc.nvidia.com/nvidia
```

Upstream values are **nested under the dependency name** in `values.yaml`:

```yaml
# deployments/gpu-operator/values.yaml
gpu-operator:           # <-- nested under the dependency name
  driver:
    enabled: true
```

Homelab-specific resources a chart doesn't ship (e.g. n8n's self-managed postgres) live in
that chart's own `templates/` directory. Plain cluster-scoped manifests that belong to no
chart — the Cilium LoadBalancer CRs — live in `deployments/cilium-lb/` and are applied by an
ArgoCD **directory source** (no `Chart.yaml`).

`charts/` and `*.tgz` are git-ignored — ArgoCD resolves dependencies at render time from
`Chart.lock`.

## Applications

**Every child Application is manual-sync** — deploys happen only on an explicit sync, so nothing
rolls out (or self-heals/prunes) on its own. The two app-of-apps **roots** stay automated so new
app files still register themselves; the child apps they create then wait for a manual sync.

| App | Chart | Version | Namespace | Sync |
|-----|-------|---------|-----------|------|
| argocd | argo-cd | 9.5.20 | argocd | Manual (self-managing) |
| cilium-lb | _plain manifests_ | — | kube-system | Manual |
| gpu-operator | gpu-operator | v26.3.2 | gpu-operator | Manual |
| envoy-gateway | gateway-helm | 1.8.1 | envoy-gateway-system | Manual |
| external-secrets | external-secrets | 2.6.0 | external-secrets | Manual |
| monitoring | victoria-metrics-k8s-stack | 0.82.0 | monitoring | Manual |
| n8n | n8n | 1.22.0 | n8n | Manual |
| vllm-llama | _plain manifests_ | vllm v0.22.1 | vllm | Manual |
| infisical | infisical-standalone | 1.9.0 | infisical | Manual |
| immich | immich | 0.12.0 | immich | Manual |

> **Cilium itself is not in this table** — the CNI is installed and managed by Kubespray.
> `cilium-lb` only carries the `CiliumLoadBalancerIPPool` + `CiliumL2AnnouncementPolicy` CRs.

> **`hami` and `nvidia-device-plugin` are parked** — their charts remain under `deployments/`
> but have no Application, so they're not deployed. The GPU Operator supersedes both (it ships
> its own device plugin; two device plugins conflict). See the GPU section below.

## Bootstrap

The cluster and its CNI (Cilium) are already provisioned by **Kubespray**, so bootstrap is
just ArgoCD + the roots:

```bash
# 1. Install ArgoCD (it then self-manages via the argocd Application).
helm dependency build deployments/argo-cd
helm upgrade --install argocd -n argocd --create-namespace \
  deployments/argo-cd -f deployments/argo-cd/values.yaml

# 2. Pre-create the n8n secrets (see "Secrets" below), then register everything
#    by applying the two roots once.
kubectl apply -f apps/templates/cluster-setup/cluster-setup-application.yaml
kubectl apply -f apps/templates/workloads/workloads-application.yaml
```

> **ArgoCD admin password:** the auto-generated `argocd-initial-admin-secret` was deleted after
> the admin password was rotated (`kubectl -n argocd delete secret argocd-initial-admin-secret`).
> It's a one-time bootstrap secret, not managed by Git, and isn't regenerated — log in with the
> rotated password. If you ever need a fresh one, reset it via the ArgoCD CLI/`argocd-secret`.

> For LoadBalancer IPs to work, the Cilium features must be on in Kubespray
> (`cilium_l2announcements: true`, `cilium_kube_proxy_replacement: true`, LB-IPAM) and
> `cilium_loadbalancer_ip_pools` left **empty** — the `cilium-lb` app owns the pool + L2
> policy CRs so they have a single owner.

> Applications reference `targetRevision: main`, so merge to `main` before (or while)
> bootstrapping. To test from a feature branch, temporarily set `targetRevision` to the
> branch name in the two roots and the child Applications.

## Adding a new app

1. Create `deployments/<app>/Chart.yaml` (depend on the upstream chart) and, if needed,
   `values.yaml` (nested under the dependency name) and `templates/` extras.
2. Run `helm dependency update deployments/<app>` to generate `Chart.lock`.
3. Add `apps/templates/<group>/<app>.yaml` (an ArgoCD Application pointing at
   `deployments/<app>`).
4. Push. The group's root app discovers it automatically.

## Secrets

Several apps reference existing secrets that are **not** stored in this repo. Create each app's
secrets in its namespace **before** that app syncs, or the pods fail to start. The manual-sync
apps (n8n, infisical, immich) won't sync until you trigger them, which gives you time to create
these first.

| Namespace | Secret | Required key(s) |
|-----------|--------|-----------------|
| `n8n` | `n8n-encryption-key` | `N8N_ENCRYPTION_KEY` |
| `n8n` | `n8n-postgresql` | `password` (the n8n DB user's password) |
| `monitoring` | `grafana-admin` | `admin-user`, `admin-password` |
| `infisical` | `infisical-postgresql` | `password` (the infisical DB user's password) |
| `infisical` | `infisical-secrets` | `ENCRYPTION_KEY`, `AUTH_SECRET`, `SITE_URL`, `DB_CONNECTION_URI`, `REDIS_URL` |
| `immich` | `immich-postgresql` | `password` (the immich DB user's password) |

n8n's Postgres is a self-managed `StatefulSet` using the official `postgres` image
(`deployments/n8n/templates/postgres.yaml`); n8n connects to it via `externalPostgresql`.
The single `n8n-postgresql` secret's `password` key is shared by both the database
container and n8n, so create it before the workloads root syncs:

```bash
kubectl create secret generic n8n-postgresql -n n8n \
  --from-literal=password="$(openssl rand -hex 24)"
kubectl create secret generic n8n-encryption-key -n n8n \
  --from-literal=N8N_ENCRYPTION_KEY="$(openssl rand -hex 24)"
```

**monitoring (Grafana admin):**

```bash
kubectl create namespace monitoring
kubectl create secret generic grafana-admin -n monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -hex 24)"
```

**infisical** — self-managed Postgres + Valkey (`deployments/infisical/templates/`); the app loads
`infisical-secrets` via `envFrom`. `DB_CONNECTION_URI`/`REDIS_URL` point at the in-cluster
services. SITE_URL must match the HTTPRoute host (HTTP until TLS is added):

```bash
kubectl create namespace infisical
PGPW="$(openssl rand -hex 24)"
kubectl create secret generic infisical-postgresql -n infisical \
  --from-literal=password="$PGPW"
kubectl create secret generic infisical-secrets -n infisical \
  --from-literal=ENCRYPTION_KEY="$(openssl rand -hex 16)" \
  --from-literal=AUTH_SECRET="$(openssl rand -base64 32)" \
  --from-literal=SITE_URL="http://infisical.attat.org" \
  --from-literal=DB_CONNECTION_URI="postgresql://infisical:${PGPW}@infisical-postgresql:5432/infisicalDB?sslmode=disable" \
  --from-literal=REDIS_URL="redis://infisical-redis:6379"
```

**immich** — self-managed VectorChord Postgres (`deployments/immich/templates/postgres.yaml`);
the `password` key is shared by the database and immich's `DB_PASSWORD`:

```bash
kubectl create namespace immich
kubectl create secret generic immich-postgresql -n immich \
  --from-literal=password="$(openssl rand -hex 24)"
```

> **Immich also needs the TrueNAS NFS library wired up** before its first sync:
> 1. Fill in `library.nfs.server` and `library.nfs.path` in `deployments/immich/values.yaml`
>    (the TrueNAS IP and the export path of the existing images dataset).
> 2. Ensure the NFS export allows the worker subnet and that the NFS client (`nfs-common`) is
>    installed on the nodes — otherwise the `immich-library` PV won't mount and immich-server
>    stays `Pending`.

> **Rotate the bootstrap secrets.** Any values generated/displayed during setup should be treated
> as compromised — see [`docs/secret-rotation.md`](docs/secret-rotation.md) for the per-secret
> rotation procedure (including the Infisical `ENCRYPTION_KEY`).

## Secrets management (External Secrets + Infisical)

`deployments/external-secrets/` deploys the **External Secrets Operator** (ESO) and a cluster-wide
`ClusterSecretStore` named `infisical` that reads from the in-cluster Infisical
(`http://infisical.infisical.svc.cluster.local:8080/api`). Once it's set up, any app can pull its
secrets from Infisical via an `ExternalSecret` instead of a hand-created `kubectl` Secret.

The `ClusterSecretStore` is **gated off** (`infisical.enabled: false`) until Infisical is
configured — it can't authenticate before a machine identity exists. To activate:

1. Sync the `external-secrets` Application (installs ESO + its CRDs).
2. In Infisical: create a project, then a **Machine Identity** with **Universal Auth**; give it
   **read** access to that project. Copy its Client ID + Client Secret.
3. Store the creds (kept out of Git):
   ```bash
   kubectl -n external-secrets create secret generic infisical-universal-auth \
     --from-literal=clientId='<client-id>' --from-literal=clientSecret='<client-secret>'
   ```
4. In `deployments/external-secrets/values.yaml` set `infisical.enabled: true` and fill
   `projectSlug` / `environmentSlug`, then re-sync `external-secrets`.

Apps then consume Infisical secrets like this (materialises a normal K8s Secret ESO keeps in sync):

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: { name: my-app, namespace: my-app }
spec:
  refreshInterval: 1h
  secretStoreRef: { kind: ClusterSecretStore, name: infisical }
  target: { name: my-app }            # the K8s Secret ESO creates
  dataFrom:
    - find: { path: "/my-app" }       # pull everything under that Infisical path
```

> The four **bootstrap** secrets (`grafana-admin`, `infisical-postgresql`, `infisical-secrets`,
> `infisical-universal-auth`) stay as plain k8s Secrets by necessity — Infisical and ESO must be
> running before they can serve anything, so their own credentials can't live inside Infisical.

## GPU & LLM serving

The GPU stack is the **NVIDIA GPU Operator** (`deployments/gpu-operator/`), which installs and
manages the driver, container-toolkit, device plugin, NFD, DCGM and GPU-feature-discovery.
`driver.enabled: true` — the operator builds/loads the NVIDIA driver on the GPU node (no host
prep; first rollout may reboot the node once). It replaces HAMi + the standalone device plugin
(both **parked**, since two device plugins conflict).

`vllm-llama` (`deployments/vllm-llama/`) serves **Llama 3.1 8B Instruct, 4-bit AWQ-INT4** via
vLLM with an OpenAI-compatible API. The model (`hugging-quants/Meta-Llama-3.1-8B-Instruct-AWQ-INT4`)
is a public repo — **no HuggingFace token needed**. In-cluster endpoint:

```
http://vllm-llama.vllm.svc.cluster.local:8000/v1     model id: llama-3.1-8b
```

n8n reaches it via its **OpenAI** node (set the Base URL to the address above). Tuning lives in
`deployments/vllm-llama/deployment.yaml` args (`--max-model-len`, `--gpu-memory-utilization`);
defaults target a 12 GB GPU.

> **Prerequisites:**
> 1. **Storage:** the **`local-path` StorageClass** (rancher.io/local-path) is the cluster
>    default and is provided by Kubespray (`local_path_provisioner_enabled: true` in
>    `addons.yml`) — it's cluster-level infra, so it belongs with Kubespray, not an ArgoCD app.
>    Most PVCs (n8n, vLLM, monitoring, the self-managed Postgres StatefulSets, immich's ML
>    cache) bind to it. Immich's photo library is the exception — it lives on a static NFS PV
>    pointing at TrueNAS (`deployments/immich/templates/library-nfs.yaml`), not local-path.
> 2. **GPU arch:** the GPU needs compute capability ≥ 7.5 (Turing+) for AWQ — confirm with
>    `nvidia-smi` once the operator is up.
> 3. **Driver mode:** `driver.enabled: true` assumes worker-1 has **no** host NVIDIA driver.
>    If `nvidia-smi` already works on the node, flip it to `false` (operator would conflict
>    with a host driver).
> 4. **Ordering:** `gpu-operator` must be healthy before `vllm-llama` can schedule — vLLM
>    stays `Pending` until `nvidia.com/gpu` is advertised, then self-resolves.

## Ingress (Envoy Gateway)

`deployments/envoy-gateway/` is a shared **L7 ingress** for the cluster — an umbrella chart
wrapping `gateway-helm` (Envoy Gateway), using the **Gateway API**. It provisions one Envoy
behind a `LoadBalancer` Service pinned to **`192.168.1.129`** (Cilium LB-IPAM), with an HTTP
listener on `:80` that accepts `HTTPRoute`s from any namespace.

Each service exposes itself by adding an `HTTPRoute` that targets the shared `homelab-gateway`.
vLLM does this in `deployments/vllm-llama/httproute.yaml` (with a 3600s timeout so streamed
generations aren't cut at Envoy's 15s default). External endpoint:

```
LLAMA_API_ENDPOINT=http://192.168.1.129/v1/chat/completions       # or http://vllm.attat.org/... with a DNS A-record
LLAMA_MODEL=llama-3.1-8b
```
> Note: the gateway listens on **port 80** and routes to vLLM's `:8000` — so the external port
> is 80, not 8000. To add a hostname, set `spec.hostnames` on the HTTPRoute and point a DNS
> A-record at `192.168.1.129`. TLS is a clean add-on later (an HTTPS:443 listener + a cert).

Apps attached to the gateway and their hostnames (point each DNS A-record at `192.168.1.129`,
or send the IP with a matching `Host:` header):

| Host | → Service:port | HTTPRoute |
|------|----------------|-----------|
| `n8n.attat.org` | `n8n:5678` | `deployments/n8n/templates/httproute.yaml` |
| `grafana.attat.org` | `grafana:80` | `deployments/monitoring/templates/httproute-grafana.yaml` |
| `infisical.attat.org` | `infisical:8080` | `deployments/infisical/templates/httproute.yaml` |
| `immich.attat.org` | `immich-server:2283` | `deployments/immich/templates/httproute.yaml` |

> n8n was migrated off the old (unused) `className: cilium` Ingress to an HTTPRoute here.

## Network

| Resource | IP / Range |
|----------|-----------|
| LoadBalancer IP pool (`cilium-lb`) | 192.168.1.128/25 |
| ArgoCD / Envoy gateway IPs | 192.168.1.128 / 192.168.1.129 |
| Pod CIDR (Kubespray default) | 10.233.64.0/18 |
| Control-plane / worker nodes | 192.168.1.31–33 / 192.168.1.34 |
| K8s API server | 192.168.1.31:6443 |
