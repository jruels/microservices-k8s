# DevOps for Microservices with Kubernetes
*2-Day Intensive Course*

## Environment Setup
This course uses a dual-VM setup:
- **Windows VM**: VS Code with Remote Development extension, AWS CLI
- **Ubuntu VM**: Docker, k3d, kubectl, and development tools
- **Connection**: VS Code Remote Development extension connects Windows VM to Ubuntu VM

Students will edit files via VS Code on Windows while executing commands on the Ubuntu VM.

## Day 1 - Local k3d Development & Core Tools

01. **K3d Getting Started** - [LAB01-K3D-GettingStarted.md](labs/LAB01-K3D-GettingStarted.md)

02. **K3d and Persistent Volumes** - [LAB02-K3D-PVC.md](labs/LAB02-K3D-PVC.md)

03. **Building a Helm Chart** - [LAB04-Helm-Chart-Build.md](labs/LAB04-Helm-Chart-Build-new.md)

04. **Working with Operator SDK** - [LAB05-OperatorSDK-Helm.md](labs/LAB05-OperatorSDK-Helm.md)

## Day 2 - Advanced Development & Cloud Integration

05. **Skaffold and k3d** - [LAB06-Skaffold.md](labs/LAB06-Skaffold.md)

06. **Prometheus Monitoring** - [LAB07-Prometheus.md](labs/LAB07-Prometheus.md)

07. **Tilt and k3d** - [LAB08-Tilt.md](labs/Tilt.md)

08. **Remote Development with Telepresence on EKS** - [LAB09-Telepresence.md](labs/LAB09-Telepresence.md)

09. **Skaffold Microservices** - [LAB10-Skaffold-Microservices.md](labs/LAB10-Skaffold-Microservices.md)

---

## Prerequisites

### Windows VM Requirements
- VS Code installed
- Remote Development extension pack installed
- AWS CLI installed
- Network connectivity to Ubuntu VM

### Ubuntu VM Requirements
- Docker installed and running
- kubectl installed
- k3d installed
- Helm installed
- Go programming language (installed during labs)
- Basic understanding of Kubernetes concepts
- Terminal/command line familiarity

### Development Tools (Installed During Labs)
- Skaffold
- Tilt
- Telepresence
- Operator SDK
- Prometheus

## Course Objectives
By the end of this course, students will be able to:
- Set up and manage local Kubernetes clusters using k3d
- Deploy and manage applications using Helm charts
- Create custom operators using Operator SDK
- Implement monitoring with Prometheus
- Use modern development tools like Skaffold and Tilt for fast iteration
- Debug applications remotely with Telepresence
- Work with persistent volumes in k3d
- Manage cluster resources using Rancher
- Deploy microservices applications effectively
