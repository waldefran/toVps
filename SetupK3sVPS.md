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

### 8. **Configurar DNS no Cloudflare**
- Tipo A: `deltasofth.cloud` → `164.92.97.238` (proxy desativado ☁️)
- CNAME: `www` → `deltasofth.cloud`

### 9. **Deploy Hello World (teste)**
```bash
nano hello-world.yaml
```
*(arquivo completo do exemplo anterior)*
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

**Stack instalada:**
- K3s (Kubernetes)
- Traefik (Ingress/Reverse Proxy)
- Portainer (Gerenciamento Web)
- Cert-Manager (SSL automático)
