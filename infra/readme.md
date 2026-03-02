# Homelab Infrastructure as Code

Unified GitOps configuration repository for the homelab Kubernetes cluster, using **ArgoCD + Kustomize + Helm**.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [How Helm and Kustomize Work Together](#how-helm-and-kustomize-work-together)
  - [What Helm Does](#what-helm-does)
  - [What Kustomize Does](#what-kustomize-does)
  - [The Integration: Kustomize Wraps Helm](#the-integration-kustomize-wraps-helm)
  - [Processing Order](#processing-order)
- [ArgoCD and the GitOps Pipeline](#argocd-and-the-gitops-pipeline)
  - [The App of Apps Pattern](#the-app-of-apps-pattern)
  - [How ArgoCD Syncs](#how-argocd-syncs)
- [File-by-File Breakdown](#file-by-file-breakdown)
  - [kustomization.yaml](#kustomizationyaml)
  - [values.yaml](#valuesyaml)
  - [app.yaml (ArgoCD Application)](#appyaml-argocd-application)
  - [Patches](#patches)
  - [Extra Resources](#extra-resources)
- [Base vs Overlays](#base-vs-overlays)
  - [What Overlays Do](#what-overlays-do)
  - [Types of Modifications in Overlays](#types-of-modifications-in-overlays)
- [Current Applications](#current-applications)
- [How to Add a New Application](#how-to-add-a-new-application)
- [Quick Reference](#quick-reference)
- [Known Issues](#known-issues)
- [Network Configuration](#network-configuration)

---

## Architecture Overview

The deployment pipeline works like this:

```
You push to Git (github.com/paul2126/homelab.git)
       |
       v
ArgoCD watches the repo on the `main` branch
       |
       v
top-level.yaml (the "App of Apps")
  discovers all **/app.yaml files under infra/base/
       |
       |--- argocd/app.yaml      --> syncs infra/overlays/prod/argocd
       |--- cilium/app.yaml      --> syncs infra/overlays/prod/cilium
       |--- hami/app.yaml        --> syncs infra/base/hami
       |--- nvidia/app.yaml      --> syncs infra/base/nvidia-device-plugin
       |
       v
ArgoCD runs:  kustomize build --enable-helm <path>
       |
       v
Kustomize downloads the Helm chart, renders it with values.yaml,
merges in extra resources, applies patches
       |
       v
Final Kubernetes manifests are applied to the cluster
```

Key point: **you never run `helm install` directly**. Kustomize handles Helm rendering, and ArgoCD handles Kustomize. You only edit files in Git and push.

---

## Repository Structure

```
homelab/
├── README.md                              # IP range notes
└── infra/
    ├── readme.md                          # This file
    ├── base/                              # Base configurations for each app
    │   ├── top-level.yaml                 # ArgoCD "App of Apps" entry point
    │   │
    │   ├── argocd/                        # ArgoCD (GitOps controller)
    │   │   ├── app.yaml                   #   ArgoCD Application manifest
    │   │   ├── kustomization.yaml         #   Kustomize config (pulls Helm chart)
    │   │   ├── values.yaml                #   Helm values overrides
    │   │   └── argocd-cm.yaml             #   Patch: enables --enable-helm flag
    │   │
    │   ├── cilium/                        # Cilium (CNI / networking)
    │   │   ├── app.yaml                   #   ArgoCD Application manifest
    │   │   ├── kustomization.yaml         #   Kustomize config (Helm + extra resources)
    │   │   ├── values.yaml                #   Helm values overrides
    │   │   ├── loadbalancer-ippool.yaml   #   Custom resource: LB IP pool
    │   │   └── l2-announcement-policy.yaml#   Custom resource: L2 announcements
    │   │
    │   ├── hami/                          # HAMi (GPU sharing)
    │   │   ├── app.yaml                   #   ArgoCD Application manifest
    │   │   └── kustomization.yaml         #   Kustomize config (Helm chart, no values)
    │   │
    │   └── nvidia-device-plugin/          # NVIDIA device plugin
    │       ├── app.yaml                   #   ArgoCD Application manifest
    │       ├── kustomization.yaml         #   Kustomize config (pulls Helm chart)
    │       └── values.yaml                #   Helm values overrides
    │
    └── overlays/                          # Environment-specific overrides
        └── prod/                          # Production environment
            ├── argocd/
            │   └── kustomization.yaml     #   References base argocd
            └── cilium/
                ├── kustomization.yaml     #   References base cilium
                └── values.yaml            #   (currently empty, for future overrides)
```

---

## How Helm and Kustomize Work Together

### What Helm Does

Helm is a **package manager** for Kubernetes. Someone packages up all the YAML manifests for a complex app (Deployments, Services, ConfigMaps, RBAC, etc.) into a **chart** and publishes it to a registry. Instead of writing 50+ YAML files yourself, you use the chart and pass a `values.yaml` to customize it.

For example, the ArgoCD Helm chart contains hundreds of Kubernetes resources. You only need to specify what you want to change (like `service.type: LoadBalancer`), and the chart handles everything else with sensible defaults.

### What Kustomize Does

Kustomize is a **configuration management tool** built into `kubectl`. It lets you compose, patch, and overlay Kubernetes manifests without templating. You write a `kustomization.yaml` that says:
- "Take these base resources"
- "Apply these patches on top"
- "Set this namespace on everything"
- "Add these labels to everything"

### The Integration: Kustomize Wraps Helm

This repo uses Kustomize's `helmCharts` directive to **consume Helm charts directly inside Kustomize**. No Helm charts are stored locally in the repo — they're all downloaded from external Helm registries at build time.

The `helmCharts` field in `kustomization.yaml` tells Kustomize:
1. Download the specified chart from the remote Helm repository
2. Run `helm template` with the provided `values.yaml`
3. Treat the rendered output as Kubernetes manifests
4. Continue with normal Kustomize processing (patches, namespace overrides, etc.)

This gives you the best of both worlds:
- **Helm**: Pre-packaged, community-maintained application configs
- **Kustomize**: The ability to patch, overlay, and compose those configs with your own resources

### Processing Order

When ArgoCD (or `kustomize build --enable-helm`) processes a directory:

```
1. Read kustomization.yaml
2. Download and render Helm chart(s) via `helm template` using values.yaml
3. Load extra resources listed under `resources:`
4. Apply patches from `patches:`
5. Apply namespace, labels, and annotations overrides
6. Output final manifests
```

Patches can target **anything** in the combined output — resources from the Helm chart, your custom resources, or resources inherited from a base. Kustomize doesn't care where a resource came from; it pattern-matches by kind/name/group.

---

## ArgoCD and the GitOps Pipeline

### The App of Apps Pattern

`infra/base/top-level.yaml` is the single bootstrap entry point. It tells ArgoCD to recursively scan `infra/base/` and deploy every `app.yaml` it finds:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: top-level
  namespace: argocd
spec:
  project: default
  destination:
    name: in-cluster
    namespace: argocd
  source:
    repoURL: https://github.com/paul2126/homelab.git
    targetRevision: main
    path: infra/base
    directory:
      recurse: true
      include: '**/app.yaml'
```

This means **adding a new app is automatic** — just create a new folder under `infra/base/` with an `app.yaml`, and the top-level Application discovers it on the next sync.

### How ArgoCD Syncs

Each `app.yaml` defines an ArgoCD Application, which tells ArgoCD:
- **Where to find the config**: Git repo URL + path + branch
- **Where to deploy it**: Target cluster + namespace
- **How to sync**: Manual or automated, with options for pruning and self-healing

Sync policies used in this repo:

| App | Sync Mode | Prune | Self-Heal | Notes |
|-----|-----------|-------|-----------|-------|
| ArgoCD | Manual | No | No | Core infra, manual sync to avoid accidental breakage |
| Cilium | Manual | No | No | Core networking, manual sync for safety |
| NVIDIA Device Plugin | Automated | Yes | Yes | Auto-syncs, prunes removed resources, reverts manual changes |
| HAMi | Automated | Yes | Yes | Auto-syncs, prunes removed resources, reverts manual changes |

The `--enable-helm` flag is critical. It's configured via a patch on ArgoCD's ConfigMap (`argocd-cm.yaml`):

```yaml
data:
  kustomize.buildOptions: --enable-helm
```

Without this, ArgoCD would not process `helmCharts` directives in Kustomize files.

---

## File-by-File Breakdown

### kustomization.yaml

The core Kustomize configuration file. In this repo, it typically contains:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: <target-namespace>     # Applied to all resources

helmCharts:                        # Helm chart to render
  - name: <chart-name>            #   Chart name in the Helm repo
    repo: <helm-repo-url>         #   Helm repository URL
    releaseName: <release-name>   #   Name for the Helm release
    namespace: <namespace>        #   Namespace passed to helm template
    version: <chart-version>      #   Pinned chart version
    valuesFile: values.yaml       #   Path to your values overrides

resources:                         # (Optional) Additional manifests
  - extra-resource.yaml

patches:                           # (Optional) Modify rendered resources
  - path: my-patch.yaml
    target:
      kind: ConfigMap
      name: some-configmap
```

**Example from NVIDIA Device Plugin** (simplest case — just a Helm chart):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: nvidia-device-plugin

helmCharts:
  - name: nvidia-device-plugin
    repo: https://nvidia.github.io/k8s-device-plugin
    releaseName: nvdp
    namespace: nvidia-device-plugin
    version: 0.18.0
    valuesFile: values.yaml
```

**Example from Cilium** (Helm chart + extra custom resources):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: kube-system

resources:
  - loadbalancer-ippool.yaml
  - l2-announcement-policy.yaml

helmCharts:
  - name: cilium
    repo: https://helm.cilium.io/
    version: 1.18.2
    releaseName: cilium
    namespace: kube-system
    valuesFile: values.yaml
```

**Example from ArgoCD** (Helm chart + patch on Helm-generated resource):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd

patches:
  - path: argocd-cm.yaml
    target:
      kind: ConfigMap
      name: argocd-cm

helmCharts:
  - name: argo-cd
    repo: https://argoproj.github.io/argo-helm
    version: 8.1.3
    releaseName: argocd
    namespace: argocd
    valuesFile: values.yaml
```

### values.yaml

Standard Helm values file. Only override what you need — the chart's defaults handle the rest. Each chart has different available values; check with `helm show values <repo>/<chart>`.

**Example (ArgoCD)** — expose the server as a LoadBalancer:

```yaml
server:
  service:
    type: LoadBalancer
```

**Example (NVIDIA)** — restrict to GPU nodes:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: feature.node.kubernetes.io/gpu
          operator: In
          values:
          - "true"
```

**Example (Cilium)** — configure networking:

```yaml
operator:
  replicas: 2
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList:
      - 10.1.0.0/16
kubeProxyReplacement: true
k8sServiceHost: k8s-control.attat.org
k8sServicePort: 6443
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
l2announcements:
  enabled: true
devices: "eth+ ens+"
```

### app.yaml (ArgoCD Application)

Tells ArgoCD about each application. Every app needs one of these.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>                    # Unique name in ArgoCD
  namespace: argocd                   # Always argocd (where ArgoCD runs)
  finalizers:                         # (Optional) Cleanup on deletion
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default                    # ArgoCD project

  source:
    repoURL: https://github.com/paul2126/homelab.git
    targetRevision: main              # Git branch/tag/SHA
    path: <path-to-kustomize-dir>     # Path to overlay or base directory

  destination:
    server: https://kubernetes.default.svc   # Target cluster
    namespace: <target-namespace>            # Target namespace

  syncPolicy:
    automated:                        # (Optional) Enable auto-sync
      prune: true                     #   Delete resources removed from Git
      selfHeal: true                  #   Revert manual cluster changes
    syncOptions:
      - CreateNamespace=true          # Auto-create namespace if missing

  ignoreDifferences:                  # (Optional) Ignore runtime-managed fields
    - group: ""
      kind: ConfigMap
      name: cilium-config
      jsonPointers:
        - /data
```

**Key fields explained:**

| Field | Purpose |
|-------|---------|
| `metadata.name` | Unique identifier for the app in ArgoCD |
| `metadata.namespace` | Must be `argocd` (where ArgoCD itself runs) |
| `spec.source.path` | Points to either `infra/base/<app>` or `infra/overlays/prod/<app>` |
| `spec.destination.namespace` | The namespace where the app's resources get deployed |
| `syncPolicy.automated` | Enables auto-sync; omit for manual sync |
| `syncPolicy.automated.prune` | If `true`, ArgoCD deletes resources that no longer exist in Git |
| `syncPolicy.automated.selfHeal` | If `true`, ArgoCD reverts manual changes made directly in the cluster |
| `syncOptions.CreateNamespace` | Automatically creates the target namespace |
| `ignoreDifferences` | Tells ArgoCD to ignore specific fields that change at runtime (prevents false drift) |

### Patches

Patches modify resources **after** Helm rendering. This lets you customize Helm-generated resources without forking the chart.

**Strategic Merge Patch** (merge YAML on top of existing resource):

```yaml
# In kustomization.yaml:
patches:
  - path: my-patch.yaml
    target:
      kind: ConfigMap
      name: argocd-cm

# my-patch.yaml:
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  kustomize.buildOptions: --enable-helm    # Added/overridden field
```

**Inline JSON Patch** (surgical add/remove/replace operations):

```yaml
# In kustomization.yaml:
patches:
  - target:
      kind: Deployment
      name: my-deployment
    patch: |
      - op: replace
        path: /spec/replicas
        value: 3
```

### Extra Resources

Custom Kubernetes manifests that exist alongside Helm chart resources. Listed under `resources:` in `kustomization.yaml`.

**Example**: Cilium includes custom CRDs that aren't part of the Helm chart:

- `loadbalancer-ippool.yaml` — Defines the IP range for LoadBalancer services (192.168.1.128/25)
- `l2-announcement-policy.yaml` — Configures L2 ARP announcements for LoadBalancer IPs

---

## Base vs Overlays

### What Overlays Do

An overlay is a layer that sits on top of a base configuration. It **inherits everything** from the base and can add, modify, or remove things.

```
base/cilium/                         overlays/prod/cilium/
+------------------------+           +-------------------------------+
| kustomization.yaml     |           | kustomization.yaml            |
|   helmCharts:          |<----------+   resources:                  |
|     - cilium 1.18.2    | inherits  |     - ../../../base/cilium    |
|   resources:           |           |                               |
|     - ippool.yaml      |           |   (can add patches, extra     |
|     - l2policy.yaml    |           |    resources, overrides)      |
|   values.yaml          |           |                               |
+------------------------+           +-------------------------------+
```

**Current state**: The prod overlays mostly just reference the base without additional modifications. But the structure is in place for future environment-specific customization.

**When to use overlays vs base directly**:
- Apps that go through overlays: ArgoCD, Cilium (their `app.yaml` points to `infra/overlays/prod/...`)
- Apps that point to base directly: HAMi, NVIDIA (their `app.yaml` points to `infra/base/...`)

If you don't need environment-specific overrides, pointing directly to base is simpler.

### Types of Modifications in Overlays

#### 1. Patches (modify existing resources)

```yaml
# overlays/prod/cilium/kustomization.yaml
resources:
  - ../../../base/cilium

patches:
  - path: increase-replicas.yaml
    target:
      kind: Deployment
      name: cilium-operator
```

#### 2. Extra resources (add new manifests)

```yaml
resources:
  - ../../../base/grafana
  - prod-ingress.yaml           # Only exists in prod
  - prod-networkpolicy.yaml     # Extra security for prod
```

#### 3. Namespace overrides

```yaml
resources:
  - ../../../base/grafana

namespace: grafana-staging       # Override namespace for staging
```

#### 4. Common labels and annotations

```yaml
commonLabels:
  environment: prod

commonAnnotations:
  team: infrastructure
```

These get added to **every** resource in the output.

#### 5. Multiple environments example

If you wanted staging and prod:

```
infra/overlays/
├── prod/
│   └── grafana/
│       ├── kustomization.yaml        # resources: [../../../base/grafana]
│       └── prod-patch.yaml           # e.g., 3 replicas, 4Gi memory
└── staging/
    └── grafana/
        ├── kustomization.yaml        # resources: [../../../base/grafana]
        └── staging-patch.yaml        # e.g., 1 replica, 512Mi memory
```

Both inherit the same base (same Helm chart, same version) but can differ in replica counts, resource limits, feature flags, ingress hostnames, storage sizes, etc.

---

## Current Applications

| App | Helm Chart | Chart Version | Helm Repo | Namespace | Sync Mode |
|-----|-----------|---------------|-----------|-----------|-----------|
| ArgoCD | argo-cd | 8.1.3 | https://argoproj.github.io/argo-helm | argocd | Manual |
| Cilium | cilium | 1.18.2 | https://helm.cilium.io/ | kube-system | Manual |
| HAMi | hami | latest | https://project-hami.github.io/HAMi/ | hami | Automated |
| NVIDIA Device Plugin | nvidia-device-plugin | 0.18.0 | https://nvidia.github.io/k8s-device-plugin | nvidia-device-plugin | Automated |

---

## How to Add a New Application

### Step 1: Create the app directory

```
infra/base/<app-name>/
├── app.yaml              # ArgoCD Application manifest
├── kustomization.yaml    # Kustomize config with Helm chart
└── values.yaml           # Helm values overrides (optional)
```

### Step 2: Write `kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: <app-namespace>

helmCharts:
  - name: <chart-name>
    repo: <helm-repo-url>
    releaseName: <release-name>
    namespace: <app-namespace>
    version: <chart-version>
    valuesFile: values.yaml
```

To find chart details:

```bash
helm repo add <repo-name> <repo-url>
helm search repo <repo-name>/<chart-name> --versions
helm show values <repo-name>/<chart-name>    # See all available values
```

### Step 3: Write `values.yaml`

Override only what you need. The chart's defaults handle the rest.

### Step 4: (Optional) Add extra resources

If you need custom resources not included in the Helm chart:

```yaml
resources:
  - my-configmap.yaml
  - my-networkpolicy.yaml

helmCharts:
  - name: <chart-name>
    ...
```

### Step 5: Write `app.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <app-name>
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/paul2126/homelab.git
    targetRevision: main
    path: infra/base/<app-name>
  destination:
    server: https://kubernetes.default.svc
    namespace: <app-namespace>
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Step 6: Push to Git

ArgoCD's `top-level.yaml` auto-discovers all `**/app.yaml` files. Your new app will appear in ArgoCD on the next sync cycle.

### (Optional) Step 7: Add an overlay

If you need environment-specific overrides, create an overlay:

```
infra/overlays/prod/<app-name>/
└── kustomization.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../../base/<app-name>

# Add patches, extra resources, etc. here
```

Then update `app.yaml` to point to the overlay path instead of the base path.

---

## Quick Reference

| I want to... | Put it in... |
|---|---|
| Change a Helm chart setting (replicas, image, etc.) | `values.yaml` in the app's base directory |
| Modify a Helm-generated resource after rendering | `patches:` in `kustomization.yaml` |
| Add a resource that isn't in the Helm chart | `resources:` in `kustomization.yaml` + the YAML file |
| Override something for a specific environment | An overlay's `kustomization.yaml` with patches |
| Add a completely new app | New folder in `infra/base/` with `app.yaml` + `kustomization.yaml` |
| Pin a chart version | `version:` field in `helmCharts` section |
| Enable auto-sync for an app | Add `syncPolicy.automated` in `app.yaml` |
| Prevent ArgoCD from flagging runtime changes as drift | Add `ignoreDifferences` in `app.yaml` |

---

## Known Issues

- Kustomize Helm wrapping issue: https://github.com/kubernetes-sigs/kustomize/issues/4658

---

## Network Configuration

| Resource | IP / Range |
|----------|-----------|
| LoadBalancer IP pool | 192.168.1.128/25 |
| ArgoCD reserved IP | 192.168.1.128 |
| Pod CIDR (Cilium IPAM) | 10.1.0.0/16 |
| K8s API server | k8s-control.attat.org:6443 |
