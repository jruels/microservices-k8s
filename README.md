# DevOps for Microservices with Kubernetes
This page contains the labs for the Developing microservices on Kubernetes class.

## Environment Setup
[Windows VM info](VM_access.md)   
[AWS accounts](https://docs.google.com/spreadsheets/d/1-1do2X-daaI3w1Z1TPPsReWpmRj2tLXXk4MND6tcpno/edit?usp=sharing)

This course uses a dual-VM setup:
- **Windows VM**: VS Code with Remote Development extension, AWS CLI
- **Ubuntu VM**: Docker, k3d, kubectl, and development tools
- **Connection**: VS Code Remote Development extension connects Windows VM to Ubuntu VM

Students will edit files via VS Code which is remotely connected to the Ubuntu VM.   
All commands will be executuded in VS Code's terminal remotely connected to Ubuntu VM.  

## Day 1 - Local k3d Development & Core Tools

0. **Set up Windows VM** - [Windows VM setup](labs/setup.md)

1. **K3d Getting Started** - [LAB01-K3D-GettingStarted.md](labs/LAB01-K3D-GettingStarted.md)

2. **K3d and Persistent Volumes** - [LAB02-K3D-PVC.md](labs/LAB02-K3D-PVC.md)

3. **Building a Helm Chart** - [LAB04-Helm-Chart-Build.md](labs/LAB04-Helm-Chart-Build-new.md)

4. **Working with Operator SDK** - [LAB05-OperatorSDK-Helm.md](labs/LAB05-OperatorSDK-Helm.md)

## Day 2 - Advanced Development & Cloud Integration

05. **Skaffold and k3d** - [LAB06-Skaffold.md](labs/LAB06-Skaffold.md)

06. **Prometheus Monitoring** - [LAB07-Prometheus.md](labs/LAB07-Prometheus.md)

07. **Tilt and k3d** - [LAB08-Tilt.md](labs/Tilt.md)

08. **Remote Development with Telepresence on EKS** - [LAB09-Telepresence.md](labs/LAB09-Telepresence.md)

09. **Skaffold Microservices** - [LAB10-Skaffold-Microservices.md](labs/LAB10-Skaffold-Microservices.md)
