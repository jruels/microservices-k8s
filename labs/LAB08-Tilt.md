# Lab 08: Tilt and k3d

In this lab, you will learn how to use Tilt for fast, multi-service development on Kubernetes.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, k3d, and kubectl installed
- VS Code connected to Ubuntu VM via Remote Development extension

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension (see Lab 01 for SSH setup). All commands will be executed in the VS Code integrated terminal connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories and files.

## Install Tilt

Install Tilt on your Ubuntu VM:

```bash
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash
```

Verify installation:

```bash
tilt version
```

## Create k3d Cluster

Create a k3d cluster for this lab:

```bash
k3d cluster create tilt --api-port 6557 --agents 2 --wait
```

## Create Working Directory

Create the working directory using VS Code file explorer or terminal:

**Option 1: Using VS Code File Explorer**
- In VS Code's file explorer, navigate to `/home/ubuntu/k3d` (create `k3d` folder if it doesn't exist)
- Right-click and create a new folder called `tilt`
- Navigate to this directory in the terminal

**Option 2: Using Terminal**
```bash
mkdir -p ~/k3d/tilt
cd ~/k3d/tilt
```

## Try Tilt Demo

First, let's try Tilt's built-in demo:

```bash
tilt demo
```

This creates a temporary demo project and starts Tilt. Access the Tilt UI at http://localhost:10350

Stop the demo (Ctrl+C) and clean up:

```bash
tilt down
```

## Create Multi-Service Application

Create a multi-service application structure using VS Code file explorer or terminal:

**Option 1: Using VS Code File Explorer**
- In your current `tilt` directory, right-click and create folders named `frontend` and `backend`

**Option 2: Using Terminal**
```bash
mkdir -p frontend backend
```

## Create Backend Service

Create a simple backend service:

```bash
cat > backend/app.py << 'EOF'
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/api/hello')
def hello():
    return jsonify({"message": "Hello from backend!", "version": "1.0"})

@app.route('/api/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

Create backend Dockerfile:

```bash
cat > backend/Dockerfile << 'EOF'
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

Create backend requirements:

```bash
cat > backend/requirements.txt << 'EOF'
Flask==2.0.1
EOF
```

## Create Frontend Service

Create a simple frontend service:

```bash
cat > frontend/app.py << 'EOF'
from flask import Flask, render_template_string
import requests
import os

app = Flask(__name__)

BACKEND_URL = os.getenv('BACKEND_URL', 'http://backend:5000')

@app.route('/')
def index():
    try:
        response = requests.get(f'{BACKEND_URL}/api/hello')
        data = response.json()
        message = data.get('message', 'No message')
    except:
        message = 'Backend not available'
    
    return render_template_string('''
    <html>
    <head><title>Frontend</title></head>
    <body>
        <h1>Frontend Service</h1>
        <p>Backend says: {{ message }}</p>
        <p><a href="/health">Health Check</a></p>
    </body>
    </html>
    ''', message=message)

@app.route('/health')
def health():
    return "Frontend is healthy"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF
```

Create frontend Dockerfile:

```bash
cat > frontend/Dockerfile << 'EOF'
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python", "app.py"]
EOF
```

Create frontend requirements:

```bash
cat > frontend/requirements.txt << 'EOF'
Flask==2.0.1
requests==2.25.1
EOF
```

## Create Kubernetes Manifests

Create backend Kubernetes manifest:

```bash
cat > backend/k8s.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: backend
        ports:
        - containerPort: 5000
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
EOF
```

Create frontend Kubernetes manifest:

```bash
cat > frontend/k8s.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: frontend
        ports:
        - containerPort: 8080
        env:
        - name: BACKEND_URL
          value: "http://backend:5000"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 8080
  type: NodePort
EOF
```

## Create Tiltfile

Create a Tiltfile to define the development workflow:

```bash
cat > Tiltfile << 'EOF'
# Load extension for HTML display
load('ext://html', 'html')

# Backend service
docker_build('backend', 'backend')
k8s_yaml('backend/k8s.yaml')
k8s_resource('backend', port_forwards=5000)

# Frontend service
docker_build('frontend', 'frontend')
k8s_yaml('frontend/k8s.yaml')
k8s_resource('frontend', port_forwards=8080)

# Local resource for info display
local_resource('welcome', 'echo "Welcome to Tilt multi-service demo!"')

# Add buttons for easy access
html('frontend_link', '<a href="http://localhost:8080" target="_blank">Open Frontend</a>')
html('backend_link', '<a href="http://localhost:5000/api/hello" target="_blank">Backend API</a>')
EOF
```

## Start Tilt

Start Tilt development environment:

```bash
tilt up
```

This will:
1. Build Docker images for both services
2. Deploy to k3d cluster
3. Set up port forwarding
4. Open the Tilt UI at http://localhost:10350

## Explore Tilt UI

Access the Tilt UI at http://localhost:10350 to see:
- Resource status
- Build logs
- Runtime logs
- Port forwarding status
- Resource actions

## Test the Application

Test the frontend:

```bash
curl http://localhost:8080
```

Test the backend API:

```bash
curl http://localhost:5000/api/hello
```

## Make Changes and Watch Auto-Rebuild

Edit the backend service:

```bash
sed -i 's/Hello from backend!/Hello from UPDATED backend!/' backend/app.py
```

Watch the Tilt UI - it should automatically rebuild and redeploy the backend service.

## View Logs

In the Tilt UI, click on the services to view their logs in real-time.

## Add More Services

You can add more services by creating additional directories and updating the Tiltfile.

## Clean Up

Stop Tilt:

```bash
tilt down
```

Delete the cluster:

```bash
k3d cluster delete tilt
```

## Summary

In this lab, you learned how to:
- Install and configure Tilt
- Create multi-service applications
- Use Tiltfile to define development workflows
- Leverage Tilt's UI for monitoring and debugging
- Implement automatic rebuilds on file changes
- Set up port forwarding for easy testing
- Clean up resources

Tilt provides an excellent development experience for multi-service applications with its intuitive UI and fast rebuild capabilities.

---

## Troubleshooting

### Common Issues:

1. **Tilt installation issues**
   - Ensure curl is available: `which curl`
   - Check installation script permissions
   - Verify Tilt binary location: `which tilt`

2. **Docker build failures**
   - Ensure Docker daemon is running: `docker ps`
   - Check Dockerfile syntax and build context
   - Verify Python/Flask dependencies are correct

3. **Port forwarding conflicts**
   - Check if ports are already in use: `netstat -tlnp | grep 8080`
   - Modify port forwarding in Tiltfile if needed
   - Use different local ports: `k8s_resource('frontend', port_forwards=8081)`

4. **Service communication issues**
   - Verify service names match in frontend app: `BACKEND_URL`
   - Check pod logs: `kubectl logs -l app=backend`
   - Ensure services are running: `kubectl get svc`

5. **Tilt UI not accessible**
   - Check if port 10350 is available: `netstat -tlnp | grep 10350`
   - Verify Tilt is running: `tilt status`
   - Try accessing via localhost instead of 127.0.0.1

---

**Next Lab:** [LAB09-Telepresence.md](LAB09-Telepresence.md)