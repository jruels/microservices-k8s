# Lab 01: Foundation Setup

## Overview
This lab sets up the essential tools for Kubernetes development on your Ubuntu 24.04 VM. You'll use VS Code Remote SSH extension from Windows 11 to connect and manage files, while running k3d and other container tools on the Ubuntu VM.

## Architecture
- **Windows 11**: VS Code, AWS CLI, kubectl, Git Bash, PowerShell (client tools)
- **Ubuntu 24.04 VM**: Docker, k3d, Tilt, Skaffold, Telepresence (container runtime)
- **Connection**: VS Code Remote SSH extension for seamless file editing

## Prerequisites
- Windows 11 machine with VS Code, AWS CLI, kubectl, Git Bash installed
- Ubuntu 24.04 VM with SSH access
- VS Code Remote SSH extension installed on Windows

## Part 1: Connect to Ubuntu VM

### Step 1: Connect via VS Code Remote SSH
1. Open VS Code on Windows
2. Press `Ctrl+Shift+P` and select "Remote-SSH: Connect to Host"
3. Enter: `ubuntu@<your-vm-ip>`
4. Enter password/key when prompted
5. VS Code opens connected to Ubuntu VM

### Step 2: Open Terminal in VS Code
- Press `Ctrl+` (backtick) to open integrated terminal
- You're now running commands on Ubuntu VM while editing with Windows VS Code

## Part 2: Ubuntu VM Setup

### Step 1: Update System
```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2: Install Docker (Required for k3d)
```bash
# Install Docker using official script
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group (avoids sudo for docker commands)
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Apply group changes
newgrp docker

# Test Docker installation
docker --version
docker run hello-world
```

### Step 3: Install k3d
```bash
# Install k3d latest version
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# Verify installation
k3d version
```

### Step 4: Install Development Tools for Ubuntu VM
```bash
# Install essential tools
sudo apt install -y curl wget jq unzip tree git

# Install kubectl on Ubuntu (for local cluster management)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install Tilt
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash

# Install Skaffold
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
chmod +x skaffold
sudo mv skaffold /usr/local/bin

# Install Telepresence
sudo curl -fL https://app.getambassador.io/download/tel2oss/releases/download/v2.23.3/telepresence-linux-amd64 -o /usr/local/bin/telepresence
sudo chmod a+x /usr/local/bin/telepresence
```

## Part 3: Verify Installation

### Step 1: Create Working Directory
```bash
mkdir -p ~/k8s-labs
cd ~/k8s-labs
```

### Step 2: Test Docker
```bash
docker run --rm hello-world
```

### Step 3: Test k3d
```bash
# Create test cluster
k3d cluster create test-cluster --agents 2

# Check cluster status
kubectl get nodes

# Delete test cluster
k3d cluster delete test-cluster
```

### Step 4: Test Other Tools
```bash
# Test tool versions
helm version
tilt version
skaffold version
telepresence version
```

## Part 4: Setup kubeconfig for Windows kubectl

### Step 1: Create kubeconfig accessible from Windows
```bash
# Create a cluster that will be used from Windows
k3d cluster create dev-cluster --api-port 6443 --agents 2

# Get kubeconfig
k3d kubeconfig get dev-cluster > ~/dev-cluster-kubeconfig.yaml

# Show the file path for Windows access
echo "Kubeconfig location: ~/dev-cluster-kubeconfig.yaml"
```

### Step 2: Configure Windows kubectl (Run on Windows)
1. Open PowerShell or Git Bash on Windows
2. Copy the kubeconfig from Ubuntu VM to Windows:
   ```bash
   # Using scp from Windows (Git Bash)
   scp ubuntu@<vm-ip>:~/dev-cluster-kubeconfig.yaml ~/.kube/config
   
   # Or use VS Code to copy the file content and paste into ~/.kube/config
   ```

3. Test from Windows:
   ```bash
   kubectl get nodes
   ```

## Part 5: VS Code Extensions Setup

### Install on Remote Ubuntu Connection
In VS Code (connected to Ubuntu):
1. **Kubernetes** extension - for cluster management
2. **YAML** extension - for Kubernetes manifests
3. **Docker** extension - for container management
4. **GitLens** extension - for Git integration

## Part 6: Network Configuration

### Step 1: Configure k3d Port Access
```bash
# For labs that need external access, create clusters with port mapping
k3d cluster create lab-cluster --api-port 6443 --port "8080:80@loadbalancer" --agents 2
```

### Step 2: Test Network Access
```bash
# Test internal cluster access
kubectl get pods -A

# Test from Windows (ensure port 6443 is accessible)
# This depends on your VM network configuration
```

## Troubleshooting

### Docker Permission Issues
```bash
# If you get permission errors
sudo usermod -aG docker $USER
newgrp docker
# Then reconnect VS Code Remote SSH
```

### k3d Port Access Issues
```bash
# Check if ports are accessible
sudo netstat -tlnp | grep 6443

# Check firewall (if enabled)
sudo ufw status
```

### VS Code Remote Connection Issues
```bash
# Restart SSH service
sudo systemctl restart ssh

# Check SSH status
sudo systemctl status ssh
```

## Workflow Summary

1. **Edit files**: Use VS Code on Windows (connected to Ubuntu via Remote SSH)
2. **Run commands**: Use VS Code integrated terminal (executes on Ubuntu)
3. **Manage clusters**: Use kubectl from Windows or Ubuntu terminal
4. **Container operations**: Run on Ubuntu VM (Docker, k3d, etc.)
5. **AWS operations**: Use AWS CLI from Windows

## Next Steps
Your environment is now configured with:
- Ubuntu VM running Docker and k3d
- Windows machine with kubectl access to clusters
- VS Code seamlessly editing files on Ubuntu from Windows

Proceed to Lab 02: Working With Kind to start building Kubernetes applications.

## Summary
- ✅ Ubuntu VM with Docker and k3d installed
- ✅ Development tools (Tilt, Skaffold, Telepresence) installed
- ✅ VS Code Remote SSH connection working
- ✅ kubectl configured on both Windows and Ubuntu
- ✅ Network ports configured for cluster access
