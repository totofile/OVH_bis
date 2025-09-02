# 🚀 ArgoCD - Guide Complet d'Installation et Configuration

ArgoCD est l'outil de GitOps qui va automatiser le déploiement de vos applications depuis Git vers Kubernetes.

## 📋 Table des matières
1. [Installation d'ArgoCD](#installation-dargocd)
2. [Configuration initiale](#configuration-initiale)
3. [Accès à l'interface web](#accès-à-linterface-web)
4. [Configuration des repositories](#configuration-des-repositories)
5. [Création d'applications](#création-dapplications)
6. [Configuration avancée](#configuration-avancée)
7. [Monitoring et troubleshooting](#monitoring-et-troubleshooting)

---

## 🔧 Installation d'ArgoCD

### 1. Installation standard

```bash
# Créer le namespace
kubectl create namespace argocd

# Installer ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Vérifier l'installation
kubectl get pods -n argocd
```

### 2. Installation via Helm (recommandée pour la production)

```bash
# Ajouter le repo Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Créer un fichier values personnalisé
cat > argocd-values.yaml << EOF
server:
  service:
    type: LoadBalancer
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - argocd.votre-domaine.com
    tls:
      - secretName: argocd-server-tls
        hosts:
          - argocd.votre-domaine.com
  config:
    url: https://argocd.votre-domaine.com
    application.instanceLabelKey: argocd.argoproj.io/instance

configs:
  secret:
    # Générer avec : openssl rand -base64 32
    argocdServerAdminPassword: '$2a$10$rRyBsGSHK6.uc8fntPwVIuLVHgsAhAX7TcdrqW/RADU0ufHSdGY2y'
  
dex:
  enabled: false

redis-ha:
  enabled: true

controller:
  replicas: 1

repoServer:
  replicas: 2

applicationSet:
  enabled: true
EOF

# Installer avec Helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values argocd-values.yaml
```

---

## ⚙️ Configuration initiale

### 1. Récupérer le mot de passe admin

```bash
# Mot de passe par défaut (username: admin)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 2. Installer ArgoCD CLI

```bash
# Linux
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Se connecter via CLI
argocd login argocd.votre-domaine.com
# ou via port-forward
kubectl port-forward svc/argocd-server -n argocd 8080:443
argocd login localhost:8080
```

### 3. Changer le mot de passe admin

```bash
argocd account update-password
```

---

## 🌐 Accès à l'interface web

### 1. Via Ingress (recommandé)

Créer un Ingress pour ArgoCD :

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    # Important pour ArgoCD
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - argocd.votre-domaine.com
    secretName: argocd-server-tls
  rules:
  - host: argocd.votre-domaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
```

### 2. Configuration ArgoCD pour Ingress

```bash
# Configurer ArgoCD pour accepter les connexions via Ingress
kubectl patch configmap argocd-cmd-params-cm -n argocd --patch='{"data":{"server.insecure":"true"}}'
kubectl rollout restart deployment argocd-server -n argocd
```

---

## 📂 Configuration des repositories

### 1. Ajouter un repository Git via CLI

```bash
# Repository public
argocd repo add https://github.com/votre-username/votre-repo.git

# Repository privé avec SSH
argocd repo add git@github.com:votre-username/votre-repo-prive.git \
  --ssh-private-key-path ~/.ssh/id_rsa

# Repository privé avec token
argocd repo add https://github.com/votre-username/votre-repo-prive.git \
  --username votre-username \
  --password ghp_votre_token_github
```

### 2. Ajouter un repository via YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/votre-username/votre-repo-prive.git
  username: votre-username
  password: ghp_votre_token_github
```

---

## 🚀 Création d'applications

### 1. Application simple via CLI

```bash
argocd app create mon-site-web \
  --repo https://github.com/votre-username/sites-repo.git \
  --path charts/website \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --values-literal-file sites/site1/values.yaml \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### 2. Application via YAML (recommandé)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: site1-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/votre-username/sites-repo.git
    targetRevision: main
    path: charts/website
    helm:
      valueFiles:
        - ../../sites/site1/values.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"
        - name: ingress.host
          value: "site1.votre-domaine.com"
  destination:
    server: https://kubernetes.default.svc
    namespace: site1
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 3. ApplicationSet pour gérer plusieurs sites

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: sites-generator
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/votre-username/sites-repo.git
      revision: main
      directories:
      - path: sites/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/votre-username/sites-repo.git
        targetRevision: main
        path: charts/website
        helm:
          valueFiles:
            - ../../{{path}}/values.yaml
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

---

## 🔧 Configuration avancée

### 1. Configuration RBAC

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Admins complets
    g, argocd-admins, role:admin
    
    # Développeurs (lecture seule + sync)
    g, developers, role:developer
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    p, role:developer, repositories, get, *, allow
    
    # Ops (tout sauf suppression)
    g, ops-team, role:ops
    p, role:ops, applications, *, */*, allow
    p, role:ops, applications, delete, */*, deny
```

### 2. Configuration des notifications

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: xoxb-votre-slack-token
  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} is now running new version.
  template.app-health-degraded: |
    message: |
      Application {{.app.metadata.name}} has degraded health status.
  trigger.on-deployed: |
    - when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
      send: [app-deployed]
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]
```

### 3. Configuration des hooks et waves

Dans vos manifests Kubernetes :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: database-migration
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-weight: "-5"
    argocd.argoproj.io/sync-wave: "-1"
spec:
  template:
    spec:
      containers:
      - name: migrate
        image: migrate/migrate
        command: ["migrate", "-path", "/migrations", "-database", "postgres://...", "up"]
      restartPolicy: Never
```

---

## 📊 Monitoring et troubleshooting

### 1. Vérifier l'état des applications

```bash
# Lister toutes les applications
argocd app list

# Détails d'une application
argocd app get mon-site-web

# Logs de synchronisation
argocd app logs mon-site-web

# Historique des déploiements
argocd app history mon-site-web
```

### 2. Debugging courant

```bash
# Forcer une synchronisation
argocd app sync mon-site-web

# Supprimer une application (avec ressources)
argocd app delete mon-site-web --cascade

# Refresh du repository
argocd app get mon-site-web --refresh

# Voir les différences
argocd app diff mon-site-web
```

### 3. Métriques et alertes

ArgoCD expose des métriques Prometheus sur `:8082/metrics`. Alertes importantes :

```yaml
# Alerte si une app est OutOfSync trop longtemps
- alert: ArgoCDAppOutOfSync
  expr: argocd_app_info{sync_status="OutOfSync"} > 0
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "ArgoCD application {{ $labels.name }} is out of sync"

# Alerte si une app est Unhealthy
- alert: ArgoCDAppUnhealthy
  expr: argocd_app_info{health_status!="Healthy"} > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "ArgoCD application {{ $labels.name }} is unhealthy"
```

---

## 🎯 Bonnes pratiques pour votre plateforme multi-sites

1. **Utilisez ApplicationSet** pour gérer automatiquement plusieurs sites
2. **Structurez vos repos** avec des environnements séparés (dev/staging/prod)
3. **Implémentez des hooks** pour les migrations de base de données
4. **Configurez les notifications** pour être alerté des déploiements
5. **Utilisez des waves** pour contrôler l'ordre de déploiement
6. **Activez l'auto-sync avec self-heal** pour une vraie approche GitOps

Cette configuration ArgoCD vous permettra de déployer automatiquement vos sites dès qu'ils sont poussés dans Git ! 🚀 