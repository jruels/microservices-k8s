# Lab 10: Skaffold Microservices

In this lab, you will work with a multi-service application using Skaffold for microservices development and deployment.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, k3d, kubectl, and Skaffold installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Completed Lab 09: Remote Development with Telepresence on EKS

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension (see Lab 01 for SSH setup). All commands will be executed in the VS Code integrated terminal connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories and files.

## Create k3d Cluster

Create a k3d cluster for this lab:

```bash
k3d cluster create microservices --api-port 6559 --servers 1 --agents 2 --port "80:80@loadbalancer" --wait
```

## Clone the Skaffold Microservices Example

Clone the official Skaffold microservices example:

**Option 1: Using VS Code File Explorer + Terminal**
- In VS Code's file explorer, navigate to `/home/ubuntu/k3d` (create `k3d` folder if it doesn't exist)
- Right-click and create a new folder called `microservices`
- Navigate to this directory in the terminal, then run the git commands

**Option 2: Using Terminal**
```bash
mkdir -p ~/k3d/microservices
cd ~/k3d/microservices
git clone https://github.com/GoogleContainerTools/skaffold.git
cd skaffold/examples/microservices
```

## Explore the Application Architecture

Examine the application structure using VS Code file explorer or terminal:

**Using VS Code**: Navigate through the folders in the file explorer to see the structure
**Using Terminal**:
```bash
ls -la
```

You should see:
- `leeroy-app/` - Backend service
- `leeroy-web/` - Frontend web service  
- `base/` - Shared base image
- `skaffold.yaml` - Skaffold configuration

## Examine the Skaffold Configuration

Look at the main Skaffold configuration using VS Code or terminal:

**Using VS Code**: Double-click `skaffold.yaml` in the file explorer to open it
**Using Terminal**:
```bash
cat skaffold.yaml
```

This shows:
- Multiple build artifacts
- Service dependencies
- Port forwarding configuration
- Deployment manifests

## Examine the Services

Look at the backend service using VS Code file explorer or terminal:

**Using VS Code**: Navigate to `leeroy-app/` folder and open the files
**Using Terminal**:
```bash
cat leeroy-app/app.go
cat leeroy-app/Dockerfile
cat leeroy-app/kubernetes/deployment.yaml
```

Look at the frontend service using VS Code file explorer or terminal:

**Using VS Code**: Navigate to `leeroy-web/` folder and open the files
**Using Terminal**:
```bash
cat leeroy-web/app.py
cat leeroy-web/Dockerfile
cat leeroy-web/kubernetes/deployment.yaml
```

## Run the Application

Start the development mode:

```bash
skaffold dev
```

This will:
1. Build all service images
2. Deploy to your k3d cluster
3. Set up port forwarding
4. Watch for changes and automatically rebuild/redeploy

## Test the Application

The application should be accessible at http://localhost:4503

In a new terminal, test the application:

```bash
curl http://localhost:4503
```

You should see the frontend calling the backend service.

## Make Changes and Watch Auto-Rebuild

### Modify the Backend Service

Edit the backend service:

```bash
sed -i 's/leeroooooy app/UPDATED leeroooooy app/' leeroy-app/app.go
```

Watch the Skaffold terminal - it should detect the change and rebuild only the backend service.

### Modify the Frontend Service

Edit the frontend service:

```bash
sed -i 's/Hello World/Hello Microservices/' leeroy-web/app.py
```

Again, watch Skaffold rebuild only the changed service.

## View Service Dependencies

Skaffold handles service dependencies automatically. The frontend service depends on the backend service, and Skaffold ensures proper build order.

## Explore Port Forwarding

Check the port forwarding configuration in `skaffold.yaml`:

```yaml
portForward:
- resourceType: service
  resourceName: leeroy-web
  port: 4503
  localPort: 4503
```

## Test Different Profiles

The example may include different profiles for different environments:

```bash
skaffold config list
```

## Debug the Application

Stop the current session (Ctrl+C) and try debug mode:

```bash
skaffold debug
```

This enables debugging capabilities for supported languages.

## Run Once Mode

Try running the application once without watching for changes:

```bash
skaffold run
```

## View Logs

Check the logs of the running services:

```bash
kubectl logs -f deployment/leeroy-web
kubectl logs -f deployment/leeroy-app
```

## Scale the Services

Scale the backend service:

```bash
kubectl scale deployment leeroy-app --replicas=3
```

Watch how the frontend continues to work with multiple backend instances.

## Clean Up

Delete the deployment:

```bash
skaffold delete
```

Delete the cluster:

```bash
k3d cluster delete microservices
```

## Summary

In this lab, you learned how to:
- Work with multi-service applications using Skaffold
- Understand service dependencies and build order
- Use automatic rebuilding and redeployment
- Test different development modes (dev, debug, run)
- Handle port forwarding for multiple services
- Scale microservices independently
- Debug complex microservice applications

This demonstrates how Skaffold simplifies the development workflow for complex microservices applications by handling the build, deploy, and development lifecycle automatically.

---

**Course Complete!** You have successfully completed all labs in the 2-day DevOps for Microservices with Kubernetes course.