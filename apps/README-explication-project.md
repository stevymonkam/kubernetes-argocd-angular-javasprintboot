# kubernetes-argocd-angular-javasprintboot
## stevy yep
## nb : pour que l'asyc sur arco fonctione il faut changer le nom de l'image que la pipeline a ecrit par le nom de l'image qui est dans base voir exemple dans overlays/dev/kustomization.yaml


pour deployer un projet et une app il faut : creer les namespace si neccessaire car c est creer automatiquement par argocd quand on creer un project

- cas du namespace prod : 
prendre le project prod-proj.yaml et l'appliquer avec la commande suivante :

```yaml
kubectl apply -f prod-proj.yaml
```
et la le project est cree et on peut deployer les apps
pour deployer les apps il faut :
prendre le app frontend-prod.yaml et l'appliquer avec la commande suivante :

```yaml
kubectl apply -f frontend-prod.yaml
```
et la l'app est deployer dans le namespace prod : il faut noter que ce project prod est utiliser pour tous les app du namespace prod cet a dire les app frontend-prod.yaml backend-prod.yaml et mysql-prod.yaml doivent etre creer a partire de ce project prod

en suite pour le backend il faut :
pour deployer le backend il faut :
prendre le app backend-prod.yaml et l'appliquer avec la commande suivante :

```yaml
kubectl apply -f backend-prod.yaml
```
et la l'app est deployer dans le namespace prod : il faut noter que ce project prod est utiliser pour tous les app du namespace prod

pour deployer le mysql il faut :
pour deployer le mysql il faut :
prendre le app mysql-prod.yaml et l'appliquer avec la commande suivante :

```yaml
kubectl apply -f mysql-prod.yaml
```
et la l'app est deployer dans le namespace prod : il faut noter que ce project prod est utiliser pour tous les app du namespace prod

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
│   │       │   └── service-patch.yaml
│   │       ├── staging/
│   │       │   └── kustomization.yaml
│   │       │   └── service-patch.yaml
│   │       └── prod/
│   │           └── kustomization.yaml
│           └── service-patch.yaml
│
│   ├── backend/
│   │   ├── base/
│   │   │   └── deployment.yaml
│   │   │   └── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   └── kustomization.yaml
│   │       │   └── service-patch.yaml
│   │       ├── staging/
│   │       │   └── kustomization.yaml
│   │       │   └── service-patch.yaml
│   │       └── prod/
│   │           └── kustomization.yaml
│           └── service-patch.yaml
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
│           │   └── service-patch.yaml
│           ├── staging/
│           │   └── kustomization.yaml
│           │   └── service-patch.yaml
│           └── prod/
│               └── kustomization.yaml
│               └── service-patch.yaml
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

# image utiliser pour le front : stevymonkam/front5:1.0
Excellente question — je vais te détailler étape par étape comment fonctionne un projet structuré comme ça dans un vrai workflow GitOps avec ArgoCD + Kustomize.

📊 Fonctionnement pas à pas de ce projet
📌 1️⃣ Structure logique : séparation base / overlays
base/ contient les manifests Kubernetes génériques et communs à tous les environnements (ex: Deployment, Service, PVC, etc).

overlays/ contient des personnalisations spécifiques à chaque environnement (dev, staging, prod), via des fichiers kustomization.yaml qui appliquent des patchs ou changent des valeurs (image tag, replicas…).

👉 But : éviter la duplication et rendre le déploiement maintenable et propre.

📌 2️⃣ Les AppProjects dans ArgoCD
Dans le dossier projects/
Chaque fichier comme dev-proj.yaml définit un contexte d’environnement :

Les namespaces et clusters autorisés

Les dépôts Git autorisés

Les types de ressources autorisées (Cluster / Namespace-scoped)

💡 Un AppProject par environnement (dev, staging, prod) pour bien cloisonner et sécuriser les déploiements.

📌 3️⃣ Les Applications ArgoCD
Dans le dossier argocd-apps/
Chaque fichier comme frontend-dev.yaml déclare :

Le projet ArgoCD (ex : dev-proj)

Le repo Git à utiliser

La branche et le dossier (path:) à surveiller

Le namespace Kubernetes cible

Le serveur (généralement https://kubernetes.default.svc)

💡 1 Application ArgoCD par app et par environnement (frontend-dev, backend-prod…)

📌 4️⃣ Le fonctionnement GitOps avec ArgoCD
📌 Scénario : tu veux déployer une nouvelle version en dev

CI Pipeline

Build image Docker

Push image dans le registre

Met à jour le apps/frontend/overlays/dev/kustomization.yaml → modifie newTag: v2.0.0

Commit & Push sur Git

CI commit cette modif dans la branche develop

ArgoCD

Surveille ce dossier et cette branche

Compare l’état du cluster avec l’état Git (via kubectl diff)

Si différent :

soit Argo sync automatique

soit demande validation manuelle

Applique les manifests générés par Kustomize (kustomize build overlays/dev) et déploie sur le cluster Kubernetes

📌 5️⃣ Différences par environnement via overlays
Dans chaque overlays/<env>/kustomization.yaml :

Tu peux modifier :

images

replicas

env vars

resources limits

ou appliquer des patchs JSON / YAML sur les manifests base

Exemple :

yaml
Copia
Modifica
bases:
  - ../../base
images:
  - name: frontend-image
    newTag: dev-v1.2.0
👉 Ça te permet d’utiliser le même déploiement de base pour tous les environnements mais avec des variations propres à chacun.

📌 6️⃣ Ce que ça permet :
✅ Déploiement contrôlé par Git
✅ Cloisonnement par environnement
✅ Déploiement automatisé et traçable
✅ Test de branches feature via ApplicationSet possible
✅ Pas de modifications manuelles via kubectl apply en prod
✅ Gestion des rollback et diff faciles dans l’interface ArgoCD

📊 Conclusion
→ Cette structure est exactement celle utilisée dans les boîtes sérieuses qui font du GitOps propre et scalable.
Elle respecte les bonnes pratiques :

DRY : ne pas dupliquer les manifest YAML

CI/CD : intégration automatique de builds

GitOps : déploiement via Git et ArgoCD


5. Stratégie de Déploiement
GitOps Workflow

Développement : Push du code → CI/CD build l'image → Push vers registry
Déploiement : Update du tag d'image dans les manifests K8s → Commit vers repo GitOps
ArgoCD : Détecte les changements → Synchronise automatiquement les clusters
Validation : Health checks et tests automatiques post-déploiement

Environnements

DEV : 1 replica par service, ressources minimales, auto-sync activé
STAGING : 2 replicas, ressources moyennes, sync manuel pour validation
PROD : 5+ replicas, HPA activé, ressources élevées, stratégie de rollback

6. Monitoring et Observabilité
Métriques collectées

Application : Latence, throughput, erreurs (via Prometheus)
Infrastructure : CPU, mémoire, réseau, stockage
Business : Transactions, utilisateurs actifs, revenus

Alerting

Critique : Service down, haute latence, erreurs 5xx
Warning : Usage élevé des ressources, latence modérée
Info : Déploiements, scaling events

7. Sécurité
Mesures implémentées

RBAC : Accès basé sur les rôles pour chaque environnement
Network Policies : Isolation réseau entre namespaces
Secrets Management : Sealed Secrets ou External Secrets Operator
Image Scanning : Vulnérabilité scanning via Trivy
Pod Security Standards : Enforcement des bonnes pratiques

Cette structure permet une gestion robuste et évolutive d'une application multi-tiers en entreprise avec une approche GitOps complète.


Workflow de déploiement typique :

Développeur : Push code → CI build image → Update tag dans manifests
ArgoCD : Détecte changement → Sync cluster → Health check
Monitoring : Alertes automatiques si problème détecté
Rollback : Automatique si les health checks échouent

Cette structure est production-ready et peut gérer des milliers d'utilisateurs avec une équipe de 5-10 développeurs.



# pour deployer sous forme de microservice separe 

explication : comme le montre la figure il ya un project par environement et 3 application qui seront fils de ce project une app pour le front end , une app pour le back end , une app pour la base de donne je dois creer le namespace dev e m'assure que le repos a une branche feature car les app pointes decu

## NB : il manque juste les projects et app des autres environement

dans project il ya le yaml du project je lance juste sa creer un project 

```yaml
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: angular-dev-project
  namespace: argocd
spec:
  description: Projet Angular + Java Spring Boot
  clusterResourceWhitelist:
  - group: ""
    kind: '*'
  destinations:
  - namespace: dev
    server: https://kubernetes.default.svc
  namespaceResourceWhitelist:
  - group: ""
    kind: '*'
  - group: "apps"
    kind: "Deployment"
  sourceRepos:
  - https://github.com/stevymonkam/kubernetes-argocd-angular-javasprintboot.git
  EOF
```

# e puis les applications 



```yaml
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: backend-dev-app
  namespace: argocd
spec:
  project: angular-dev-project
  source:
    repoURL: https://github.com/stevymonkam/kubernetes-argocd-angular-javasprintboot.git
    targetRevision: feature
    path: apps/backend/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
  EOF
```


```yaml
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mysql-dev-app
  namespace: argocd
spec:
  project: angular-dev-project
  source:
    repoURL: https://github.com/stevymonkam/kubernetes-argocd-angular-javasprintboot.git
    targetRevision: feature
    path: apps/mysql/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
      EOF
```


```yaml
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: angular-dev-app
  namespace: argocd
spec:
  project: angular-dev-project
  source:
    repoURL: https://github.com/stevymonkam/kubernetes-argocd-angular-javasprintboot.git
    targetRevision: feature
    path: apps/frontend/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
  EOF
```