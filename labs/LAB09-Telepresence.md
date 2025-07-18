# Lab 09: Remote Development with Telepresence on EKS

In this lab, you will learn how to use Telepresence for remote development and debugging of microservices running on Amazon EKS (Elastic Kubernetes Service).

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, kubectl, and AWS CLI installed
- VS Code connected to Ubuntu VM via Remote Development extension
- AWS account with appropriate permissions for EKS
- Completed Lab 08: Tilt and k3d

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Development. All commands will be executed on the Ubuntu VM.

## Install Prerequisites

### Install eksctl
Install eksctl to manage EKS clusters:

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### Install AWS CLI v2 (if not already installed)
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Install Telepresence
```bash
sudo curl -fL https://app.getambassador.io/download/tel2/linux/amd64/latest/telepresence -o /usr/local/bin/telepresence
sudo chmod a+x /usr/local/bin/telepresence
```

Verify installations:
```bash
eksctl version
aws --version
kubectl version --client
telepresence version
```

## Configure AWS Credentials

Configure your AWS credentials (you'll need access key ID and secret access key):

```bash
aws configure
```

Or set environment variables:
```bash
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-west-2
```

## Create EKS Cluster

Create an EKS cluster using eksctl:

```bash
eksctl create cluster \
  --name telepresence-demo \
  --region us-west-2 \
  --nodegroup-name standard-workers \
  --node-type m5.large \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

This will take approximately 10-15 minutes to complete.

Verify the cluster is created:
```bash
kubectl get nodes
kubectl cluster-info
```

## Create Working Directory

```bash
mkdir -p ~/eks/telepresence
cd ~/eks/telepresence
```

## Deploy Sample Microservices Application

Create a sample microservices application to work with:

```bash
cat > microservices-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 2
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: frontend-config
          mountPath: /usr/share/nginx/html
      volumes:
      - name: frontend-config
        configMap:
          name: frontend-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Frontend Service</title>
    </head>
    <body>
        <h1>Frontend Service</h1>
        <p>This is the frontend service running on EKS</p>
        <p>Backend API: <a href="/api">Call Backend</a></p>
    </body>
    </html>
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
    port: 80
    targetPort: 80
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
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
        image: jruels/simple-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        - name: MESSAGE
          value: "Hello from EKS Backend!"
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
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  labels:
    app: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: database
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "appdb"
        - name: POSTGRES_USER
          value: "appuser"
        - name: POSTGRES_PASSWORD
          value: "apppass"
---
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  selector:
    app: database
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
EOF
```

Deploy the application:
```bash
kubectl apply -f microservices-app.yaml
```

Wait for pods to be ready:
```bash
kubectl get pods -w
```

## Test the Application

Get the frontend service external IP:
```bash
kubectl get service frontend
```

Wait for the EXTERNAL-IP to be assigned (this may take a few minutes with EKS Load Balancer).

Test the application:
```bash
# Get the external IP
FRONTEND_IP=$(kubectl get service frontend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Frontend URL: http://$FRONTEND_IP"

# Test the frontend
curl http://$FRONTEND_IP
```

## Connect Telepresence to EKS

Connect Telepresence to your EKS cluster:

```bash
telepresence connect
```

Verify the connection:
```bash
telepresence status
```

## Test Cluster Connectivity

Test that you can reach services directly from your local machine:

```bash
# Test backend service
curl http://backend:8080

# Test database connection (if you have psql installed)
# psql -h database -U appuser -d appdb
```

## Create Local Development Service

Create a local version of the backend service for development:

```bash
mkdir -p local-backend
cd local-backend
```

Create a simple Node.js backend service:

```bash
cat > package.json << 'EOF'
{
  "name": "local-backend",
  "version": "1.0.0",
  "description": "Local development backend",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.8.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.20"
  }
}
EOF
```

Create the server:

```bash
cat > server.js << 'EOF'
const express = require('express');
const { Client } = require('pg');

const app = express();
const port = process.env.PORT || 8080;

// Database connection
const dbClient = new Client({
  host: process.env.DB_HOST || 'database',
  port: process.env.DB_PORT || 5432,
  user: process.env.DB_USER || 'appuser',
  password: process.env.DB_PASSWORD || 'apppass',
  database: process.env.DB_NAME || 'appdb'
});

app.use(express.json());

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    message: 'Local development backend is running',
    timestamp: new Date().toISOString(),
    environment: 'local-development'
  });
});

// Main API endpoint
app.get('/api', (req, res) => {
  res.json({
    message: 'Hello from LOCAL DEVELOPMENT backend!',
    version: '2.0.0-dev',
    environment: 'local',
    timestamp: new Date().toISOString()
  });
});

// API endpoint that uses database
app.get('/api/data', async (req, res) => {
  try {
    await dbClient.connect();
    // Simple query to test database connectivity
    const result = await dbClient.query('SELECT NOW() as current_time');
    res.json({
      message: 'Database connected successfully',
      data: result.rows[0],
      source: 'local-development'
    });
  } catch (error) {
    res.status(500).json({
      error: 'Database connection failed',
      message: error.message,
      source: 'local-development'
    });
  } finally {
    await dbClient.end();
  }
});

// Route to test connectivity to other services
app.get('/api/test-connectivity', (req, res) => {
  res.json({
    message: 'Testing connectivity from local development',
    database_host: process.env.DB_HOST || 'database',
    backend_environment: 'local-development',
    can_reach_cluster: true
  });
});

app.listen(port, () => {
  console.log(`Local backend server running on port ${port}`);
  console.log(`Environment: local-development`);
});
EOF
```

Install dependencies:
```bash
npm install
```

## Create Telepresence Intercept

Create an intercept to route traffic from the EKS backend service to your local development service:

```bash
telepresence intercept backend --port 8080
```

The intercept will show you a preview URL and provide instructions for routing traffic.

## Test the Intercept

Start your local development server:
```bash
npm start
```

In a new terminal, test the intercept:
```bash
# Test via the frontend service
FRONTEND_IP=$(kubectl get service frontend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://$FRONTEND_IP/api

# Test directly
curl http://backend:8080/api
```

You should see responses from your local development service instead of the EKS backend.

## Test Database Connectivity

Test that your local service can connect to the database running in EKS:

```bash
curl http://backend:8080/api/data
curl http://backend:8080/api/test-connectivity
```

## Make Development Changes

Edit your local service to see immediate changes:

```bash
# Modify the API response
sed -i 's/Hello from LOCAL DEVELOPMENT backend!/Hello from UPDATED local backend!/' server.js

# Restart the service
npm start
```

Test the changes:
```bash
curl http://backend:8080/api
```

## Monitor Traffic

List active intercepts:
```bash
telepresence list
```

View detailed intercept status:
```bash
telepresence status
```

## Advanced: Conditional Intercepts

Create a conditional intercept that only routes specific traffic:

```bash
# First, leave the current intercept
telepresence leave backend

# Create a conditional intercept based on headers
telepresence intercept backend --port 8080 --http-header dev-user=local
```

Test with and without the header:
```bash
# This will go to the cluster
curl http://backend:8080/api

# This will go to your local service
curl -H "dev-user: local" http://backend:8080/api
```

## Clean Up Intercepts

Remove the intercept when done:
```bash
telepresence leave backend
```

## Test Normal Operation

Verify that traffic is now going to the original EKS backend:
```bash
curl http://backend:8080/api
```

## Disconnect Telepresence

When finished with development:
```bash
telepresence quit
```

## Clean Up EKS Resources

Clean up the application:
```bash
kubectl delete -f microservices-app.yaml
```

Delete the EKS cluster:
```bash
eksctl delete cluster --name telepresence-demo --region us-west-2
```

This will take several minutes to complete.

## Summary

In this lab, you learned how to:
- Create an EKS cluster using eksctl
- Deploy a microservices application to EKS
- Install and configure Telepresence for remote development
- Create intercepts to route EKS traffic to local development services
- Test connectivity between local services and EKS resources
- Use conditional intercepts for selective traffic routing
- Monitor and manage intercepts
- Clean up resources and restore normal cluster operation

## Key Benefits of Telepresence with EKS

1. **Realistic Development Environment**: Develop against real cloud services and databases
2. **Fast Iteration**: Make changes locally without rebuilding containers or redeploying
3. **Secure**: Traffic stays within your cluster's security boundaries
4. **Collaborative**: Multiple developers can work on different services simultaneously
5. **Cost-Effective**: Develop with minimal local resources while using cloud infrastructure

## Troubleshooting

### Common Issues:

1. **EKS cluster creation fails**
   - Check AWS credentials and permissions
   - Verify region availability and quotas
   - Ensure eksctl version is recent

2. **Telepresence connection issues**
   - Verify kubectl context: `kubectl config current-context`
   - Check cluster connectivity: `kubectl get nodes`
   - Restart Telepresence: `telepresence quit && telepresence connect`

3. **Intercept not working**
   - Verify service exists: `kubectl get svc backend`
   - Check port configuration matches service
   - Review intercept logs: `telepresence list`

4. **Load balancer not getting external IP**
   - Wait longer (can take 3-5 minutes)
   - Check AWS load balancer console
   - Verify security groups and subnets

5. **Local service can't reach cluster services**
   - Verify Telepresence is connected: `telepresence status`
   - Check service names and namespaces
   - Test DNS resolution: `nslookup database`

---

**Next Lab:** [LAB10-Advanced-Concepts.md](LAB10-Advanced-Concepts.md)