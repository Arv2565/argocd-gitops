# ArgoCD GitOps Repository

This is the **single source of truth** for all applications managed by ArgoCD. The [setup script](https://github.com/Arv2565/argocd-monitoring-setup) creates a single root Application that points to the `apps/` folder in this repo — ArgoCD discovers and deploys everything else automatically.

## Repository Structure

```
argocd-gitops/
├── apps/                        # ArgoCD Application CRDs (App of Apps pattern)
│   └── <app-name>.yaml
│
├── <app-name>/                  # Manifests or Helm values for each app
│   └── manifest.yaml            #   (raw manifests)
│   └── values.yaml              #   (Helm values override)
│
└── README.md
```

## How It Works

1. The setup script creates **one** root ArgoCD Application pointing to `apps/`
2. ArgoCD clones this repo and finds all Application CRDs inside `apps/`
3. Each Application CRD tells ArgoCD where to find the chart/manifests and values
4. ArgoCD syncs and deploys every app to the cluster
5. Every 3 minutes, ArgoCD re-checks this repo for changes

No changes to the setup script, Prometheus, or Grafana are needed when adding or removing apps — everything is driven by this repo.

## Adding a New App

### Raw manifests app

For apps where you write the Kubernetes YAML yourself (Deployments, Services, ConfigMaps, etc.):

1. Create a folder with your manifests:
   ```
   my-app/
   └── manifest.yaml       # All resources in one file, separated by ---
   ```

2. Create the Application CRD in `apps/`:
   ```yaml
   # apps/my-app.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: my-app
     namespace: argocd
     labels:
       app.kubernetes.io/part-of: gitops-apps
     finalizers:
       - resources-finalizer.argocd.argoproj.io
   spec:
     project: default
     source:
       repoURL: https://github.com/Arv2565/argocd-gitops.git
       targetRevision: master
       path: my-app
     destination:
       server: https://kubernetes.default.svc
       namespace: my-app
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

3. Commit and push. ArgoCD picks it up automatically.

### Helm chart app

For apps deployed from an upstream Helm repository with your own values override:

1. Create a values override file:
   ```
   my-helm-app/
   └── values.yaml
   ```

2. Create the Application CRD in `apps/`:
   ```yaml
   # apps/my-helm-app.yaml
   apiVersion: argoproj.io/v1alpha1
   kind: Application
   metadata:
     name: my-helm-app
     namespace: argocd
     labels:
       app.kubernetes.io/part-of: gitops-apps
     finalizers:
       - resources-finalizer.argocd.argoproj.io
   spec:
     project: default
     sources:
       - repoURL: https://charts.example.com        # Upstream Helm repo URL
         chart: my-chart                             # Chart name
         targetRevision: 1.x.x                      # Chart version
         helm:
           valueFiles:
             - $values/my-helm-app/values.yaml       # Path to your values in this repo
       - repoURL: https://github.com/Arv2565/argocd-gitops.git
         targetRevision: master
         ref: values                                 # Referenced by $values above
     destination:
       server: https://kubernetes.default.svc
       namespace: my-helm-app
     syncPolicy:
       automated:
         prune: true
         selfHeal: true
       syncOptions:
         - CreateNamespace=true
   ```

3. Commit and push.

## Removing an App

Delete the Application CRD from `apps/` and optionally its values/manifest folder, then commit and push. ArgoCD will:

1. Detect the Application CRD was removed from Git
2. Delete the Application from the cluster (due to `prune: true` on the root app)
3. Clean up all managed resources (due to the `resources-finalizer`)

## Notes

- The `targetRevision: master` in all Application CRDs must match your repo's default branch — update to `main` if that's what your repo uses
- For local KIND deployments, consider disabling persistence (`persistence.enabled: false`) in Helm values to avoid PVC issues
- Each app automatically gets a tile in the Grafana monitoring dashboard — no dashboard reconfiguration needed
