# Lab 02: Working With Kind

## Overview
In this lab, you'll learn to use Kind (Kubernetes in Docker) for local development. Kind is a tool for running local Kubernetes clusters using Docker container "nodes" and is great for testing and development.

## Prerequisites
- Lab 01 completed (Ubuntu VM with Docker and tools installed)
- VS Code connected to Ubuntu VM via Remote SSH

## Part 1: Install Kind on Ubuntu VM

### Step 1: Install Kind (Latest Version)
```bash
# Connect to Ubuntu VM via VS Code Remote SSH
# Open terminal in VS Code (Ctrl+`)

# Install Kind v0.24.0
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify installation
kind version
```

### Step 2: Create Working Directory
```bash
mkdir -p ~/k8s-labs/kind-labs
cd ~/k8s-labs/kind-labs
```

## Part 2: Create Your First Kind Cluster

### Step 1: Create Basic Cluster
```bash
# Create a simple cluster
kind create cluster --name my-cluster

# Check cluster status
kubectl cluster-info --context kind-my-cluster

# View nodes
kubectl get nodes
```

### Step 2: Explore the Cluster
```bash
# List all pods in all namespaces
kubectl get pods -A

# Check cluster info
kubectl cluster-info dump | head -20
```

## Part 3: Multi-Node Cluster Configuration

### Step 1: Create Multi-Node Cluster Config
Create a file called `kind-config.yaml`:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
```

### Step 2: Create Multi-Node Cluster
```bash
# Delete previous cluster
kind delete cluster --name my-cluster

# Create multi-node cluster
kind create cluster --name multi-node --config kind-config.yaml

# Verify nodes
kubectl get nodes
```

## Part 4: Deploy Sample Application

### Step 1: Create Nginx Deployment
```bash
# Create nginx deployment
kubectl create deployment nginx --image=nginx:1.25

# Expose the deployment
kubectl expose deployment nginx --port=80 --type=NodePort

# Check deployment
kubectl get pods,svc
```

### Step 2: Test Application Access
```bash
# Get node port
kubectl get svc nginx

# Test from within cluster (since we're on VM)
kubectl port-forward svc/nginx 8080:80 &

# Test access (in another terminal)
curl http://localhost:8080
```

## Part 5: Install Ingress Controller

### Step 1: Install Nginx Ingress Controller
```bash
# Install nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Step 2: Create Ingress Resource
Create `nginx-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

Apply the ingress:
```bash
kubectl apply -f nginx-ingress.yaml

# Check ingress
kubectl get ingress
```

## Part 6: Working with Persistent Volumes

### Step 1: Create Persistent Volume
Create `pv-example.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /tmp/data
```

### Step 2: Create Persistent Volume Claim
Create `pvc-example.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: standard
```

### Step 3: Deploy with Persistent Storage
```bash
# Apply PV and PVC
kubectl apply -f pv-example.yaml
kubectl apply -f pvc-example.yaml

# Check status
kubectl get pv,pvc
```

## Part 7: Load Docker Images into Kind

### Step 1: Build Custom Image
```bash
# Create simple Dockerfile
cat > Dockerfile << 'EOF'
FROM nginx:1.25
COPY index.html /usr/share/nginx/html/
EOF

# Create custom index.html
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Kind Lab</title>
</head>
<body>
    <h1>Hello from Kind Cluster!</h1>
    <p>This is running in a Kind cluster on Ubuntu VM</p>
</body>
</html>
EOF

# Build image
docker build -t my-nginx:v1.0 .
```

### Step 2: Load Image into Kind
```bash
# Load image into kind cluster
kind load docker-image my-nginx:v1.0 --name multi-node

# Update deployment to use custom image
kubectl set image deployment/nginx nginx=my-nginx:v1.0

# Check rollout
kubectl rollout status deployment/nginx
```

## Part 8: Cluster Management

### Step 1: View Cluster Information
```bash
# List all kind clusters
kind get clusters

# Get kubeconfig for specific cluster
kind get kubeconfig --name multi-node

# View cluster nodes (as containers)
docker ps
```

### Step 2: Troubleshooting
```bash
# View logs from kind nodes
kind export logs --name multi-node /tmp/kind-logs

# Access node directly
docker exec -it multi-node-control-plane bash
```

## Part 9: Clean Up

### Step 1: Delete Resources
```bash
# Delete deployments and services
kubectl delete deployment nginx
kubectl delete svc nginx
kubectl delete ingress nginx-ingress
kubectl delete pvc example-pvc
kubectl delete pv example-pv
```

### Step 2: Delete Cluster
```bash
# Delete the cluster
kind delete cluster --name multi-node

# Verify deletion
kind get clusters
```

## Troubleshooting

### Common Issues

**1. Docker Permission Errors**
```bash
sudo usermod -aG docker $USER
newgrp docker
```

**2. Port Already in Use**
```bash
# Check what's using the port
sudo netstat -tlnp | grep :80
# Kill process or use different port
```

**3. Ingress Not Working**
```bash
# Check ingress controller status
kubectl get pods -n ingress-nginx
kubectl describe pod -n ingress-nginx
```

## Key Concepts Learned

1. **Kind Clusters**: Local Kubernetes clusters using Docker containers
2. **Multi-Node Setup**: Control plane and worker nodes configuration
3. **Ingress**: External access to services in the cluster
4. **Persistent Volumes**: Storage that persists beyond pod lifecycle
5. **Custom Images**: Building and loading custom Docker images

## Next Steps

You now have experience with:
- Creating and managing Kind clusters
- Deploying applications with persistent storage
- Setting up ingress for external access
- Loading custom Docker images

Proceed to Lab 03: Air Gapped Kind for advanced scenarios.

## Summary
- ✅ Kind installed and configured
- ✅ Multi-node cluster created
- ✅ Sample application deployed
- ✅ Ingress controller configured
- ✅ Persistent volumes working
- ✅ Custom images loaded and used
