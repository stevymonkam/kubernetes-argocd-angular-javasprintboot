# Restrictions ArgoCD - AppProjects vs Applications

## 1. Restrictions au niveau AppProject (Niveau Équipe/Organisation)

### 1.1 Restrictions de Repositories Source
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: app-team-alpha-project
spec:
  # RESTRICTION: Quels repositories peuvent être utilisés
  sourceRepos:
  - 'https://github.com/company/gitops-repo'
  - 'https://helm-charts.company.com/*'
  # Bloque l'utilisation de repositories non autorisés
  # Les Applications de ce projet ne pourront utiliser QUE ces repos
```

### 1.2 Restrictions de Destinations (Clusters/Namespaces)
```yaml
spec:
  # RESTRICTION: Où peuvent être déployées les applications
  destinations:
  - namespace: 'alpha-*'              # Wildcard: tous les namespaces alpha-*
    server: https://kubernetes.default.svc
  - namespace: 'alpha-dev'            # Namespace spécifique
    server: https://kubernetes.default.svc
  - namespace: 'shared-services'      # Namespace partagé autorisé
    server: https://prod-cluster.company.com
  
  # BLOQUE: Déploiement dans kube-system, default, ou autres namespaces sensibles
  # BLOQUE: Déploiement sur d'autres clusters non autorisés
```

### 1.3 Restrictions de Ressources Kubernetes
```yaml
spec:
  # RESTRICTION: Ressources Cluster-level autorisées (dangereuses)
  clusterResourceWhitelist:
  - group: 'networking.k8s.io'
    kind: 'NetworkPolicy'
  # BLOQUE: ClusterRole, ClusterRoleBinding, PersistentVolume, etc.
  
  # RESTRICTION: Ressources Namespace-level autorisées
  namespaceResourceWhitelist:
  - group: 'apps'
    kind: 'Deployment'
  - group: 'apps' 
    kind: 'ReplicaSet'
  - group: ''
    kind: 'Service'
  - group: ''
    kind: 'ConfigMap'
  - group: ''
    kind: 'Secret'
  - group: 'networking.k8s.io'
    kind: 'Ingress'
  # BLOQUE: DaemonSet, Job, CronJob si non listés
  
  # RESTRICTION: Ressources complètement interdites
  namespaceResourceBlacklist:
  - group: 'policy'
    kind: 'PodSecurityPolicy'
  - group: 'extensions'
    kind: 'PodSecurityPolicy'
```

### 1.4 Restrictions de Synchronisation
```yaml
spec:
  # RESTRICTION: Contrôle des stratégies de sync
  syncWindows:
  - kind: 'deny'
    schedule: '0 2 * * *'  # Interdit sync entre 2h-3h (maintenance)
    duration: 1h
    applications:
    - '*-prod'
  - kind: 'allow'
    schedule: '0 9-17 * * 1-5'  # Autorise sync heures bureau
    duration: 8h
    applications:
    - '*'
```

### 1.5 Restrictions RBAC (Rôles)
```yaml
spec:
  roles:
  - name: dev-role
    policies:
    - p, proj:app-team-alpha-project:dev-role, applications, get, app-team-alpha-project/*, allow
    - p, proj:app-team-alpha-project:dev-role, applications, sync, app-team-alpha-project/*-dev, allow
    - p, proj:app-team-alpha-project:dev-role, applications, sync, app-team-alpha-project/*-prod, deny
    # RESTRICTION: Les devs peuvent sync dev mais pas prod
    groups:
    - alpha-developers
```

## 2. Restrictions au niveau Application (Niveau Application Individuelle)

### 2.1 Restrictions de Source
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-frontend-prod
spec:
  # RESTRICTION: Source spécifique (doit respecter AppProject)
  source:
    repoURL: https://github.com/company/gitops-repo  # Doit être dans sourceRepos du projet
    targetRevision: HEAD                             # Branche/tag/commit spécifique
    path: applications/team-alpha/ecommerce-frontend/overlays/prod  # Chemin exact
    
    # RESTRICTION: Paramètres Helm (si applicable)
    helm:
      valueFiles:
      - values.yaml
      - values-prod.yaml
      parameters:
      - name: "image.tag"
        value: "v1.2.3"
      # BLOQUE: Autres value files ou paramètres non définis
```

### 2.2 Restrictions de Destination
```yaml
spec:
  # RESTRICTION: Destination unique et spécifique
  destination:
    server: https://kubernetes.default.svc    # Doit être autorisé dans AppProject
    namespace: alpha-prod                     # Doit matcher les destinations du projet
  # BLOQUE: Déploiement simultané multi-namespace ou multi-cluster
```

### 2.3 Restrictions de Synchronisation
```yaml
spec:
  syncPolicy:
    # RESTRICTION: Stratégie de sync automatique
    automated:
      prune: true          # PEUT supprimer ressources orphelines
      selfHeal: true       # PEUT corriger la dérive automatiquement
    
    # RESTRICTION: Options de sync spécifiques
    syncOptions:
    - CreateNamespace=true     # PEUT créer le namespace
    - PrunePropagationPolicy=foreground
    - PruneLast=true          # Supprime les ressources en dernier
    - Validate=false          # PEUT ignorer validation kubectl
    - ServerSideApply=true    # Utilise server-side apply
    - RespectIgnoreDifferences=true
    
    # RESTRICTION: Retry en cas d'échec
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 2.4 Restrictions de Différences Ignorées
```yaml
spec:
  # RESTRICTION: Ignore certains champs lors de la comparaison
  ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
    - /spec/replicas           # Ignore les changements de replicas (HPA)
  - group: ""
    kind: Secret
    name: oauth-token
    jsonPointers:
    - /data/token              # Ignore les changements de token
```

## 3. Comparaison des Restrictions

| Type de Restriction | AppProject (Niveau Équipe) | Application (Niveau App) |
|---------------------|---------------------------|-------------------------|
| **Repositories** | ✅ Liste globale autorisée | ✅ Repo spécifique (sous-ensemble) |
| **Destinations** | ✅ Clusters/Namespaces autorisés | ✅ Destination unique |
| **Ressources K8s** | ✅ Types autorisés/interdits | ❌ Hérite du projet |
| **RBAC** | ✅ Qui peut faire quoi | ❌ Hérite du projet |
| **Sync Windows** | ✅ Fenêtres autorisées/interdites | ❌ Hérite du projet |
| **Source Path** | ❌ Non applicable | ✅ Chemin spécifique |
| **Sync Policy** | ❌ Non applicable | ✅ Comportement sync |
| **Ignore Differences** | ❌ Non applicable | ✅ Champs ignorés |

## 4. Exemples Pratiques de Violations

### 4.1 Violations AppProject
```yaml
# ❌ VIOLATION: Repository non autorisé
apiVersion: argoproj.io/v1alpha1
kind: Application
spec:
  source:
    repoURL: https://github.com/hacker/malicious-repo  # Pas dans sourceRepos
    
# ❌ VIOLATION: Namespace non autorisé  
spec:
  destination:
    namespace: kube-system  # Pas dans destinations autorisées
    
# ❌ VIOLATION: Ressource interdite
# Dans le manifest K8s déployé:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole  # Pas dans clusterResourceWhitelist
```

### 4.2 Exemples de Restrictions Effectives

#### Projet DevOps (Permissif)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: infra-project
spec:
  sourceRepos: ['*']  # Tous repositories
  destinations:
  - server: '*'       # Tous clusters
    namespace: '*'    # Tous namespaces
  clusterResourceWhitelist:
  - group: '*'        # Toutes ressources cluster
    kind: '*'
  # = AUCUNE RESTRICTION pour DevOps
```

#### Projet Équipe Dev (Restrictif)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: app-team-alpha-project
spec:
  sourceRepos:
  - 'https://github.com/company/gitops-repo'  # SEUL repo autorisé
  
  destinations:
  - namespace: 'alpha-*'                      # SEULS namespaces alpha-*
    server: https://kubernetes.default.svc    # SEUL cluster autorisé
    
  clusterResourceWhitelist: []                # AUCUNE ressource cluster
  
  namespaceResourceWhitelist:
  - group: 'apps'                            # SEULEMENT apps
    kind: 'Deployment'
  - group: ''                                # SEULEMENT services de base
    kind: 'Service'
  # = RESTRICTIONS STRICTES pour développeurs
```

## 5. Ordre de Priorité des Restrictions

1. **AppProject** définit les limites globales (ce qui est POSSIBLE)
2. **Application** spécifie le comportement dans ces limites (ce qui est FAIT)
3. **RBAC ArgoCD** contrôle qui peut créer/modifier (QUI peut faire QUOI)

### Exemple de Cascade
```
AppProject autorise:
├── Repos: github.com/company/*
├── Namespaces: alpha-*
└── Ressources: Deployment, Service, ConfigMap

Application spécifie:
├── Repo: github.com/company/gitops-repo      ✅ OK (dans la liste)
├── Namespace: alpha-prod                     ✅ OK (match alpha-*)  
├── Resources déployées: Deployment + Service ✅ OK (dans la whitelist)
└── Sync: automatique avec prune             ✅ OK (comportement app)

RBAC vérifie:
└── Utilisateur 'dev-john' peut-il sync cette app? ✅ Vérifie les policies
```

Cette architecture en couches assure une sécurité défense en profondeur !