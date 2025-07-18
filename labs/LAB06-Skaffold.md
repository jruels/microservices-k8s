# Lab 06: Skaffold and k3d

In this lab, you will learn how to use Skaffold to optimize the development workflow for Kubernetes applications.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, k3d, and kubectl installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Completed Lab 01: K3d Getting Started

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension (see Lab 01 for SSH setup). All commands will be executed in the VS Code integrated terminal connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories and files.

## Install Skaffold

Install Skaffold on your Ubuntu VM:

```bash
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64
sudo install skaffold /usr/local/bin/
```

Verify installation:

```bash
skaffold version
```

## Create k3d Cluster

Create a k3d cluster for this lab:

```bash
k3d cluster create skaffold --api-port 6555 --agents 2 --wait
```

## Create Working Directory

Create the working directory using VS Code file explorer or terminal:

**Option 1: Using VS Code File Explorer**
- In VS Code's file explorer, navigate to `/home/ubuntu/k3d` (create `k3d` folder if it doesn't exist)
- Right-click and create a new folder called `skaffold`
- Navigate to this directory in the terminal

**Option 2: Using Terminal**
```bash
mkdir -p ~/k3d/skaffold
cd ~/k3d/skaffold
```

## Clone Demo Application

Clone the Skaffold demo repository:

```bash
git clone https://github.com/jruels/skaffold-demo.git
cd skaffold-demo
```

## Examine the Application

Look at the application structure:

```bash
ls -la
cat src/main/java/hello/Application.java
```

## Create Dockerfile

Create a Dockerfile for the Spring Boot application. You can either:
- **Use VS Code**: Create a new file called `Dockerfile` in VS Code editor and add the contents below
- **Use terminal**: Run the command below to create the file

```bash
cat > Dockerfile << 'EOF'
FROM maven:3-jdk-11 as BUILD
COPY . /usr/src/app
RUN mvn --batch-mode -f /usr/src/app/pom.xml clean package

FROM openjdk:11-jre-slim
ENV PORT 8080
EXPOSE 8080
COPY --from=BUILD /usr/src/app/target /opt/target
WORKDIR /opt/target
CMD ["/bin/bash", "-c", "find -type f -name '*.jar' | xargs java -jar"]
EOF
```

## Create Kubernetes Manifest

Create a Kubernetes deployment manifest:

```bash
cat > k8s-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: skaffold-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: skaffold-demo
  template:
    metadata:
      labels:
        app: skaffold-demo
    spec:
      containers:
      - name: skaffold-demo
        image: skaffold-demo
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: skaffold-demo
spec:
  selector:
    app: skaffold-demo
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  type: NodePort
EOF
```

## Initialize Skaffold

Initialize Skaffold configuration:

```bash
skaffold init --generate-manifests=false
```

When prompted, select the appropriate options for your Dockerfile and Kubernetes manifest.

## Examine Skaffold Configuration

View the generated skaffold.yaml:

```bash
cat skaffold.yaml
```

## Development with Skaffold

Start development mode (watches for changes and redeploys automatically):

```bash
skaffold dev
```

Skaffold will:
1. Build the Docker image
2. Deploy to your k3d cluster
3. Stream logs
4. Watch for file changes

## Test the Application

In a new terminal, test the application:

```bash
kubectl get pods
kubectl get services
kubectl port-forward service/skaffold-demo 8080:8080
```

In another terminal:

```bash
curl http://localhost:8080
```

## Make Code Changes

Edit the source code and watch Skaffold automatically rebuild and redeploy:

```bash
# Edit the main application file
sed -i 's/Hello World!/Hello Skaffold!/' src/main/java/hello/Application.java
```

Watch the Skaffold terminal - it should detect the change and automatically rebuild.

## One-time Deployment

Stop the development mode (Ctrl+C) and try a one-time deployment:

```bash
skaffold run
```

## Debug Mode

Start in debug mode:

```bash
skaffold debug
```

This enables debugging capabilities for your application.

## Clean Up

Delete the deployment:

```bash
skaffold delete
```

Delete the cluster:

```bash
k3d cluster delete skaffold
```

## Summary

In this lab, you learned how to:
- Install and configure Skaffold
- Create a development environment with automatic rebuilds
- Use Skaffold for continuous development workflows
- Deploy applications once with Skaffold run
- Enable debugging capabilities
- Clean up resources

Skaffold significantly speeds up the development cycle by automating the build, push, and deploy process while providing real-time feedback.

---

## Troubleshooting

### Common Issues:

1. **Skaffold build errors**
   - Ensure Docker is running: `docker ps`
   - Check Dockerfile syntax and build context
   - Verify Maven/Java installation for Spring Boot apps

2. **Port forwarding issues**
   - Check if port 8080 is already in use: `netstat -tlnp | grep 8080`
   - Use a different port: `kubectl port-forward service/skaffold-demo 8081:8080`

3. **File sync issues**
   - Ensure files are in the correct directory structure
   - Check skaffold.yaml configuration
   - Verify watch patterns are correct

4. **Application not responding**
   - Check pod logs: `kubectl logs -l app=skaffold-demo`
   - Verify service configuration: `kubectl get svc skaffold-demo`

---

**Next Lab:** [LAB07-Prometheus.md](LAB07-Prometheus.md)