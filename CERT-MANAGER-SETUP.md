# 🔒 cert-manager - Guide Complet TLS Automatique avec Let's Encrypt

cert-manager automatise la gestion des certificats TLS dans Kubernetes, permettant d'avoir HTTPS automatiquement sur tous vos sites.

## 📋 Table des matières
1. [Installation de cert-manager](#installation-de-cert-manager)
2. [Configuration des ClusterIssuers](#configuration-des-clusterissuers)
3. [Validation HTTP-01](#validation-http-01)
4. [Validation DNS-01](#validation-dns-01)
5. [Utilisation avec Ingress](#utilisation-avec-ingress)
6. [Certificats wildcards](#certificats-wildcards)
7. [Monitoring et troubleshooting](#monitoring-et-troubleshooting)
8. [Renouvellement automatique](#renouvellement-automatique)

---

## 🔧 Installation de cert-manager

### 1. Installation via Helm (recommandée)

```bash
# Ajouter le repository Helm
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Créer le namespace
kubectl create namespace cert-manager

# Installer cert-manager avec les CRDs
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.0 \
  --set installCRDs=true \
  --set prometheus.enabled=true \
  --set webhook.timeoutSeconds=4
```

### 2. Installation via manifests YAML

```bash
# Installer les CRDs
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.crds.yaml

# Créer le namespace
kubectl create namespace cert-manager

# Installer cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
```

### 3. Vérifier l'installation

```bash
# Vérifier que tous les pods sont en cours d'exécution
kubectl get pods --namespace cert-manager

# Vérifier les CRDs
kubectl get crd | grep cert-manager

# Test de l'installation
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF

# Vérifier que le certificat est créé
kubectl describe certificate -n cert-manager-test

# Nettoyer le test
kubectl delete namespace cert-manager-test
```

---

## 🌐 Configuration des ClusterIssuers

### 1. ClusterIssuer Let's Encrypt Production

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # URL du serveur ACME Let's Encrypt (production)
    server: https://acme-v02.api.letsencrypt.org/directory
    
    # Email pour les notifications importantes
    email: votre-email@example.com
    
    # Secret pour stocker la clé privée ACME
    privateKeySecretRef:
      name: letsencrypt-prod
    
    # Solveurs pour la validation des défis
    solvers:
    # Validation HTTP-01 via Traefik
    - http01:
        ingress:
          class: traefik
          podTemplate:
            spec:
              nodeSelector:
                "kubernetes.io/os": linux
    
    # Validation DNS-01 pour les wildcards (exemple Cloudflare)
    - dns01:
        cloudflare:
          email: votre-email@example.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
      selector:
        dnsNames:
        - "*.votre-domaine.com"
        - "votre-domaine.com"
```

### 2. ClusterIssuer Let's Encrypt Staging (pour les tests)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # URL du serveur ACME Let's Encrypt (staging - limites plus élevées)
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    
    email: votre-email@example.com
    
    privateKeySecretRef:
      name: letsencrypt-staging
    
    solvers:
    - http01:
        ingress:
          class: traefik
```

### 3. Appliquer les ClusterIssuers

```bash
# Appliquer les configurations
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
EOF

# Vérifier l'état
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```

---

## 🌍 Validation HTTP-01

La validation HTTP-01 est la méthode la plus simple pour les certificats de domaines individuels.

### 1. Configuration pour Traefik

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-http01
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-http01
    solvers:
    - http01:
        ingress:
          class: traefik
          # Configuration spécifique à Traefik
          podTemplate:
            metadata:
              annotations:
                traefik.ingress.kubernetes.io/router.entrypoints: web
            spec:
              containers:
              - name: acme-challenge-solver
                resources:
                  requests:
                    cpu: 10m
                    memory: 64Mi
                  limits:
                    cpu: 100m
                    memory: 128Mi
```

### 2. Configuration pour NGINX Ingress

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-nginx
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-nginx
    solvers:
    - http01:
        ingress:
          class: nginx
          podTemplate:
            spec:
              nodeSelector:
                "kubernetes.io/os": linux
```

### 3. Exemple d'Ingress avec certificat automatique

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-site-ingress
  namespace: default
  annotations:
    # Spécifier le ClusterIssuer à utiliser
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    
    # Annotations spécifiques à Traefik
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    
    # Redirection HTTP -> HTTPS
    traefik.ingress.kubernetes.io/redirect-to-https: "true"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - mon-site.votre-domaine.com
    # Le secret sera créé automatiquement par cert-manager
    secretName: mon-site-tls
  rules:
  - host: mon-site.votre-domaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mon-site-service
            port:
              number: 80
```

---

## 🔑 Validation DNS-01

La validation DNS-01 est nécessaire pour les certificats wildcards et fonctionne même derrière des firewalls.

### 1. Configuration Cloudflare

```bash
# Créer le secret avec le token API Cloudflare
kubectl create secret generic cloudflare-api-token-secret \
  --from-literal=api-token=votre-token-cloudflare \
  --namespace cert-manager
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01-cloudflare
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-dns01-cloudflare
    solvers:
    - dns01:
        cloudflare:
          email: votre-email@example.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
      selector:
        dnsNames:
        - "*.votre-domaine.com"
        - "votre-domaine.com"
```

### 2. Configuration Route53 (AWS)

```bash
# Créer le secret avec les credentials AWS
kubectl create secret generic route53-credentials-secret \
  --from-literal=secret-access-key=VOTRE_SECRET_KEY \
  --namespace cert-manager
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01-route53
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-dns01-route53
    solvers:
    - dns01:
        route53:
          region: us-east-1
          accessKeyID: VOTRE_ACCESS_KEY_ID
          secretAccessKeySecretRef:
            name: route53-credentials-secret
            key: secret-access-key
      selector:
        dnsNames:
        - "*.votre-domaine.com"
        - "votre-domaine.com"
```

### 3. Configuration OVH

```bash
# Créer les secrets pour OVH API
kubectl create secret generic ovh-credentials \
  --from-literal=applicationKey=VOTRE_APP_KEY \
  --from-literal=applicationSecret=VOTRE_APP_SECRET \
  --from-literal=consumerKey=VOTRE_CONSUMER_KEY \
  --namespace cert-manager
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns01-ovh
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-dns01-ovh
    solvers:
    - dns01:
        webhook:
          groupName: acme.bwolf.me
          solverName: ovh
          config:
            endpoint: ovh-eu
            applicationKeySecretRef:
              name: ovh-credentials
              key: applicationKey
            applicationSecretSecretRef:
              name: ovh-credentials
              key: applicationSecret
            consumerKeySecretRef:
              name: ovh-credentials
              key: consumerKey
```

---

## 🌟 Certificats Wildcards

### 1. Certificat wildcard avec DNS-01

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-certificate
  namespace: default
spec:
  secretName: wildcard-tls-secret
  issuerRef:
    name: letsencrypt-dns01-cloudflare
    kind: ClusterIssuer
  dnsNames:
  - "*.votre-domaine.com"
  - "votre-domaine.com"
```

### 2. Utiliser le certificat wildcard dans un Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: site1-ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - site1.votre-domaine.com
    # Utiliser le certificat wildcard existant
    secretName: wildcard-tls-secret
  rules:
  - host: site1.votre-domaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: site1-service
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: site2-ingress
  namespace: default
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - site2.votre-domaine.com
    # Même certificat wildcard pour tous les sous-domaines
    secretName: wildcard-tls-secret
  rules:
  - host: site2.votre-domaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: site2-service
            port:
              number: 80
```

---

## 📊 Monitoring et troubleshooting

### 1. Commandes de diagnostic

```bash
# Lister tous les certificats
kubectl get certificates --all-namespaces

# Détails d'un certificat
kubectl describe certificate mon-certificat -n default

# Voir les événements cert-manager
kubectl get events --field-selector involvedObject.kind=Certificate

# Logs de cert-manager
kubectl logs -n cert-manager deployment/cert-manager -f

# Vérifier les CertificateRequests
kubectl get certificaterequests --all-namespaces

# Vérifier les Orders (défis ACME)
kubectl get orders --all-namespaces

# Vérifier les Challenges
kubectl get challenges --all-namespaces
```

### 2. Debug d'un certificat qui ne se génère pas

```bash
# Vérifier l'état du certificat
kubectl describe certificate mon-certificat -n default

# Vérifier les CertificateRequests associés
kubectl get certificaterequests -n default
kubectl describe certificaterequest <nom-du-request> -n default

# Vérifier les Orders ACME
kubectl get orders -n default
kubectl describe order <nom-de-l-order> -n default

# Vérifier les Challenges
kubectl get challenges -n default
kubectl describe challenge <nom-du-challenge> -n default

# Forcer le renouvellement d'un certificat
kubectl annotate certificate mon-certificat cert-manager.io/force-renewal=$(date +%s)
```

### 3. Résolution des problèmes courants

```bash
# Problème 1: Certificat en état "False"
# Solution: Vérifier les logs et les événements
kubectl describe certificate mon-certificat
kubectl logs -n cert-manager deployment/cert-manager --tail=100

# Problème 2: Challenge HTTP-01 qui échoue
# Solution: Vérifier que l'Ingress Controller fonctionne
kubectl get pods -n kube-system | grep traefik
kubectl logs -n kube-system deployment/traefik

# Problème 3: Challenge DNS-01 qui échoue
# Solution: Vérifier les credentials DNS
kubectl get secrets -n cert-manager | grep dns
kubectl describe secret cloudflare-api-token-secret -n cert-manager

# Problème 4: Certificat expiré
# Solution: Forcer le renouvellement
kubectl delete certificaterequest --all -n default
kubectl annotate certificate mon-certificat cert-manager.io/force-renewal=$(date +%s)
```

---

## 🔄 Renouvellement automatique

### 1. Configuration du renouvellement

cert-manager renouvelle automatiquement les certificats 30 jours avant leur expiration. Configuration par défaut :

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-certificat
spec:
  secretName: mon-certificat-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - mon-site.votre-domaine.com
  # Renouveler 30 jours avant expiration (par défaut)
  renewBefore: 720h # 30 jours
  # Durée de vie du certificat (Let's Encrypt: 90 jours)
  duration: 2160h # 90 jours
```

### 2. Monitoring du renouvellement

```bash
# Voir les certificats et leur date d'expiration
kubectl get certificates -o wide --all-namespaces

# Alertes Prometheus pour les certificats qui expirent
cat > cert-manager-alerts.yaml << EOF
groups:
- name: cert-manager
  rules:
  - alert: CertManagerCertExpiringSoon
    expr: certmanager_certificate_expiration_timestamp_seconds - time() < 604800
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "Certificate expiring soon"
      description: "Certificate {{ \$labels.name }} in namespace {{ \$labels.namespace }} expires in less than 7 days"

  - alert: CertManagerCertNotReady
    expr: certmanager_certificate_ready_status == 0
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "Certificate not ready"
      description: "Certificate {{ \$labels.name }} in namespace {{ \$labels.namespace }} is not ready"
EOF
```

### 3. Script de vérification automatique

```bash
#!/bin/bash
# check-certificates.sh

echo "=== État des certificats ==="
kubectl get certificates --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,READY:.status.conditions[?(@.type==\"Ready\")].status,SECRET:.spec.secretName,AGE:.metadata.creationTimestamp

echo -e "\n=== Certificats qui expirent bientôt ==="
kubectl get certificates --all-namespaces -o json | jq -r '
.items[] | 
select(.status.notAfter != null) | 
select((now + 604800) > (.status.notAfter | fromdateiso8601)) |
"\(.metadata.namespace)/\(.metadata.name) expires: \(.status.notAfter)"'

echo -e "\n=== Certificats en erreur ==="
kubectl get certificates --all-namespaces -o json | jq -r '
.items[] | 
select(.status.conditions != null) |
select(.status.conditions[] | select(.type == "Ready" and .status == "False")) |
"\(.metadata.namespace)/\(.metadata.name): \(.status.conditions[] | select(.type == "Ready").message)"'
```

---

## 🎯 Configuration complète pour votre plateforme multi-sites

### Template Helm pour vos sites avec certificats automatiques

```yaml
# charts/website/templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "website.fullname" . }}
  namespace: {{ .Values.namespace | default .Release.Namespace }}
  annotations:
    cert-manager.io/cluster-issuer: {{ .Values.ingress.clusterIssuer | default "letsencrypt-prod" }}
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/redirect-to-https: "true"
    {{- with .Values.ingress.annotations }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className | default "traefik" }}
  tls:
  - hosts:
    - {{ .Values.ingress.host }}
    secretName: {{ include "website.fullname" . }}-tls
  rules:
  - host: {{ .Values.ingress.host }}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: {{ include "website.fullname" . }}
            port:
              number: {{ .Values.service.port }}
```

### Exemple de values.yaml pour un site

```yaml
# sites/site1/values.yaml
ingress:
  host: site1.votre-domaine.com
  clusterIssuer: letsencrypt-prod
  className: traefik
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: default-compress@kubernetescrd

image:
  repository: nginx
  tag: "1.21"

service:
  port: 80
```

Cette configuration vous donnera des certificats TLS automatiques pour tous vos sites ! 🔒✨ 