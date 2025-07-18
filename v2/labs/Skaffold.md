# Using Skaffold to speed up development

In this lab, you will use Skaffold to optimize application development and deployment on Kubernetes.

## Prerequisites
- macOS or Ubuntu VM with Docker and k3d installed
- VS Code
- kubectl installed

## Step 1: Install Skaffold
```bash
brew install skaffold  # On macOS
# For Linux, see https://skaffold.dev/docs/install/
```

## Step 2: Create a Working Directory
```bash
mkdir -p ~/k3d/skaffold
cd ~/k3d/skaffold
```

## Step 3: Create the k3d Cluster
```bash
k3d cluster create skaffold --api-port 6550 --agents 2
```

## Step 4: Clone the Demo Application
```bash
git clone https://github.com/jruels/skaffold-demo.git
cd skaffold-demo
```

## Step 5: Add a Dockerfile
Create a `Dockerfile` in the project root:
```Dockerfile
FROM maven:3-jdk-11 as BUILD
COPY . /usr/src/app
RUN mvn --batch-mode -f /usr/src/app/pom.xml clean package
FROM openjdk:11-jre-slim
ENV PORT 42050
EXPOSE 42050
COPY --from=BUILD /usr/src/app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Step 6: Build and Deploy with Skaffold
```bash
skaffold dev
```

## Step 7: Test the Application
Access the endpoint using `kubectl port-forward` or via the cluster ingress.

## Cleanup
```bash
k3d cluster delete skaffold
```

## Next Steps
Proceed to the Tilt lab for automated development workflows.

