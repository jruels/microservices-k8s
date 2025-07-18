# Lab 01: K3d Getting Started

In this lab, you will learn how to install and use k3d to create local Kubernetes clusters.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker installed and running
- VS Code connected to Ubuntu VM via Remote Development extension
- kubectl installed on Ubuntu VM

## Environment Setup

### Set up SSH connection in Visual Studio Code

1. **Open Visual Studio Code on your Windows VM**

2. **Create the SSH configuration file:**
   - On the left sidebar, click the Remote Explorer icon (computer with connection icon)
   - In the Remote Explorer, hover over **SSH**, click the gear icon (⚙️), and select the SSH config file (usually `C:\Users\<username>\.ssh\config`)

3. **Add SSH configuration for your Ubuntu VM:**
   Add the following to your SSH config file, replacing the placeholders with your actual values:
   ```plaintext
   Host ubuntu-vm
     HostName <IP address of your Ubuntu VM>
     IdentityFile <Path to your SSH private key file>
     User ubuntu
   ```
   **PROTIP**: If you have the SSH key file in VS Code Explorer, right-click it and select "Copy Path" to get the correct path for `IdentityFile`

4. **Save the SSH configuration file**

5. **Connect to your Ubuntu VM:**
   - In the Remote Explorer, you should now see "ubuntu-vm" under "SSH Targets"
   - Click on the entry to connect to your Ubuntu VM
   - VS Code will open a new window connected to your Ubuntu VM
   - Click **Open Folder** and select `/home/ubuntu` as your working directory

6. **Open integrated terminal:**
   - In the connected VS Code window, open Terminal → New Terminal
   - All commands in this lab will be executed in this VS Code integrated terminal
   - Use VS Code's file explorer to navigate and manage files/directories on the Ubuntu VM

## Install k3d

Using the VS Code integrated terminal connected to your Ubuntu VM, install k3d with the Linux installation method:

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

**Verify Docker is accessible (since k3d requires Docker):**
```bash
docker ps
```

This should list running containers without any permission errors.

## Create Your First k3d Cluster

1. **Check existing clusters:**
   ```bash
   k3d cluster list
   ```

2. **Create a cluster with 1 server and 2 agents (takes 1-3 minutes):**
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

**Verify kubectl context is set correctly:**
```bash
kubectl config current-context
```
This should show: `k3d-demo`

**Verify all nodes are ready:**
```bash
kubectl get nodes
```
You should see 3 nodes (1 server + 2 agents) with STATUS "Ready"

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

## End-to-End Cluster Test

Test that the cluster can actually run workloads:

```bash
# Deploy a test pod
kubectl create deployment test-nginx --image=nginx:latest

# Wait for pod to be ready
kubectl wait --for=condition=ready pod -l app=test-nginx --timeout=60s

# Verify deployment is working
kubectl get pods -l app=test-nginx

# Clean up test deployment
kubectl delete deployment test-nginx
```

If all commands succeed, your cluster is working correctly!

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

**Verify cleanup was successful:**
```bash
# Should show no 'demo' cluster
k3d cluster list

# Should show no k3d-demo containers
docker ps -a | grep k3d-demo
```

If the second command returns no results, cleanup was successful.

## Summary

In this lab, you learned how to:
- Install k3d on your system
- Create a multi-node Kubernetes cluster locally
- Configure kubectl to connect to your cluster
- Clean up resources when done

---

**Next Lab:** [LAB02-K3D-PVC.md](LAB02-K3D-PVC.md)
