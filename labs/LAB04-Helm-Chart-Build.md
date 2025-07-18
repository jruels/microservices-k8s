# Lab 04: Building a Helm Chart

In this lab, you will learn how to create and deploy a custom Helm chart for an Nginx application.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, k3d, kubectl, and Helm installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Completed Lab 03: K3d, Helm and Rancher

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Development. All commands will be executed on the Ubuntu VM.

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

Create a new Helm chart called `nginx-chart`:

```bash
mkdir -p ~/k3d/helm-charts
cd ~/k3d/helm-charts
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

### Chart.yaml
Look at the chart metadata:

```bash
cat nginx-chart/Chart.yaml
```

### Values.yaml
Examine the default values:

```bash
cat nginx-chart/values.yaml
```

### Templates Directory
View the template files:

```bash
ls -la nginx-chart/templates/
```

## Customize the Chart

Edit the values.yaml file to customize the deployment:

```bash
cat > nginx-chart/values.yaml << 'EOF'
# Default values for nginx-chart
replicaCount: 2

image:
  repository: nginx
  pullPolicy: Always
  tag: "latest"

imagePullSecrets: []
nameOverride: "nginx-awesome-app"
fullnameOverride: "nginx-chart"

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
helm install my-nginx-app nginx-chart/ --values nginx-chart/values.yaml
```

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
helm upgrade my-nginx-app nginx-chart/ --values nginx-chart/values.yaml
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
helm template my-nginx-app nginx-chart/ --values nginx-chart/values.yaml
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