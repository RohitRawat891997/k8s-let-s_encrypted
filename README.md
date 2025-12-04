This is a solid setup flow for EKS + NGINX Ingress + cert-manager + Let’s Encrypt + custom domain.

1. ✅ First: **clean + correct your notes**
2. ✅ Then: **convert it into an easy LinkedIn post explanation**

---

## 1️⃣ Clean & Corrected Notes (with small fixes)

### 🔹 Install basics, AWS CLI, Docker, kubectl & eksctl

```bash
# Become root
sudo -i

# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
sudo apt install -y docker.io
docker ps

# Give current user permission to use Docker
sudo chown $USER /var/run/docker.sock

# Install unzip (for AWS CLI)
sudo apt install -y unzip

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS credentials
aws configure
```

```bash
# Install kubectl (CLI for Kubernetes)
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client

# Install eksctl (to create/manage EKS clusters)
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

### 🔹 Create EKS Cluster & Connect

```bash
# Create EKS cluster
eksctl create cluster \
  --name three-tier-cluster \
  --region us-east-1 \
  --node-type t2.medium \
  --nodes-min 2 \
  --nodes-max 2

# Update kubeconfig to talk to the new cluster
aws eks update-kubeconfig --region us-east-1 --name three-tier-cluster

# Verify nodes
kubectl get nodes
```

---

### 🔹 Install Helm & NGINX Ingress Controller

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add Helm repo for ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install NGINX Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.type=LoadBalancer \
  --set controller.metrics.enabled=true \
  --set controller.admissionWebhooks.enabled=true

# Get LoadBalancer external IP
kubectl get svc -n ingress-nginx
```
<img width="933" height="143" alt="image" src="https://github.com/user-attachments/assets/2ce23468-51d1-442e-a3f7-7dec6165a6dc" />
<img width="753" height="205" alt="image" src="https://github.com/user-attachments/assets/8d7752e4-fcd5-4894-a500-5a5379078d30" />
<img width="789" height="552" alt="image" src="https://github.com/user-attachments/assets/41d30d08-ad6a-473f-8fba-3bdf2e568f11" />


---

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

---

### 🔹 Deploy Application (Deployment + Service)

```bash
kubectl create namespace production
```

Create `app-deploy.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: java
          image: rohitrawat891997/java:v1
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: production
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
```

Apply:

```bash
kubectl apply -f app-deploy.yaml
kubectl get all -n production
```

---

### 🔹 Create Ingress with TLS (app.ur-voip.com)

Create `app-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - app.ur-voip.com
      secretName: app-ur-voip-com-tls
  rules:
    - host: app.ur-voip.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

Apply and verify:

```bash
kubectl apply -f app-ingress.yaml
kubectl get ingress -n production
kubectl describe ingress my-app-ingress -n production
```
