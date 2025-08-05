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

01. **Set up Windows VM** - [Windows VM setup](labs/setup.md)

02. **K3d Getting Started** - [K3D GettingStarted](labs/LAB01-K3D-GettingStarted.md)

03. **K3d and Persistent Volumes** - [K3D PVC](labs/LAB02-K3D-PVC.md)

04. **Building a Helm Chart** - [Helm Chart](labs/LAB04-Helm-Chart-Build-new.md)

05. **Working with Operator SDK** - [OperatorSDK-Helm](labs/LAB05-OperatorSDK-Helm.md)

## Day 2 - Advanced Development & Cloud Integration

06. **Skaffold and k3d** - [Skaffold](labs/LAB06-Skaffold.md)

07. **Prometheus Monitoring** - [Prometheus](labs/LAB07-Prometheus.md)

08. **Tilt and k3d** - [Tilt](labs/Tilt.md)

09. **Remote Development with Telepresence on EKS** - [Telepresence](labs/LAB09-Telepresence.md)

10. **Skaffold Microservices** - [Skaffold-Microservices](labs/LAB10-Skaffold-Microservices.md)
