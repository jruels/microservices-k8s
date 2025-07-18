# Lab 04: Building a Helm Chart

In this lab, you will learn how to create and deploy a custom Helm chart for an Nginx application.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, k3d, kubectl, and Helm installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Completed Lab 03: K3d, Helm and Rancher

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension (see Lab 01 for SSH setup). All commands will be executed in the VS Code integrated terminal connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories and files.

## Create k3d Cluster

Create a k3d cluster for this lab:

```bash
k3d cluster create helm-demo --api-port 6553 --servers 1 --agents 2 --port "80:80@loadbalancer" --wait
```

Verify the cluster is running:

```bash
k3d cluster list
kubectl cluster-info
```

## Create a Helm Chart

Create a new Helm chart called `nginx-chart`. First, create the directory structure:

**Option 1: Using VS Code File Explorer**
- In VS Code's file explorer, navigate to `/home/ubuntu`
- Right-click and create a new folder called `k3d`
- Inside `k3d`, create a new folder called `helm-charts`
- Navigate to this directory in the terminal

**Option 2: Using Terminal**
```bash
mkdir -p ~/k3d/helm-charts
cd ~/k3d/helm-charts
```

Then create the Helm chart:
```bash
helm create nginx-chart
```

Examine the chart structure:

```bash
ls -la nginx-chart/
```

You should see:
- `Chart.yaml` - Chart metadata
- `values.yaml` - Default configuration values
- `templates/` - Kubernetes manifest templates
- `charts/` - Dependency charts (empty initially)

## Examine Chart Structure

You can explore the chart structure using VS Code's file explorer by navigating to the `nginx-chart` folder, or use the terminal commands below:

### Chart.yaml
Look at the chart metadata using VS Code (double-click the file in explorer) or terminal:

```bash
cat nginx-chart/Chart.yaml
```

### Values.yaml
Examine the default values using VS Code (double-click the file in explorer) or terminal:

```bash
cat nginx-chart/values.yaml
```

### Templates Directory
View the template files in VS Code file explorer or using terminal:

```bash
ls -la nginx-chart/templates/
```

## Customize the Chart

Edit the values.yaml file to customize the deployment. You can either:
- **Use VS Code**: Open `nginx-chart/values.yaml` in VS Code editor and replace the contents with the configuration below
- **Use terminal**: Run the command below to replace the file contents

```bash
cat > nginx-chart/values.yaml << 'EOF'
# Default values for nginx-chart
replicaCount: 2

image:
  repository: nginx
  pullPolicy: Always
  tag: "latest"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # Annotations to add to the service account
  annotations: {}
  # The name of the service account to use.
  name: "nginxrocks"

podAnnotations: {}

podSecurityContext: {}

securityContext: {}

service:
  type: NodePort
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

## Validate the Chart

Before deploying, validate the chart syntax:

```bash
helm lint nginx-chart/
```

## Deploy the Chart

Deploy the chart to your cluster:

```bash
helm install my-nginx-app nginx-chart/
```

**Note**: The `values.yaml` file is automatically used by default, so specifying `--values nginx-chart/values.yaml` is redundant.

You should see output similar to:
```
NAME: my-nginx-app
LAST DEPLOYED: [timestamp]
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-nginx-app)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
```

## Verify the Deployment

Check the deployed resources:

```bash
# View Helm releases
helm list

# Check pods
kubectl get pods -l app.kubernetes.io/name=nginx-chart

# Check services
kubectl get services -l app.kubernetes.io/name=nginx-chart

# Check deployments
kubectl get deployments -l app.kubernetes.io/name=nginx-chart
```

## Test the Application

Set up port forwarding to test the application:

```bash
export POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=nginx-chart,app.kubernetes.io/instance=my-nginx-app" -o jsonpath="{.items[0].metadata.name}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl port-forward $POD_NAME 8080:80
```

In a new terminal, test the application:

```bash
curl http://127.0.0.1:8080
```

You should see the Nginx welcome page HTML.

## Upgrade the Chart

Make a change to the values and upgrade:

```bash
# Edit values.yaml to change replica count
sed -i 's/replicaCount: 2/replicaCount: 3/' nginx-chart/values.yaml

# Upgrade the release
helm upgrade my-nginx-app nginx-chart/
```

Verify the upgrade:

```bash
kubectl get pods -l app.kubernetes.io/name=nginx-chart
```

You should now see 3 pods running.

## View Helm History

Check the release history:

```bash
helm history my-nginx-app
```

## Package the Chart

Package the chart for distribution:

```bash
helm package nginx-chart/
```

This creates a `.tgz` file that can be shared or stored in a chart repository.

## Template Debugging

Debug and view the rendered templates:

```bash
helm template my-nginx-app nginx-chart/
```

This shows you the exact Kubernetes manifests that Helm would generate without actually deploying them.

## Rollback Example

If you need to rollback to a previous version:

```bash
# View release history
helm history my-nginx-app

# Rollback to previous version (revision 1)
helm rollback my-nginx-app 1

# Verify the rollback
kubectl get pods
```

## Clean Up

When you're done with the lab:

```bash
# Uninstall the Helm release
helm uninstall my-nginx-app

# Delete the cluster
k3d cluster delete helm-demo
```

## Summary

In this lab, you learned how to:
- Create a new Helm chart from scratch
- Understand the structure of a Helm chart
- Customize chart values to configure deployments
- Deploy applications using Helm
- Upgrade and rollback Helm releases
- Package charts for distribution
- Debug template rendering
- Clean up resources

Helm charts provide a powerful way to manage complex Kubernetes applications with templating, versioning, and easy deployment management.

---

## Troubleshooting

### Common Issues:

1. **Chart validation errors**
   - Run `helm lint nginx-chart/` to check for syntax errors
   - Verify indentation in YAML files

2. **Port forwarding issues**
   - Check if port 8080 is already in use: `netstat -tlnp | grep 8080`
   - Use a different port: `kubectl port-forward $POD_NAME 8081:80`

3. **Image pull errors**
   - Ensure the nginx image is available
   - Check image tag and repository settings

4. **Service connection issues**
   - Verify service selector matches pod labels
   - Check if pods are running: `kubectl get pods -l app.kubernetes.io/name=nginx-chart`

---

**Next Lab:** [LAB05-OperatorSDK-Helm.md](LAB05-OperatorSDK-Helm.md)