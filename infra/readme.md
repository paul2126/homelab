# homelab cd-manifests

Unified configuration repository for clusters.

## Structure

```
.
└── infra/
    ├── base/                     # Application Source Definition
    │   └── app/
    │       ├── kustomization.yaml   # <-- 1. GENERATOR (Can references Helm Chart)
    │       └── app-values-base.yaml # Base values for the Helm Chart
    │
    └── overlays/                 # Environment-Specific Customizations
        └── prod/
            └── app/
                ├── kustomization.yaml   # <-- 2. DEPLOYMENT TARGET (References Base & Patches)
                └── prod-patch-storage.yaml # Patch specific resources (e.g., set storageClass)
```