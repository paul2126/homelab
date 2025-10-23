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

## app.yaml explaination
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  # 1. APPLICATION NAME: Unique name for your application in ArgoCD (e.g., app-name-prod)
  name: my-app-prod
  # 2. ARGO CD NAMESPACE: This is the namespace where the ArgoCD components are running (typically 'argocd')
  namespace: argocd 
  # Optional: Labels for organization
  labels:
    app.kubernetes.io/name: my-app
    environment: prod
spec:
  # 3. PROJECT: The ArgoCD Project this application belongs to (usually 'default')
  project: default

  # --- SOURCE (Where to find the config) ---
  source:
    # 4. REPO URL: The URL of your Git repository
    repoURL: https://github.com/my-org/gitops-repo.git
    
    # 5. PATH: The path to your Kustomize OVERLAY directory in the repository
    path: my-app/overlays/prod 
    
    # 6. TARGET REVISION: The branch, tag, or commit SHA to deploy
    targetRevision: main # Or a specific tag/SHA for a production environment

    # 7. KUSTOMIZE (Indicates Kustomize is the tool)
    kustomize:
      # Optional: You can specify Kustomize-specific settings here, 
      # but it's best practice to put them in kustomization.yaml itself.
      # namePrefix: prod- # Example: A Kustomize option applied globally

  # --- DESTINATION (Where to deploy) ---
  destination:
    # 8. SERVER: The API server URL of your target Kubernetes cluster
    server: https://kubernetes.default.svc # Use this for the cluster ArgoCD is running in
    
    # 9. NAMESPACE: The Kubernetes namespace where the resources will be deployed
    namespace: my-app-prod-namespace

  # --- SYNC POLICY (How ArgoCD manages sync) ---
  syncPolicy:
    # 10. AUTOMATIC SYNC: Set to {} to enable automatic synchronization
    automated:
      prune: true     # Automatically remove resources no longer defined in Git
      selfHeal: true  # Automatically sync if the cluster state deviates from Git state
      allowEmpty: false # Prohibit sync if no manifests are found

    # 11. SYNC OPTIONS: Important for Kustomize/Helm
    syncOptions:
      # Automatically create the destination namespace if it doesn't exist
    - CreateNamespace=true
```

## issue on wrapping helm with kustomize
https://github.com/kubernetes-sigs/kustomize/issues/4658