# OVH_bis
# Multi-Sites Kubernetes Platform (GitOps)

Cette plateforme permet de déployer automatiquement des sites web dans un cluster Kubernetes (k3s ou autre) via **ArgoCD**, **Helm**, **Ingress Controller**, **cert-manager** et **ExternalDNS**.  

Chaque nouveau site peut être déployé en quelques secondes simplement en ajoutant un `values.yaml` et une Application ArgoCD.  

---

## 🧱 Stack utilisée

| Composant | Rôle |
|-----------|------|
| **ArgoCD** | GitOps : déploiement automatique depuis GitHub |
| **Helm** | Templating des manifests Kubernetes pour tous les sites |
| **Traefik / NGINX Ingress** | Routage HTTP(S) vers les pods |
| **cert-manager** | TLS automatique via Let’s Encrypt |
| **ExternalDNS** | Gestion automatique des enregistrements DNS |

---

## 📂 Structure du repo
```bash 
sites-repo/
  charts/
    website/          # ton chart Helm générique
      templates/
        deployment.yaml
        service.yaml
        ingress.yaml
      values.yaml      # valeurs par défaut
  sites/
    site1/
      values.yaml
    site2/
      values.yaml
```

🔹 Installation des composants du cluster

1️⃣ ArgoCD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
2️⃣ Traefik (Ingress Controller)

K3s installe Traefik par défaut. Sinon :
```bash
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v2.10/docs/content/reference/dynamic-configuration/k8s-crd-definition.yaml
```

3️⃣ cert-manager (TLS automatique)
```bash
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.0 \
  --set installCRDs=true
```
ClusterIssuer Let’s Encrypt :

```yaml 
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ton-email@example.com
    privateKeySecretRef:
      name: letsencrypt-key
    solvers:
      - http01:
          ingress:
            class: traefik
```

4️⃣ ExternalDNS (DNS automatique)

Exemple pour Cloudflare :
```bash
kubectl create namespace external-dns
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install external-dns bitnami/external-dns \
  --namespace external-dns \
  --set provider=cloudflare \
  --set cloudflare.apiToken=CF_API_TOKEN \
  --set txtOwnerId=argo-externaldns \
  --set domainFilters={example.com} \
  --set policy=sync
```


🔹 Workflow de déploiement
	1.	Ajouter un nouveau site dans sites/ avec son values.yaml
	2.	Créer une Application ArgoCD pour ce site ou réutiliser le chart existant avec le nouveau values.yaml
	3.	ArgoCD déploie le chart Helm :
	•	Deployment / Service / Ingress
	4.	Traefik/NGINX expose le site
	5.	cert-manager génère automatiquement le certificat TLS
	6.	ExternalDNS crée le record DNS correspondant
	7.	Le site est en ligne en HTTPS automatiquement 🚀


🔹 Avantages
	•	Nouveau site = quelques lignes de config + Git push
	•	Déploiement 100% automatique grâce à ArgoCD + Helm
	•	DNS + TLS entièrement gérés par le cluster
	•	Plateforme multi-sites clé en main, évolutive



🔹 Ressources utiles
	•	ArgoCD Documentation
	•	Helm Documentation
	•	cert-manager Documentation
	•	ExternalDNS Documentation