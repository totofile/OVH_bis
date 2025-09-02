# 🌐 ExternalDNS - Guide Complet de Gestion DNS Automatique

ExternalDNS synchronise automatiquement les enregistrements DNS avec vos Ingress et Services Kubernetes, éliminant la gestion manuelle des DNS.

## 📋 Table des matières
1. [Installation d'ExternalDNS](#installation-dexternaldns)
2. [Configuration Cloudflare](#configuration-cloudflare)
3. [Configuration OVH](#configuration-ovh)
4. [Configuration Route53 (AWS)](#configuration-route53-aws)
5. [Configuration Google Cloud DNS](#configuration-google-cloud-dns)
6. [Utilisation avec Ingress](#utilisation-avec-ingress)
7. [Gestion des domaines et zones](#gestion-des-domaines-et-zones)
8. [Monitoring et troubleshooting](#monitoring-et-troubleshooting)
9. [Sécurité et bonnes pratiques](#sécurité-et-bonnes-pratiques)

---

## 🔧 Installation d'ExternalDNS

### 1. Installation via Helm (recommandée)

```bash
# Ajouter le repository Helm
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

# Créer le namespace
kubectl create namespace external-dns
```

### 2. Configuration de base

```yaml
# external-dns-values.yaml
# Configuration commune à tous les providers
image:
  tag: "0.14.0"

sources:
  - ingress
  - service

# Politique de gestion des enregistrements
policy: sync  # Options: sync, upsert-only, create-only

# Intervalle de synchronisation
interval: 1m

# Registre pour tracker les enregistrements créés
registry: "txt"
txtOwnerId: "external-dns"

# Logging
logLevel: info
logFormat: text

# Métriques Prometheus
metrics:
  enabled: true
  port: 7979

# Resources
resources:
  limits:
    memory: 256Mi
    cpu: 100m
  requests:
    memory: 128Mi
    cpu: 50m

# Sécurité
securityContext:
  fsGroup: 65534
  runAsNonRoot: true
  runAsUser: 65534
```

---

## ☁️ Configuration Cloudflare

### 1. Créer un token API Cloudflare

1. Allez sur https://dash.cloudflare.com/profile/api-tokens
2. Cliquez sur "Create Token"
3. Utilisez le template "Custom token" avec ces permissions :
   - Zone:Zone:Read
   - Zone:DNS:Edit
4. Limitez aux zones nécessaires
5. Copiez le token généré

### 2. Configuration ExternalDNS pour Cloudflare

```bash
# Créer le secret avec le token API
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=votre-token-cloudflare \
  --namespace external-dns
```

```yaml
# cloudflare-values.yaml
provider: cloudflare

cloudflare:
  # Utiliser le token API (recommandé)
  apiToken: ""
  apiTokenSecretRef:
    name: cloudflare-api-token
    key: api-token
  
  # Ou utiliser email + API key (moins sécurisé)
  # email: votre-email@example.com
  # apiKey: votre-api-key

# Domaines à gérer
domainFilters:
  - "votre-domaine.com"
  - "autre-domaine.com"

# Zones à exclure (optionnel)
excludeDomains:
  - "internal.votre-domaine.com"

# Configuration spécifique Cloudflare
extraArgs:
  - --cloudflare-proxied  # Activer le proxy Cloudflare par défaut
  - --cloudflare-dns-records-per-page=100

# Annotations à surveiller
annotationFilter: external-dns.alpha.kubernetes.io/hostname
```

### 3. Installation avec Helm

```bash
helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --values external-dns-values.yaml \
  --values cloudflare-values.yaml
```

### 4. Exemple d'Ingress avec Cloudflare

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-site-cloudflare
  namespace: default
  annotations:
    # ExternalDNS créera automatiquement l'enregistrement DNS
    external-dns.alpha.kubernetes.io/hostname: "site1.votre-domaine.com"
    
    # Contrôler le proxy Cloudflare par Ingress
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
    
    # TTL personnalisé
    external-dns.alpha.kubernetes.io/ttl: "300"
    
    # cert-manager pour TLS
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    
    # Traefik
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - site1.votre-domaine.com
    secretName: site1-tls
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
```

---

## 🔶 Configuration OVH

### 1. Créer les credentials OVH API

1. Allez sur https://eu.api.ovh.com/createToken/
2. Remplissez les champs :
   - Application name: external-dns
   - Application description: Kubernetes ExternalDNS
3. Droits nécessaires :
   - GET /domain/zone/*
   - POST /domain/zone/*/record
   - DELETE /domain/zone/*/record/*
   - POST /domain/zone/*/refresh
4. Notez : Application Key, Application Secret, Consumer Key

### 2. Configuration ExternalDNS pour OVH

```bash
# Créer le secret avec les credentials OVH
kubectl create secret generic ovh-credentials \
  --from-literal=application-key=votre-app-key \
  --from-literal=application-secret=votre-app-secret \
  --from-literal=consumer-key=votre-consumer-key \
  --namespace external-dns
```

```yaml
# ovh-values.yaml
provider: ovh

ovh:
  endpoint: ovh-eu  # ou ovh-us, ovh-ca selon votre région
  applicationKeySecretRef:
    name: ovh-credentials
    key: application-key
  applicationSecretSecretRef:
    name: ovh-credentials
    key: application-secret
  consumerKeySecretRef:
    name: ovh-credentials
    key: consumer-key

# Domaines à gérer
domainFilters:
  - "votre-domaine.com"

# Configuration spécifique OVH
extraArgs:
  - --ovh-endpoint=ovh-eu
```

### 3. Installation pour OVH

```bash
helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --values external-dns-values.yaml \
  --values ovh-values.yaml
```

---

## 🛡️ Configuration Route53 (AWS)

### 1. Créer un utilisateur IAM pour ExternalDNS

Politique IAM minimale :

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "route53:ChangeResourceRecordSets"
            ],
            "Resource": [
                "arn:aws:route53:::hostedzone/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "route53:ListHostedZones",
                "route53:ListResourceRecordSets",
                "route53:ListTagsForResource"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

### 2. Configuration ExternalDNS pour Route53

```bash
# Créer le secret avec les credentials AWS
kubectl create secret generic aws-credentials \
  --from-literal=access-key-id=VOTRE_ACCESS_KEY_ID \
  --from-literal=secret-access-key=VOTRE_SECRET_ACCESS_KEY \
  --namespace external-dns
```

```yaml
# route53-values.yaml
provider: aws

aws:
  region: eu-west-1
  zoneType: public
  
  # Credentials via secret
  credentials:
    secretName: aws-credentials
    accessKey: access-key-id
    secretKey: secret-access-key
  
  # Ou utiliser IAM roles (recommandé pour EKS)
  # assumeRole: arn:aws:iam::ACCOUNT-ID:role/external-dns

# Zones spécifiques (optionnel)
zoneIdFilters:
  - "Z1D633PJN98FT9"  # Zone ID de votre domaine

# Domaines à gérer
domainFilters:
  - "votre-domaine.aws"

extraArgs:
  - --aws-zone-type=public
  - --aws-prefer-cname
```

---

## 🌐 Configuration Google Cloud DNS

### 1. Créer une clé de service Google Cloud

```bash
# Créer la clé de service
gcloud iam service-accounts create external-dns \
    --display-name "ExternalDNS Service Account"

# Attribuer les permissions
gcloud projects add-iam-policy-binding VOTRE-PROJECT-ID \
    --member serviceAccount:external-dns@VOTRE-PROJECT-ID.iam.gserviceaccount.com \
    --role roles/dns.admin

# Créer et télécharger la clé
gcloud iam service-accounts keys create credentials.json \
    --iam-account external-dns@VOTRE-PROJECT-ID.iam.gserviceaccount.com
```

### 2. Configuration ExternalDNS pour Google Cloud DNS

```bash
# Créer le secret avec la clé de service
kubectl create secret generic google-credentials \
  --from-file=credentials.json=credentials.json \
  --namespace external-dns
```

```yaml
# gcp-values.yaml
provider: google

google:
  project: votre-project-id
  serviceAccountSecretRef:
    name: google-credentials
    key: credentials.json

# Zones à gérer
zoneIdFilters:
  - "votre-zone-dns"

domainFilters:
  - "votre-domaine.gcp"
```

---

## 🔗 Utilisation avec Ingress

### 1. Annotations ExternalDNS importantes

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: exemple-complet
  namespace: default
  annotations:
    # === ExternalDNS ===
    # Hostname principal (obligatoire)
    external-dns.alpha.kubernetes.io/hostname: "app.votre-domaine.com"
    
    # Hostnames multiples
    external-dns.alpha.kubernetes.io/hostname: "app.votre-domaine.com,www.app.votre-domaine.com"
    
    # TTL personnalisé (en secondes)
    external-dns.alpha.kubernetes.io/ttl: "300"
    
    # Type d'enregistrement (A, CNAME, etc.)
    external-dns.alpha.kubernetes.io/type: "A"
    
    # Cible personnalisée (utile pour CNAME)
    external-dns.alpha.kubernetes.io/target: "lb.votre-domaine.com"
    
    # === Provider spécifique ===
    # Cloudflare: activer le proxy
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "true"
    
    # AWS Route53: politique de routage
    external-dns.alpha.kubernetes.io/aws-weight: "100"
    
    # === cert-manager ===
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    
    # === Traefik ===
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls:
  - hosts:
    - app.votre-domaine.com
    - www.app.votre-domaine.com
    secretName: app-tls
  rules:
  - host: app.votre-domaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
  - host: www.app.votre-domaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

### 2. Utilisation avec Services (LoadBalancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-loadbalancer
  namespace: default
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "api.votre-domaine.com"
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: mon-app
```

---

## 🎯 Gestion des domaines et zones

### 1. Configuration multi-domaines

```yaml
# Configuration pour gérer plusieurs domaines
domainFilters:
  - "prod.example.com"
  - "staging.example.com"
  - "dev.example.com"

# Exclure certains sous-domaines
excludeDomains:
  - "internal.prod.example.com"
  - "admin.staging.example.com"

# Zones spécifiques (pour AWS Route53)
zoneIdFilters:
  - "Z1D633PJN98FT9"  # Zone prod
  - "Z2E744QWX87ABC"  # Zone staging
```

### 2. Stratégies de déploiement par environnement

```yaml
# Production - politique stricte
policy: upsert-only  # Ne supprime jamais d'enregistrements
txtOwnerId: "external-dns-prod"

# Staging - politique flexible
policy: sync  # Peut créer et supprimer
txtOwnerId: "external-dns-staging"

# Development - politique permissive
policy: sync
txtOwnerId: "external-dns-dev"
```

### 3. Template Helm pour vos sites avec DNS automatique

```yaml
# charts/website/templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "website.fullname" . }}
  namespace: {{ .Values.namespace | default .Release.Namespace }}
  annotations:
    # ExternalDNS
    external-dns.alpha.kubernetes.io/hostname: {{ .Values.ingress.host }}
    {{- if .Values.ingress.ttl }}
    external-dns.alpha.kubernetes.io/ttl: "{{ .Values.ingress.ttl }}"
    {{- end }}
    {{- if .Values.ingress.cloudflareProxied }}
    external-dns.alpha.kubernetes.io/cloudflare-proxied: "{{ .Values.ingress.cloudflareProxied }}"
    {{- end }}
    
    # cert-manager
    cert-manager.io/cluster-issuer: {{ .Values.ingress.clusterIssuer | default "letsencrypt-prod" }}
    
    # Traefik
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

---

## 📊 Monitoring et troubleshooting

### 1. Commandes de diagnostic

```bash
# Vérifier l'état d'ExternalDNS
kubectl get pods -n external-dns
kubectl logs -n external-dns deployment/external-dns -f

# Voir les enregistrements DNS gérés
kubectl get ingress --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTS:.spec.rules[*].host

# Vérifier les annotations ExternalDNS
kubectl get ingress mon-ingress -o yaml | grep external-dns

# Événements liés à ExternalDNS
kubectl get events --field-selector involvedObject.name=external-dns -n external-dns
```

### 2. Debug des problèmes DNS

```bash
# Vérifier si l'enregistrement DNS existe
nslookup site1.votre-domaine.com

# Vérifier les enregistrements TXT de suivi
nslookup -type=TXT external-dns-site1.votre-domaine.com

# Forcer une synchronisation
kubectl rollout restart deployment/external-dns -n external-dns

# Mode debug
kubectl set env deployment/external-dns -n external-dns LOG_LEVEL=debug
```

### 3. Métriques et alertes

```yaml
# Alertes Prometheus pour ExternalDNS
groups:
- name: external-dns
  rules:
  - alert: ExternalDNSDown
    expr: up{job="external-dns"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "ExternalDNS is down"
      description: "ExternalDNS has been down for more than 5 minutes"

  - alert: ExternalDNSErrorRate
    expr: rate(external_dns_registry_errors_total[5m]) > 0.1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High error rate in ExternalDNS"
      description: "ExternalDNS error rate is {{ $value }} errors per second"

  - alert: ExternalDNSRecordLag
    expr: time() - external_dns_last_registry_sync_timestamp > 600
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "ExternalDNS sync lag"
      description: "ExternalDNS hasn't synced for more than 10 minutes"
```

---

## 🔒 Sécurité et bonnes pratiques

### 1. Principe du moindre privilège

```yaml
# Limiter les domaines gérés
domainFilters:
  - "apps.votre-domaine.com"  # Seulement le sous-domaine apps

# Exclure les domaines sensibles
excludeDomains:
  - "admin.votre-domaine.com"
  - "api.votre-domaine.com"

# Politique restrictive
policy: upsert-only  # Ne peut pas supprimer d'enregistrements existants
```

### 2. Séparation par environnements

```bash
# ExternalDNS pour production
helm install external-dns-prod external-dns/external-dns \
  --namespace external-dns-prod \
  --set txtOwnerId=external-dns-prod \
  --set domainFilters="{prod.votre-domaine.com}" \
  --set policy=upsert-only

# ExternalDNS pour staging
helm install external-dns-staging external-dns/external-dns \
  --namespace external-dns-staging \
  --set txtOwnerId=external-dns-staging \
  --set domainFilters="{staging.votre-domaine.com}" \
  --set policy=sync
```

### 3. Rotation des credentials

```bash
# Script de rotation des tokens Cloudflare
#!/bin/bash

# Générer un nouveau token
NEW_TOKEN="nouveau-token-cloudflare"

# Mettre à jour le secret
kubectl patch secret cloudflare-api-token \
  -n external-dns \
  --type='json' \
  -p='[{"op": "replace", "path": "/data/api-token", "value": "'$(echo -n $NEW_TOKEN | base64)'"}]'

# Redémarrer ExternalDNS pour prendre en compte le nouveau token
kubectl rollout restart deployment/external-dns -n external-dns
```

### 4. Validation et tests

```bash
# Test de création d'enregistrement DNS
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-dns
  namespace: default
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "test.votre-domaine.com"
    external-dns.alpha.kubernetes.io/ttl: "60"
spec:
  ingressClassName: traefik
  rules:
  - host: test.votre-domaine.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
EOF

# Attendre quelques minutes puis vérifier
nslookup test.votre-domaine.com

# Nettoyer le test
kubectl delete ingress test-dns
```

---

## 🎯 Configuration complète pour votre plateforme multi-sites

### Exemple de values.yaml pour un site avec DNS automatique

```yaml
# sites/site1/values.yaml
ingress:
  host: site1.votre-domaine.com
  className: traefik
  clusterIssuer: letsencrypt-prod
  
  # ExternalDNS
  ttl: 300
  cloudflareProxied: true
  
  annotations:
    # Middleware Traefik personnalisé
    traefik.ingress.kubernetes.io/router.middlewares: default-compress@kubernetescrd,default-security-headers@kubernetescrd

image:
  repository: nginx
  tag: "1.21"

service:
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi
```

Cette configuration vous permet d'avoir une gestion DNS entièrement automatisée pour tous vos sites ! 🌐✨

Le workflow complet sera :
1. **Vous créez** un nouveau site avec son `values.yaml`
2. **ArgoCD déploie** automatiquement le site
3. **ExternalDNS crée** l'enregistrement DNS
4. **cert-manager génère** le certificat TLS
5. **Le site est accessible** en HTTPS automatiquement !

Plus besoin de gérer manuellement les DNS ! 🚀 