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
