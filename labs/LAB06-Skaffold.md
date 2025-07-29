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

Look at the application structure to understand what we're working with:

```bash
ls -la
tree src/
cat src/main/java/org/roukou/skaffold/SkaffoldDemoApplication.java
cat src/main/java/org/roukou/skaffold/controller/RoukouController.java
cat src/main/resources/application.properties
```

These commands help us:
- `ls -la`: Lists all files in the project root to see what's available
- `tree src/`: Shows the source code directory structure in a tree format
- `cat SkaffoldDemoApplication.java`: Views the main Spring Boot application class that starts the service
- `cat RoukouController.java`: Views the REST controller that defines our API endpoints and handles HTTP requests
- `cat application.properties`: Shows the Spring Boot configuration, including the server port (42050)

## Create Dockerfile

Create a Dockerfile to containerize the Spring Boot application. This multi-stage build will:
1. Use Maven to compile and package the Java application
2. Create a lightweight runtime image with just the JRE and our JAR file

You can either:
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

Create a Kubernetes deployment manifest that defines how our application should run in the cluster. This YAML file includes:
- A **Deployment** to manage our application pods with replica count and container specifications
- A **Service** to expose our application internally and provide load balancing
- **NodePort** service type to allow external access for testing

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
        - containerPort: 42050
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
    port: 42050
    targetPort: 42050
  type: NodePort
EOF
```

## Initialize Skaffold

Initialize Skaffold configuration to automate the build and deployment process:

```bash
skaffold init --generate-manifests=false
```

This command:
- Creates a `skaffold.yaml` configuration file
- Scans for Dockerfiles and Kubernetes manifests in your project
- Sets up the build and deploy pipeline configuration
- The `--generate-manifests=false` flag tells Skaffold to use our existing k8s-app.yaml instead of generating new manifests

When prompted:
1. **Choose the builder**: Select "Docker (Dockerfile)" (you may see a Java/Jib warning, which is normal if Java isn't installed locally)
2. **Choose builders for Kubernetes resources**: Press `<left>` to select none, or simply press Enter without selecting anything. Since we used `--generate-manifests=false` and already have our `k8s-app.yaml` file, we don't need Skaffold to generate additional manifests.

## Examine and Update Skaffold Configuration

View the generated skaffold.yaml to understand how Skaffold will manage your application:

```bash
cat skaffold.yaml
```

This file defines:
- **Build configuration**: How to build Docker images (which Dockerfile to use)
- **Deploy configuration**: Which Kubernetes manifests to apply
- **Profiles**: Different configurations for different environments (dev, staging, prod)

## Configure Automatic Port Forwarding

To avoid manually managing port-forward connections when pods restart during development, add Skaffold's built-in port forwarding. Update your skaffold.yaml file:

```bash
cat > skaffold.yaml << 'EOF'
apiVersion: skaffold/v4beta13
kind: Config
metadata:
  name: skaffold-demo
build:
  artifacts:
    - image: skaffold-demo
      docker:
        dockerfile: Dockerfile
manifests:
  rawYaml:
    - k8s-app.yaml
portForward:
  - resourceType: service
    resourceName: skaffold-demo
    namespace: default
    port: 42050
    localPort: 8080
EOF
```

The **portForward** section provides several benefits:
- **Automatic port forwarding** during `skaffold dev` without manual intervention
- **Persistent connections** that survive pod restarts during code changes
- **Eliminates manual reconnections** that would otherwise be needed when Skaffold redeploys
- **Maps the Spring Boot service port 42050** to **local port 8080** for convenient testing
- **Namespace specification** ensures Skaffold finds the service in the correct location

## Development with Skaffold

Start development mode for continuous development workflow:

```bash
skaffold dev
```

Skaffold development mode provides:
1. **Automated builds**: Builds the Docker image from your Dockerfile
2. **Automatic deployment**: Deploys to your k3d cluster using the Kubernetes manifests
3. **Log streaming**: Shows real-time logs from your running pods
4. **File watching**: Monitors source code changes and automatically rebuilds/redeploys
5. **Port forwarding**: Can automatically forward ports for easy testing

This eliminates the manual docker build → docker tag → kubectl apply cycle during development.

## Test the Application

Verify the deployment and test the application. With Skaffold's automatic port forwarding configured, you can directly test the application:

```bash
kubectl get pods
kubectl get services
```

These commands:
- `kubectl get pods`: Shows the running application pods to verify deployment
- `kubectl get services`: Lists services to see how the app is exposed

Since Skaffold is automatically forwarding port 42050 to localhost:8080, you can directly test the REST API:

```bash
curl http://localhost:8080/skaffold/api/v1.0/
```

This calls the REST endpoint defined in RoukouController, which returns a JSON response with a greeting message and request counter. Notice that you don't need to manually run `kubectl port-forward` - Skaffold handles this automatically.

```
{"request_id":1,"message":"skaffold demo=1"}
```



## Make Code Changes

Demonstrate Skaffold's hot reload capability by modifying the source code. In a new terminal, navigate to the project directory and edit the source code:

```bash
cd ~/k3d/skaffold/skaffold-demo
# Edit the controller file to change the greeting message
sed -i 's/skaffold demo=/Hello Skaffold! demo=/' src/main/java/org/roukou/skaffold/controller/RoukouController.java
```

This command changes the message returned by the REST API. Watch the Skaffold terminal - it should:
1. **Detect the file change** through its file watching mechanism
2. **Rebuild the Docker image** with the updated code
3. **Redeploy to Kubernetes** with the new image
4. **Stream new logs** showing the deployment process

**Note:** With Skaffold's automatic port forwarding, the connection will be maintained even when the pod restarts. You don't need to manually restart any port-forward commands.

Test the change by calling the API again to see the updated message:

```bash
curl http://localhost:8080/skaffold/api/v1.0/
```

You should now see: `{"request_id":1,"message":"Hello Skaffold! demo=1"}` instead of the original message.

## One-time Deployment

Stop the development mode (Ctrl+C) and try a one-time deployment for production-like scenarios:

```bash
skaffold run
```

Unlike `skaffold dev`, the `run` command:
- **Builds and deploys once** without file watching
- **Doesn't stream logs** continuously
- **Exits after deployment** instead of staying active
- **Suitable for CI/CD pipelines** and production deployments

## Debug Mode

Start in debug mode for troubleshooting application issues:

```bash
skaffold debug
```

Debug mode:
- **Enables JVM debug ports** for Java applications
- **Configures remote debugging** so you can attach IDE debuggers
- **Sets up port forwarding** for debug connections
- **Provides enhanced logging** for troubleshooting deployment issues
- **Allows breakpoint debugging** of your containerized application

## Clean Up

Clean up the resources when you're done:

```bash
skaffold delete
```

This command:
- **Removes all Kubernetes resources** that Skaffold deployed
- **Uses the same manifests** that were used for deployment
- **Ensures complete cleanup** of your application

Delete the entire k3d cluster to free up system resources:

```bash
k3d cluster delete skaffold
```

This removes the entire Kubernetes cluster and all its resources.

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