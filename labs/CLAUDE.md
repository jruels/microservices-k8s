# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Kubernetes training lab repository focused on microservice development, testing, and optimization. The repository contains lab exercises, configuration files, and documentation for various Kubernetes development tools and workflows.

## Key Architecture Components

### Development Tools Integration
- **Skaffold**: Continuous development for Kubernetes applications with hot reloading
- **Telepresence**: Local debugging of microservices running in remote Kubernetes clusters  
- **Tilt**: Live update and development workflow automation for Kubernetes

### Kubernetes Resources
- Sample deployments with persistent volumes (PV/PVC)
- Kustomization configurations for managing secrets and resources
- Various service exposure patterns (NodePort, Ingress)
- Monitoring stack configurations (Prometheus, Grafana)

## Common Development Commands

### Cluster Management
```bash
# Create k3d clusters for different tools
k3d cluster create skaffold --api-port 6550 --agents 2
k3d cluster create tilt --api-port 6550 --agents 2

# Delete clusters when done
k3d cluster delete <cluster-name>
```

### Skaffold Development
```bash
# Continuous development mode (watches for changes)
skaffold dev --default-repo localhost:32000

# One-time deployment
skaffold run --default-repo localhost:32000

# Initialize skaffold config in new projects
skaffold init

# Clean up deployments
skaffold delete
```

### Telepresence Debugging
```bash
# Connect to remote cluster
telepresence connect

# Create intercept for debugging
telepresence intercept <service-name> --port <local-port>

# Test remote service connectivity
curl http://<service-name>.<namespace>.svc.cluster.local:<port>/<endpoint>
```

### Tilt Development
```bash
# Start development environment
tilt up

# Demo mode (creates temporary cluster and sample project)
tilt demo

# Access UI at http://localhost:3366/
```

### Kubernetes Operations
```bash
# Apply configurations
kubectl apply -f <yaml-file>

# Apply kustomization
kubectl apply -k .

# Monitor resources
kubectl get pods,services,pv,pvc
```

## Development Workflow

1. **Local Development Setup**: Use k3d to create local Kubernetes clusters that simulate remote environments
2. **Continuous Development**: Leverage Skaffold or Tilt for automatic rebuilds and deployments on code changes
3. **Remote Debugging**: Use Telepresence to intercept remote traffic and debug locally while maintaining full cluster context
4. **Resource Management**: Use Kustomize for environment-specific configurations and secret management

## File Structure Context

- `.yaml` files: Kubernetes resource definitions (deployments, services, PVs, etc.)
- `kustomization.yaml`: Kustomize configuration for resource management and secret generation
- `.md` files: Tool-specific documentation and tutorials for Skaffold, Telepresence, and Tilt
- `.pdf` files: Lab exercise instructions

## Important Notes

- The repository uses k3d for local Kubernetes clusters with specific API ports (6550)
- Default container registry for Skaffold is localhost:32000 (insecure registry)
- Tilt provides a web UI on localhost:3366 for monitoring development workflows
- Telepresence enables bidirectional networking between local development and remote clusters