# How to Debug Kubernetes Services With VS Code and Telepresence

Use VS Code and Telepresence connected to a remote Kubernetes cluster to locally debug Node.js microservices.

## Prerequisites
- Kubernetes cluster (remote or local k3d)
- Node.js (latest version)
- VS Code
- Telepresence installed ([Install Guide](https://www.telepresence.io/docs/latest/install/))

## Step 1: Deploy a Sample Microservice Application
```bash
kubectl apply -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml
kubectl get service
```

## Step 2: Set Up Local Node.js Development
```bash
mkdir -p ~/k3d/telepresence
cd ~/k3d/telepresence
git clone https://github.com/datawire/edgey-corp-nodejs.git
cd edgey-corp-nodejs/DataProcessingService
npm install
```

## Step 3: Start Telepresence for Local Debugging
```bash
telepresence connect
telepresence intercept dataprocessingservice --port 3000:http
```

- This will route traffic for `dataprocessingservice` to your local machine.
- You can now debug locally using VS Code.

## Step 4: Debug in VS Code
- Set breakpoints and use the VS Code debugger as usual.
- Requests to the service in Kubernetes will be routed to your local code.

## Cleanup
```bash
telepresence leave dataprocessingservice
kubectl delete -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml
```

## Next Steps
Explore more advanced Telepresence features or move to the Skaffold Microservices example.





