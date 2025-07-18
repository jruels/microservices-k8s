# Lab 01: K3d Getting Started

In this lab, you will learn how to install and use k3d to create local Kubernetes clusters.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker installed and running
- VS Code connected to Ubuntu VM via Remote Development extension
- kubectl installed on Ubuntu VM

## Environment Setup
You should be connected to your Ubuntu VM through VS Code Remote Development. All commands in this lab will be executed on the Ubuntu VM through the VS Code terminal.

## Install k3d

Since you're working on an Ubuntu VM, use the Linux installation method:

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

**Note:** If you prefer to install manually, you can download the binary from the [k3d releases page](https://github.com/rancher/k3d/releases) and place it in your PATH.

## Verify k3d Installation

```bash
k3d version
```

You should see output similar to:
```
k3d version v5.x.x
k3s version v1.28.x-k3s1 (default)
```

## Create Your First k3d Cluster

1. **Check existing clusters:**
   ```bash
   k3d cluster list
   ```

2. **Create a cluster with 1 server and 2 agents:**
   ```bash
   k3d cluster create demo --servers 1 --agents 2 --api-port 6550 --wait
   ```

   You should see output indicating the cluster creation process:
   ```
   INFO[0000] Created network 'k3d-demo'
   INFO[0000] Created volume 'k3d-demo-images'
   INFO[0000] Creating initializing server node
   INFO[0000] Creating node 'k3d-demo-server-0'
   ...
   INFO[0158] Cluster 'demo' created successfully!
   ```

3. **Verify cluster status:**
   ```bash
   k3d cluster list
   ```

4. **List all nodes:**
   ```bash
   k3d node list
   ```

## Configure kubectl

The cluster configuration is automatically added to your kubeconfig. Test the connection:

```bash
kubectl cluster-info
```

You should see:
```
Kubernetes control plane is running at https://0.0.0.0:6550
CoreDNS is running at https://0.0.0.0:6550/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:6550/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

## Verify Cluster is Working

Run additional verification commands:

```bash
# Check nodes
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check cluster status
kubectl get all --all-namespaces
```

## Troubleshooting

**If k3d installation fails:**
```bash
# Check if Docker is running
sudo systemctl status docker

# If Docker is not running, start it
sudo systemctl start docker
```

**If cluster creation fails:**
```bash
# Check Docker permissions
sudo usermod -aG docker $USER
# Log out and log back in

# Or run with sudo temporarily
sudo k3d cluster create demo --servers 1 --agents 2 --api-port 6550 --wait
```

## Clean Up

When you're done with the cluster:

```bash
k3d cluster delete demo
```

## Summary

In this lab, you learned how to:
- Install k3d on your system
- Create a multi-node Kubernetes cluster locally
- Configure kubectl to connect to your cluster
- Clean up resources when done

---

**Next Lab:** [LAB02-K3D-PVC.md](LAB02-K3D-PVC.md)
