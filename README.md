# kubernetes-argocd-angular-javasprintboot


```yaml
ton-projet-gitops/
├── apps/
│   ├── frontend/
│   │   ├── base/
│   │   │   └── deployment.yaml
│   │   │   └── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   └── kustomization.yaml
│   │       ├── staging/
│   │       │   └── kustomization.yaml
│   │       └── prod/
│   │           └── kustomization.yaml
│
│   ├── backend/
│   │   ├── base/
│   │   │   └── deployment.yaml
│   │   │   └── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   └── kustomization.yaml
│   │       ├── staging/
│   │       │   └── kustomization.yaml
│   │       └── prod/
│   │           └── kustomization.yaml
│
│   └── mysql/
│       ├── base/
│       │   └── deployment.yaml
│       │   └── service.yaml
│       │   └── pvc.yaml
│       │   └── kustomization.yaml
│       └── overlays/
│           ├── dev/
│           │   └── kustomization.yaml
│           ├── staging/
│           │   └── kustomization.yaml
│           └── prod/
│               └── kustomization.yaml
│
├── projects/
│   ├── dev-proj.yaml
│   ├── staging-proj.yaml
│   └── prod-proj.yaml
│
└── argocd-apps/
    ├── frontend-dev.yaml
    ├── backend-dev.yaml
    ├── mysql-dev.yaml
    ├── frontend-staging.yaml
    ├── backend-staging.yaml
    ├── mysql-staging.yaml
    ├── frontend-prod.yaml
    ├── backend-prod.yaml
    └── mysql-prod.yaml
```

```yaml
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: angular-new-project
  namespace: argocd
spec:
  description: Projet Angular + Java Spring Boot
  clusterResourceWhitelist:
  - group: ""
    kind: '*'
  destinations:
  - namespace: feature
    server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
  - group: ""
    kind: '*'
  - group: "apps"
    kind: "Deployment"
  sourceRepos:
  - https://github.com/stevymonkam/kubernetes-argocd-angular-javasprintboot.git
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: angular-new-app
  namespace: argocd
spec:
  project: angular-new-project
  source:
    repoURL: https://github.com/stevymonkam/kubernetes-argocd-angular-javasprintboot.git
    targetRevision: HEAD
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: feature
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```