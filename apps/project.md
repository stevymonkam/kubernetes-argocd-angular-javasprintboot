# Structure Complète des Repositories GitOps

## 1. Architecture Globale des Repositories

### Vue d'Ensemble
```
REPOSITORIES CODE SOURCE (Séparés par équipe)
├── team-alpha-ecommerce-frontend/     # Code source Team Alpha
├── team-alpha-ecommerce-backend/      # Code source Team Alpha  
├── team-beta-billing-service/         # Code source Team Beta
├── team-beta-payment-gateway/         # Code source Team Beta
└── devops-infrastructure-code/        # Code Terraform/Ansible DevOps

REPOSITORY GITOPS UNIQUE (Partagé mais contrôlé)
└── company-gitops-repo/               # Manifests K8s de TOUTES les équipes
```

## 2. Stratégies de Gestion Possible

### Option A: Repository GitOps Unique avec Contrôle d'Accès (RECOMMANDÉ)

```
company-gitops-repo/  (Repository unique, accès contrôlé par dossiers)
├── infrastructure/                    # DevOps: Write, Others: Read
│   ├── base/
│   │   ├── ingress-controller.yaml
│   │   ├── cert-manager.yaml
│   │   └── monitoring.yaml
│   └── overlays/
│       ├── dev/
│       ├── staging/
│       └── prod/
├── applications/
│   ├── team-alpha/                    # Team Alpha: Write, Others: Read
│   │   ├── ecommerce-frontend/
│   │   │   ├── base/
│   │   │   │   ├── deployment.yaml
│   │   │   │   ├── service.yaml
│   │   │   │   └── kustomization.yaml
│   │   │   └── overlays/
│   │   │       ├── dev/
│   │   │       ├── staging/
│   │   │       └── prod/
│   │   └── ecommerce-backend/
│   │       └── ...
│   └── team-beta/                     # Team Beta: Write, Others: Read
│       ├── billing-service/
│       └── payment-gateway/
├── argocd-config/                     # DevOps: Write, Others: Read
│   ├── projects/
│   │   ├── infra-project.yaml
│   │   ├── team-alpha-project.yaml
│   │   └── team-beta-project.yaml
│   └── applications/
│       ├── infra-apps/
│       ├── team-alpha-apps/
│       └── team-beta-apps/
└── shared-services/                   # Cloud + DevOps: Write, Others: Read
    ├── monitoring/
    ├── security/
    └── networking/
```

#### Permissions GitHub/GitLab sur company-gitops-repo:
```yaml
Repository: company-gitops-repo
├── Branch protection: main (require PR + reviews)
├── CODEOWNERS file:
│   ├── /infrastructure/               @devops-team
│   ├── /applications/team-alpha/      @team-alpha-leads @devops-team  
│   ├── /applications/team-beta/       @team-beta-leads @devops-team
│   ├── /argocd-config/               @devops-team
│   └── /shared-services/             @cloud-team @devops-team
└── Team Permissions:
    ├── DevOps: Admin (peut merger partout)
    ├── Team Alpha: Write (peut créer PR sur /applications/team-alpha/)
    ├── Team Beta: Write (peut créer PR sur /applications/team-beta/) 
    ├── Cloud: Write (peut créer PR sur /shared-services/)
    └── Managers: Read (lecture seule)
```

### Option B: Repositories GitOps Séparés par Équipe

```
REPOSITORIES GITOPS MULTIPLES
├── infrastructure-gitops/             # DevOps seulement
│   ├── clusters/
│   ├── networking/
│   └── monitoring/
├── team-alpha-gitops/                # Team Alpha + DevOps review
│   ├── ecommerce-frontend/
│   └── ecommerce-backend/
├── team-beta-gitops/                 # Team Beta + DevOps review  
│   ├── billing-service/
│   └── payment-gateway/
└── argocd-bootstrap/                 # DevOps seulement
    ├── projects/
    └── root-applications/
```

## 3. Structure Détaillée des Repositories Code

### Repository Team Alpha - Ecommerce Frontend
```
team-alpha-ecommerce-frontend/
├── src/                              # Code applicatif
│   ├── components/
│   ├── pages/
│   └── utils/
├── tests/
├── Dockerfile                        # Build de l'image
├── .github/workflows/                # CI/CD Pipeline
│   ├── build-and-test.yml           # Tests + Build image
│   └── deploy.yml                   # Met à jour GitOps repo
├── k8s-templates/                   # Templates pour GitOps
│   ├── deployment.template.yaml
│   ├── service.template.yaml  
│   └── configmap.template.yaml
├── helm-chart/                      # Optionnel: Chart Helm
└── scripts/
    └── update-gitops.sh            # Script pour mettre à jour GitOps
```

### Repository DevOps Infrastructure
```
devops-infrastructure-code/
├── terraform/                       # Infrastructure as Code
│   ├── clusters/
│   ├── networking/
│   └── monitoring/
├── ansible/                        # Configuration Management
├── helm-charts/                    # Charts Helm custom
│   ├── monitoring-stack/
│   └── ingress-setup/
├── scripts/
│   └── bootstrap-cluster.sh
└── .github/workflows/
    ├── terraform-plan.yml
    └── terraform-apply.yml
```

## 4. Workflow de Développement Complet

### Cas d'Usage: Team Alpha déploie une nouvelle version

#### 1. Développement (Repository Code)
```bash
# Développeur Team Alpha travaille sur team-alpha-ecommerce-frontend/
git checkout -b feature/new-login
# ... développe la feature
git commit -m "Add new login feature"
git push origin feature/new-login
```

#### 2. CI/CD Pipeline (Repository Code)
```yaml
# .github/workflows/deploy.yml dans team-alpha-ecommerce-frontend/
name: Build and Deploy
on:
  push:
    branches: [main]
    
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    # Build et push image Docker
    - name: Build Image  
      run: |
        docker build -t company/ecommerce-frontend:${{ github.sha }} .
        docker push company/ecommerce-frontend:${{ github.sha }}
    
    # Met à jour le repository GitOps
    - name: Update GitOps
      run: |
        git clone https://github.com/company/company-gitops-repo
        cd company-gitops-repo
        
        # Met à jour l'image dans dev
        yq eval '.spec.template.spec.containers[0].image = "company/ecommerce-frontend:${{ github.sha }}"' \
          -i applications/team-alpha/ecommerce-frontend/overlays/dev/deployment.yaml
          
        git add .
        git commit -m "Update ecommerce-frontend dev image to ${{ github.sha }}"
        git push
```

#### 3. ArgoCD Detection et Déploiement
```yaml
# ArgoCD détecte le changement dans company-gitops-repo
# Application ecommerce-frontend-dev se synchronise automatiquement
# Nouveau deployment avec la nouvelle image
```

#### 4. Promotion vers Staging (Manuel)
```bash
# Team Alpha Leader fait une PR pour promouvoir vers staging
cd company-gitops-repo
git checkout -b promote-ecommerce-frontend-staging

# Copie la version dev vers staging
cp applications/team-alpha/ecommerce-frontend/overlays/dev/deployment.yaml \
   applications/team-alpha/ecommerce-frontend/overlays/staging/

# Crée PR - DevOps review et merge
# ArgoCD déploie automatiquement en staging
```

## 5. Contrôles de Sécurité par Couches

### Couche 1: Repository Git (GitHub/GitLab)
```
Qui peut faire quoi sur les repositories:
├── team-alpha-ecommerce-frontend/    # Team Alpha: Admin
├── team-beta-billing-service/        # Team Beta: Admin  
├── company-gitops-repo/              # Contrôlé par CODEOWNERS
│   ├── /applications/team-alpha/     # Team Alpha: Write via PR
│   ├── /applications/team-beta/      # Team Beta: Write via PR
│   └── /infrastructure/              # DevOps: Write direct
└── devops-infrastructure-code/       # DevOps: Admin
```

### Couche 2: ArgoCD AppProjects
```
Restrictions sur ce qui peut être déployé:
├── team-alpha-project:
│   ├── sourceRepos: [company-gitops-repo]      # SEUL repo autorisé
│   ├── destinations: [alpha-* namespaces]      # SEULS namespaces autorisés  
│   └── resources: [Deployment, Service...]     # SEULES ressources autorisées
├── team-beta-project: # Restrictions similaires pour team-beta
└── infra-project: # Permissions étendues pour DevOps
```

### Couche 3: ArgoCD RBAC
```
Qui peut manipuler quoi dans ArgoCD UI:
├── team-alpha-developers:
│   ├── CAN: sync/rollback team-alpha applications
│   └── CANNOT: touch team-beta or infra applications
├── devops-team:
│   └── CAN: everything (admin role)
└── managers:
    └── CAN: view only (readonly)
```

### Couche 4: Kubernetes RBAC
```
Permissions au niveau cluster K8s:
├── argocd-team-alpha ServiceAccount:
│   ├── CAN: deploy in alpha-* namespaces
│   └── CANNOT: access kube-system or other teams' namespaces
└── argocd-devops ServiceAccount:
    └── CAN: cluster-admin (infrastructure management)
```

## 6. Exemple de Structure de Projet ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-alpha-project
spec:
  description: "Team Alpha Applications - Ecommerce Platform"
  
  # RESTRICTION: SEUL repository autorisé  
  sourceRepos:
  - 'https://github.com/company/company-gitops-repo'
  
  # RESTRICTION: SEULS namespaces/clusters autorisés
  destinations:
  - namespace: 'team-alpha-dev'
    server: https://kubernetes.default.svc
  - namespace: 'team-alpha-staging'  
    server: https://kubernetes.default.svc
  - namespace: 'team-alpha-prod'
    server: https://kubernetes.default.svc
  
  # RESTRICTION: SEULS paths autorisés dans le repo GitOps
  sourceNamespaces:
  - 'applications/team-alpha/*'  # Ne peut accéder qu'à leurs dossiers
  
  # RESTRICTION: Ressources autorisées
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: 'Deployment'
  - group: ''
    kind: 'Service'
  - group: ''
    kind: 'ConfigMap'
  - group: ''  
    kind: 'Secret'
  - group: 'networking.k8s.io'
    kind: 'Ingress'
  
  # RESTRICTION: Aucune ressource cluster autorisée
  clusterResourceWhitelist: []
  
  # RESTRICTION: Fenêtres de déploiement
  syncWindows:
  - kind: 'deny'
    schedule: '0 2 * * *'         # Maintenance 2h-3h
    duration: 1h
    applications:
    - '*-prod'                    # Bloque prod seulement
    
  roles:
  - name: team-lead
                      # Accès complet pour team lead
    policies:
    - p, proj:team-alpha-project:team-lead, applications, *, team-alpha-project/*, allow
    groups:
    - team-alpha-leads
    
  - name: developer
    policies:
    - p, proj:team-alpha-project:developer, applications, get, team-alpha-project/*, allow
    - p, proj:team-alpha-project:developer, applications, sync, team-alpha-project/*-dev, allow
    - p, proj:team-alpha-project:developer, applications, sync, team-alpha-project/*-staging, allow  
    - p, proj:team-alpha-project:developer, applications, sync, team-alpha-project/*-prod, deny
    groups:
    - team-alpha-developers    # Devs peuvent sync dev/staging mais pas prod
```

## 7. Résumé de l'Architecture de Contrôle

### Les Projects ArgoCD définissent les "RÈGLES DU JEU":
- ✅ Quels repositories peuvent être utilisés
- ✅ Dans quels namespaces/clusters déployer  
- ✅ Quels types de ressources K8s sont autorisés
- ✅ Qui peut faire quoi (RBAC)
- ✅ Quand les déploiements sont autorisés

### Les Permissions Git définissent "QUI PEUT MODIFIER":
- ✅ Qui peut modifier les manifests GitOps
- ✅ Processus de review/approbation
- ✅ Protection des branches
- ✅ Audit trail des modifications

### Ensemble, ils créent une matrice de permissions:

| Équipe | Repos Code | Repos GitOps | ArgoCD Projects | Résultat |
|---------|------------|--------------|----------------|----------|
| **Team Alpha** | ✅ Admin sur leurs repos | ✅ Write sur `/applications/team-alpha/` | ✅ team-alpha-project | 🎯 Autonomie totale sur leurs apps |
| **Team Beta** | ✅ Admin sur leurs repos | ✅ Write sur `/applications/team-beta/` | ✅ team-beta-project | 🎯 Autonomie totale sur leurs apps |
| **DevOps** | ✅ Review sur tous | ✅ Admin sur tout | ✅ Tous projects | 🎯 Gouvernance et infrastructure |
| **Cloud** | ❌ Pas d'accès code | ✅ Write sur `/shared-services/` | ✅ Lecture + infra réseau | 🎯 Gestion réseau/sécurité |

Cette architecture garantit que chaque équipe a l'autonomie nécessaire tout en respectant les contraintes de sécurité et de gouvernance !