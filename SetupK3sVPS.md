## Passo a Passo Completo - Setup K3s + Portainer + Traefik + SSL

### 1. **Preparar sistema**
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. **Instalar K3s**
```bash
curl -sfL https://get.k3s.io | sh -
```

### 3. **Configurar kubectl**
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
```

### 4. **Instalar Helm**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 5. **Instalar Portainer**
```bash
helm repo add portainer https://portainer.github.io/k8s/
helm repo update
helm install portainer portainer/portainer --namespace portainer --create-namespace \
  --set service.type=NodePort --set tls.force=false
```

### 6. **Instalar Cert-Manager**
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.2/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.13.2
```

### 7. **Configurar Let's Encrypt**
```bash
nano letsencrypt-prod.yaml
```
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: SEU_EMAIL@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
```
```bash
kubectl apply -f letsencrypt-prod.yaml
```
### 7. **Opcional: ClusterIssuer de Staging (para testes)**
```bash
nano letsencrypt-staging.yaml
```
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: SEU_EMAIL@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: traefik
```
```bash
kubectl apply -f letsencrypt-staging.yaml
```

### 8. **Configurar DNS no Cloudflare**
- Tipo A: `deltasofth.cloud` → `164.92.97.238` (proxy desativado ☁️)
- CNAME: `www` → `deltasofth.cloud`

### 9. **Deploy Hello World (teste)**

*Arquivo*
```bash
nano hello-world.yaml
```
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hello
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello
        image: nginxdemos/hello
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: hello
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world
  namespace: hello
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
spec:
  ingressClassName: traefik
  rules:
  - host: deltasofth.cloud
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
  tls:
  - hosts:
    - deltasofth.cloud
    secretName: hello-world-tls
```
```bash
kubectl apply -f hello-world.yaml
```

### 10. **Verificar**
```bash
kubectl get pods -A
kubectl get ingress -A
```

**Resultado:** Site funcionando em `http://deltasofth.cloud` com SSL automático via Let's Encrypt.

---
Add environment → Edge Agent Standard → Kubernetes
Preencha:

Name: Azure-Worker
Portainer API server URL: https://164.92.97.238:30779 ⚠️ IMPORTANTE: Use o IP, não o domínio!

**Stack instalada:**
- K3s (Kubernetes)
- Traefik (Ingress/Reverse Proxy)
- Portainer (Gerenciamento Web)
- Cert-Manager (SSL automático)
