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
   apps/templates/cluster-setup/hami.yaml    ──► deployments/hami       (Helm umbrella chart)
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
│       │   └── gpu-operator.yaml                # Application: gpu-operator (NVIDIA GPU stack)
│       └── workloads/                           # workloads group
│           ├── workloads-application.yaml        # root #2 (app of apps)
│           ├── n8n.yaml                           # Application: n8n
│           └── vllm-llama.yaml                    # Application: vllm-llama (Llama 3.1 8B serving)
└── deployments/                                 # umbrella charts + plain-manifest dirs
    ├── argo-cd/      { Chart.yaml, Chart.lock, values.yaml }
    ├── cilium-lb/    { loadbalancer-ippool.yaml, l2-announcement-policy.yaml }   # plain CRs
    ├── gpu-operator/ { Chart.yaml, Chart.lock, values.yaml }
    ├── n8n/          { Chart.yaml, Chart.lock, values.yaml, templates/postgres.yaml }
    ├── vllm-llama/   { deployment.yaml, service.yaml, pvc.yaml }   # plain manifests
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

| App | Chart | Version | Namespace | Sync |
|-----|-------|---------|-----------|------|
| argocd | argo-cd | 9.5.20 | argocd | Manual (self-managing) |
| cilium-lb | _plain manifests_ | — | kube-system | Automated (prune + self-heal) |
| gpu-operator | gpu-operator | v26.3.2 | gpu-operator | Automated (prune + self-heal) |
| n8n | n8n | 1.22.0 | n8n | Manual |
| vllm-llama | _plain manifests_ | vllm v0.22.1 | vllm | Automated (prune + self-heal) |

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

n8n references existing secrets that are **not** stored in this repo. Create them in the
`n8n` namespace before the workloads root syncs, or the n8n / postgres pods fail to start:

| Secret | Required key |
|--------|-------------|
| `n8n-encryption-key` | `N8N_ENCRYPTION_KEY` |
| `n8n-postgresql` | `password` (the n8n DB user's password) |

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
> 1. **Storage:** a **`local-path` StorageClass must exist** — the n8n *and* vLLM PVCs depend
>    on it, but the cluster currently has **no storage provisioner at all** (`kubectl get sc`
>    is empty). Add it in **Kubespray** (`local_path_provisioner_enabled: true` in
>    `addons.yml`, then re-run) — it's cluster-level infra, so it belongs with Kubespray, not
>    an ArgoCD app. Until then no PVC binds (n8n included).
> 2. **GPU arch:** the GPU needs compute capability ≥ 7.5 (Turing+) for AWQ — confirm with
>    `nvidia-smi` once the operator is up.
> 3. **Driver mode:** `driver.enabled: true` assumes worker-1 has **no** host NVIDIA driver.
>    If `nvidia-smi` already works on the node, flip it to `false` (operator would conflict
>    with a host driver).
> 4. **Ordering:** `gpu-operator` must be healthy before `vllm-llama` can schedule — vLLM
>    stays `Pending` until `nvidia.com/gpu` is advertised, then self-resolves.

## Network

| Resource | IP / Range |
|----------|-----------|
| LoadBalancer IP pool (`cilium-lb`) | 192.168.1.128/25 |
| Pod CIDR (Kubespray default) | 10.233.64.0/18 |
| Control-plane / worker nodes | 192.168.1.31–33 / 192.168.1.34 |
| K8s API server | 192.168.1.31:6443 |
