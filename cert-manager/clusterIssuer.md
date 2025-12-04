### 🔹 Install cert-manager & Create Let’s Encrypt ClusterIssuer

```bash
# Namespace for cert-manager
kubectl create namespace cert-manager

# Add Jetstack Helm repo
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager (with CRDs)
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set crds.enabled=true

# Check cert-manager pods
kubectl get pods -n cert-manager
```

Create `cluster-issuer-prod.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: rohitrawat891997@gmail.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

Apply and verify:

```bash
kubectl apply -f cluster-issuer-prod.yaml
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod
```
