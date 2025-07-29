# Lab 05: Working with Operator SDK and Helm

In this lab, you will learn how to use the Operator SDK to create a Kubernetes operator using Helm charts.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, k3d, kubectl, and Helm installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Completed Lab 04: Building a Helm Chart
- Go programming language installed (we'll install this)

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension (see Lab 01 for SSH setup). All commands will be executed in the VS Code integrated terminal connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories and files.

## Install Prerequisites

Install Go (if not already installed):

```bash
# Install Go
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

Install Operator SDK:

```bash
# Install Operator SDK
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.32.0
curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
chmod +x operator-sdk_${OS}_${ARCH}
sudo mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk
```

Install Kustomize:

```bash
# Install Kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

Verify installations:

```bash
go version
operator-sdk version
kustomize version
```

## Create k3d Cluster

Create a k3d cluster for this lab:

```bash
k3d cluster create operator-demo --api-port 6554 --servers 1 --agents 2 --port "80:80@loadbalancer" --wait
```

## Create Operator Project

Create a new directory for the operator project using VS Code file explorer or terminal:

**Option 1: Using VS Code File Explorer**
- In VS Code's file explorer, navigate to `/home/ubuntu/k3d` (create `k3d` folder if it doesn't exist)
- Right-click and create a new folder called `operators`
- Navigate to this directory in the terminal

**Option 2: Using Terminal**
```bash
mkdir -p ~/k3d/operators
cd ~/k3d/operators
```

Initialize a new Helm-based operator:

```bash
operator-sdk init --plugins=helm --domain=example.com --group=webapp --version=v1 --kind=Nginx
```

This creates the operator project structure with:
- `Dockerfile` for building the operator image
- `Makefile` for building and deploying
- `config/` directory with Kubernetes manifests
- `helm-charts/` directory for the Helm chart

## Examine the Project Structure

Explore the generated files:

```bash
ls -la
tree . -L 2
```

Key files and directories:
- `PROJECT` - Project configuration
- `Makefile` - Build and deployment commands
- `config/` - Kubernetes manifests for the operator
- `helm-charts/nginx/` - Helm chart for the managed application

## Customize the Helm Chart

Navigate to the Helm chart directory and examine it:

```bash
cd helm-charts/nginx
ls -la
```

Edit the values.yaml to customize the Nginx deployment. You can either:
- **Use VS Code**: Open `values.yaml` in VS Code editor and replace the contents with the configuration below
- **Use terminal**: Run the command below to replace the file contents

```bash
cat > values.yaml << 'EOF'
# Default values for nginx.
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "latest"

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 100m
    memory: 128Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}

tolerations: []

affinity: {}
EOF
```

## Build and Deploy the Operator

Return to the project root:

```bash
cd ~/k3d/operators
```

Build the operator:

```bash
make docker-build IMG=nginx-operator:latest
```

Load the image into k3d:

```bash
k3d image import nginx-operator:latest -c operator-demo
```

Deploy the operator (using local image without pushing to registry):

```bash
make deploy IMG=nginx-operator:latest REGISTRY_PUSH=false
```

If you encounter ImagePullBackOff errors, patch the deployment to use the local image:

```bash
kubectl patch deployment operators-controller-manager -n operators-system -p '{"spec":{"template":{"spec":{"containers":[{"name":"manager","imagePullPolicy":"Never"}]}}}}'
```

## Verify Operator Installation

Check that the operator is running:

```bash
kubectl get pods -n operators-system
```

View the Custom Resource Definitions (CRDs):

```bash
kubectl get crd
```

You should see `nginxes.webapp.example.com` in the list.

## Create a Custom Resource

Create a sample Nginx instance using the custom resource:

```bash
cat > nginx-sample.yaml << 'EOF'
apiVersion: webapp.example.com/v1
kind: Nginx
metadata:
  name: nginx-sample
spec:
  replicaCount: 3
  image:
    repository: nginx
    tag: "latest"
  service:
    type: NodePort
    port: 80
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi
EOF
```

Apply the custom resource:

```bash
kubectl apply -f nginx-sample.yaml
```

## Verify the Deployment

Check the custom resource:

```bash
kubectl get nginx
kubectl describe nginx nginx-sample
```

Verify the created resources:

```bash
kubectl get pods
kubectl get services
kubectl get deployments
```

## Test the Application

Port forward to test the Nginx application:

```bash
kubectl port-forward service/nginx-sample 8080:80
```

In a new terminal, test the application:

```bash
curl http://127.0.0.1:8080
```

## Modify the Custom Resource

Update the replica count:

```bash
kubectl patch nginx nginx-sample --type='merge' -p='{"spec":{"replicaCount":5}}'
```

Watch the pods scale:

```bash
kubectl get pods -w
```

## View Operator Logs

Check the operator logs:

```bash
kubectl logs -n operators-system -l control-plane=controller-manager -f
```

## Create Another Instance

Create a second Nginx instance:

```bash
cat > nginx-sample2.yaml << 'EOF'
apiVersion: webapp.example.com/v1
kind: Nginx
metadata:
  name: nginx-sample2
spec:
  replicaCount: 2
  image:
    repository: nginx
    tag: "alpine"
  service:
    type: ClusterIP
    port: 80
EOF
```

Apply it:

```bash
kubectl apply -f nginx-sample2.yaml
```

Verify both instances are running:

```bash
kubectl get nginx
kubectl get pods
```

## Clean Up

Delete the custom resources:

```bash
kubectl delete nginx nginx-sample nginx-sample2
```

Uninstall the operator:

```bash
make undeploy
```

Delete the cluster:

```bash
k3d cluster delete operator-demo
```

## Summary

In this lab, you learned how to:
- Install and configure the Operator SDK
- Create a Helm-based Kubernetes operator
- Customize the underlying Helm chart
- Build and deploy the operator to a k3d cluster
- Create and manage custom resources
- Scale applications through custom resource modifications
- Monitor operator behavior through logs
- Clean up operator resources

The Operator SDK provides a powerful framework for creating operators that can manage complex applications on Kubernetes using familiar Helm charts while adding custom logic and automation.

---

## Troubleshooting

### Common Issues:

1. **Operator SDK build errors**
   - Ensure Go is properly installed and GOPATH is set
   - Check Docker daemon is running: `docker ps`
   - Verify operator-sdk version compatibility

2. **Image loading issues**
   - Confirm image was built successfully: `docker images | grep nginx-operator`
   - Check k3d cluster name: `k3d cluster list`

3. **CRD not found**
   - Verify CRDs are installed: `kubectl get crd | grep nginx`
   - Check operator deployment: `kubectl get pods -n nginx-operator-system`

4. **Custom resource not working**
   - Check operator logs: `kubectl logs -n nginx-operator-system -l control-plane=controller-manager`
   - Verify custom resource syntax: `kubectl describe nginx nginx-sample`

---

**Next Lab:** [LAB06-Skaffold.md](LAB06-Skaffold.md)