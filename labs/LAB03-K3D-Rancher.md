# Lab 03: K3d, Helm and Rancher

In this lab, you will learn how to work with k3d clusters and install Rancher using Helm for cluster management.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, k3d, and kubectl installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Completed Lab 01: K3d Getting Started

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Development. All commands will be executed on the Ubuntu VM.

## Install Helm

First, install Helm on your Ubuntu VM:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify Helm installation:

```bash
helm version
```

## Create k3d Cluster for Rancher

Create a k3d cluster specifically configured for Rancher:

```bash
k3d cluster create rancher --api-port 6552 --servers 1 --agents 2 --port "443:443@loadbalancer" --port "80:80@loadbalancer" --wait
```

You should see output similar to:
```
INFO[0000] Created network 'k3d-rancher'
INFO[0000] Created volume 'k3d-rancher-images'
INFO[0001] Creating node 'k3d-rancher-server-0'
INFO[0001] Creating node 'k3d-rancher-agent-0'
INFO[0001] Creating node 'k3d-rancher-agent-1'
INFO[0005] Creating node 'k3d-rancher-agent-2'
INFO[0005] Creating LoadBalancer 'k3d-rancher-serverlb'
INFO[0013] Cluster 'rancher' created successfully!
```

## Verify Cluster Setup

Verify your cluster is running:

```bash
k3d cluster list
kubectl cluster-info
kubectl get nodes
```

## Install cert-manager

Rancher requires cert-manager for TLS certificate management. Install it using Helm:

```bash
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.14.0 \
  --set installCRDs=true
```

Verify cert-manager is running:

```bash
kubectl get pods --namespace cert-manager
```

Wait for all cert-manager pods to be Running before proceeding.

## Install Rancher

Add the Rancher Helm repository:

```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

Install Rancher:

```bash
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher.localhost \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=rancher \
  --set replicas=1
```

## Verify Rancher Installation

Check the Rancher deployment:

```bash
kubectl -n cattle-system get pods
```

Wait for all Rancher pods to be Running. This may take a few minutes.

## Access Rancher UI

1. **Get your Ubuntu VM IP address:**
   ```bash
   hostname -I | awk '{print $1}'
   ```

2. **Create a tunnel to access from Windows VM:**
   ```bash
   kubectl -n cattle-system port-forward service/rancher 8080:80 --address=0.0.0.0
   ```

3. **Access Rancher UI:**
   - Open your web browser on the Windows VM
   - Navigate to: `http://[UBUNTU_VM_IP]:8080`
   - Use the bootstrap password: `admin`

## Troubleshooting

**If cert-manager pods don't start:**
```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager
```

**If Rancher doesn't start:**
```bash
# Check Rancher pods
kubectl get pods -n cattle-system

# Check Rancher logs
kubectl logs -n cattle-system -l app=rancher

# Check if cert-manager is ready first
kubectl get pods -n cert-manager
```

**If you can't access the UI:**
```bash
# Verify port forward is working
netstat -tlnp | grep 8080

# Check Ubuntu VM firewall
sudo ufw status

# If firewall is active, allow the port
sudo ufw allow 8080
```

## Explore Rancher Features

Once logged in to Rancher:

1. **Change the default password** when prompted
2. **Explore the dashboard** - View cluster resources, nodes, and workloads
3. **Deploy a sample application** through the Rancher UI
4. **View monitoring** - Check built-in monitoring capabilities

## Deploy Sample Application via Rancher

1. In Rancher UI, go to **Workloads** > **Deployments**
2. Click **Create**
3. Use these settings:
   - **Name**: `nginx-demo`
   - **Image**: `nginx:latest`
   - **Replicas**: `3`
4. Click **Create**

## Verify Application Deployment

Check the deployment via kubectl:

```bash
kubectl get deployments
kubectl get pods
kubectl get services
```

## Create a Service for the Application

Create a service to expose the nginx application:

```bash
kubectl expose deployment nginx-demo --type=NodePort --port=80
```

Check the service:

```bash
kubectl get services
```

## Clean Up

When you're done with the lab:

```bash
# Delete the sample application
kubectl delete deployment nginx-demo
kubectl delete service nginx-demo

# Uninstall Rancher
helm uninstall rancher -n cattle-system

# Uninstall cert-manager
helm uninstall cert-manager -n cert-manager

# Delete the cluster
k3d cluster delete rancher
```

## Summary

In this lab, you learned how to:
- Install Helm on your Ubuntu VM
- Create a k3d cluster suitable for Rancher
- Install cert-manager as a prerequisite for Rancher
- Install Rancher using Helm
- Access the Rancher UI from your Windows VM
- Deploy applications through the Rancher interface
- Manage Kubernetes resources via Rancher
- Clean up resources when done

---

**Next Lab:** [LAB04-Helm-Chart-Build.md](LAB04-Helm-Chart-Build.md)