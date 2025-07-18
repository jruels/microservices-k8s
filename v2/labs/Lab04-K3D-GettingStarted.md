# Lab 04: K3D Getting Started

## Overview
This lab introduces k3d, a lightweight wrapper to run k3s (Rancher Lab's minimal Kubernetes distribution) in Docker. k3d makes it easy to create multi-node k3s clusters for development and testing.

## Prerequisites
- Ubuntu VM with Docker and k3d installed
- VS Code connected to Ubuntu VM via Remote SSH

## Part 1: Verify k3d Installation

### Step 1: Check k3d Version
```bash
k3d version  # Should show the latest version

docker ps    # Docker must be running
```

### Step 2: Create Working Directory
```bash
mkdir -p ~/k8s-labs/k3d-labs
cd ~/k8s-labs/k3d-labs
```

## Part 2: Create Your First k3d Cluster

### Step 1: Create Basic Cluster
```bash
k3d cluster create my-cluster
kubectl cluster-info
kubectl get nodes
```

### Step 2: Explore the Cluster
```bash
kubectl get pods -A
kubectl get pods -n kube-system | grep -E "(traefik|local-path)"
```

## Part 3: Multi-Node Cluster with Custom Configuration

### Step 1: Delete Previous Cluster
```bash
k3d cluster delete my-cluster
```

### Step 2: Create Multi-Node Cluster
```bash
k3d cluster create dev-cluster --agents 2 --api-port 6550
kubectl get nodes
```

### Step 3: List k3d Clusters and Nodes
```bash
k3d cluster list
k3d node list
```

docker ps | grep k3d  # See k3d containers
```

## Cleanup
```bash
k3d cluster delete dev-cluster
```

## Next Steps
Proceed to the next lab for persistent volumes.
