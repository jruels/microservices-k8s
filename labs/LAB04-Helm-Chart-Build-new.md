# Comprehensive Helm Lab Guide

This comprehensive lab will teach you how to use Helm effectively, from basic installation to advanced features like chart creation, debugging, deployment strategies, and rollbacks. By the end of this lab, you'll have hands-on experience with all major Helm concepts.

## Prerequisites
- Windows VM with VS Code, Git Bash, and Remote Development extension
- Ubuntu VM with Docker, k3d, kubectl, and Helm installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Basic understanding of Kubernetes concepts

## Environment Setup
**IMPORTANT**: Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension. All commands in this lab will be executed in the VS Code integrated terminal (using Git Bash) connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories and files when creating or editing chart files.

**Important Network Setup Notes**: 
- The k3d cluster runs on the Ubuntu VM with kubeconfig only available there
- All `kubectl` commands execute on the Ubuntu VM through your VS Code Remote connection
- Application testing is done using `curl` commands from within the Ubuntu VM

## Lab 1: Helm Fundamentals

### Create k3d Cluster

First, clean up any existing cluster and create a fresh k3d cluster for this lab:

```bash
# Clean up any existing cluster with the same name
k3d cluster delete helm-lab 2>/dev/null || true

# Create a new cluster
k3d cluster create helm-lab --api-port 6555 --servers 1 --agents 2 --port "80:80@loadbalancer" --wait
```

Verify the cluster is running:

```bash
k3d cluster list
kubectl cluster-info
```

### Install Helm

First, confirm if `helm` is installed:
```bash
helm version
```

You should see output showing a Helm 3 version:
```
version.BuildInfo{Version:"v3.18.1", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.21.0"}
```

If Helm is not installed, install it on your Ubuntu VM:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Understanding Helm

Helm simplifies installing, managing, updating, and removing applications on your Kubernetes cluster using "charts" - packages of pre-configured Kubernetes resources.

### Explore the Helm Hub

Search for available charts:
```bash
helm search hub nginx
helm search hub wordpress
helm search hub prometheus
```

### Add and Use a Chart Repository

Add the official Bitnami repository:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Search for charts in the repository:
```bash
helm search repo bitnami/nginx
```

## Lab 2: Creating and Customizing Charts

### Create Your First Chart

First, create a working directory for this lab. You can either:
- **Use VS Code File Explorer**: Navigate to `/home/ubuntu` and create a new folder called `helm-labs`
- **Use Terminal**: Run the command below

```bash
mkdir -p ~/helm-labs
cd ~/helm-labs
```

Create a new chart called `webapp`:
```bash
helm create webapp
```

Examine the chart structure using VS Code file explorer by navigating to the `webapp` folder, or use terminal:
```bash
ls -la webapp/
```

You'll see:
```
webapp/
├── charts/              # Dependencies
├── Chart.yaml          # Chart metadata
├── templates/           # Kubernetes manifest templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests/
│       └── test-connection.yaml
└── values.yaml         # Default configuration values
```

### Understanding Key Files

**Chart.yaml** contains chart metadata. You can examine it using VS Code (double-click the file in explorer) or terminal:
```bash
cat webapp/Chart.yaml
```

**values.yaml** contains default configuration. View it using VS Code (double-click the file in explorer) or terminal:
```bash
cat webapp/values.yaml
```

**Templates** contain Kubernetes manifests with Go template syntax. Examine them using VS Code file explorer or terminal:
```bash
cat webapp/templates/deployment.yaml
```

### Deploy the Default Chart

Deploy the chart (it creates an nginx deployment by default):
```bash
helm install webapp ./webapp
```

Verify the deployment:
```bash
helm list
kubectl get pods -l app.kubernetes.io/name=webapp
kubectl get svc -l app.kubernetes.io/name=webapp
```

Test the application using port-forward:
```bash
kubectl port-forward service/webapp 8080:80
```

Open another VS Code terminal and test:
```bash
curl http://localhost:8080
```

You should see the default nginx welcome page HTML.

### Customize the Chart

Update `webapp/Chart.yaml`. You can either:
- **Use VS Code**: Open `webapp/Chart.yaml` in VS Code editor and replace the contents with the configuration below
- **Use terminal**: Run the command below to replace the file contents

```bash
cat > webapp/Chart.yaml << 'EOF'
apiVersion: v2
name: webapp
description: A comprehensive demo web application
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: Your Name
    email: your.email@example.com
EOF
```

Update `webapp/values.yaml` for a custom application. You can either:
- **Use VS Code**: Open `webapp/values.yaml` in VS Code editor and replace the contents with the configuration below
- **Use terminal**: Run the command below to replace the file contents

```bash
cat > webapp/values.yaml << 'EOF'
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "alpine"

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

podSecurityContext: {}

securityContext: {}

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: webapp.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 200m
    memory: 256Mi
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

## Lab 3: Advanced Chart Features

**Ensure you're in the correct directory:**
```bash
cd ~/helm-labs
```

### Working with Template Functions

Create a custom template file `webapp/templates/configmap.yaml`. You can either:
- **Use VS Code**: Create a new file `configmap.yaml` in the `webapp/templates/` folder using VS Code file explorer and add the content below
- **Use terminal**: Run the command below to create the file

```bash
cat > webapp/templates/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "webapp.fullname" . }}-config
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
data:
  app.properties: |
    environment={{ .Values.environment | default "development" }}
    debug={{ .Values.debug | default "false" }}
    replicas={{ .Values.replicaCount }}
    version={{ .Chart.AppVersion }}
EOF
```

Add these values to `values.yaml`. You can either:
- **Use VS Code**: Open `webapp/values.yaml` in VS Code and add the lines below to the end of the file
- **Use terminal**: Run the command below to append the values

```bash
cat >> webapp/values.yaml << 'EOF'
environment: "production"
debug: "true"
EOF
```

Now upgrade the release to apply the new ConfigMap:
```bash
helm upgrade webapp webapp/
```

Verify the ConfigMap was created:
```bash
kubectl get configmap
kubectl describe configmap webapp-config
```

### Add Conditional Templates

Update `webapp/templates/deployment.yaml` to include the ConfigMap environment variables. 

**Using a safe automated approach** - Run this script to add the environment section:

```bash
# Create a backup first
cp webapp/templates/deployment.yaml webapp/templates/deployment.yaml.backup

# Use sed to add environment variables after the image line
sed -i '/image: /a\
        env:\
        - name: APP_ENV\
          valueFrom:\
            configMapKeyRef:\
              name: {{ include "webapp.fullname" . }}-config\
              key: app.properties' webapp/templates/deployment.yaml
```

**Alternative VS Code approach**: If you prefer manual editing, add these lines under the `image:` line in the containers section:
```yaml
        env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: {{ include "webapp.fullname" . }}-config
              key: app.properties
```

Verify the change was applied correctly:
```bash
grep -A 6 "env:" webapp/templates/deployment.yaml
```

Upgrade the release to apply the template changes:
```bash
helm upgrade webapp webapp/
```

### Template Helpers

Examine the helper functions using VS Code (double-click the file in explorer) or terminal:
```bash
cat webapp/templates/_helpers.tpl
```

Add a custom helper. You can either:
- **Use VS Code**: Open `webapp/templates/_helpers.tpl` in VS Code and add the helper function below to the end of the file
- **Use terminal**: Run the command below to append the helper

```bash
cat >> webapp/templates/_helpers.tpl << 'EOF'

{{/*
Create environment-specific labels
*/}}
{{- define "webapp.envLabels" -}}
environment: {{ .Values.environment | default "development" }}
{{- end }}
EOF
```

## Lab 4: Debugging and Troubleshooting

**Ensure you're in the correct directory:**
```bash
cd ~/helm-labs
```

### Lint Your Chart

Check for syntax errors and best practices:
```bash
helm lint webapp/
```

Fix any issues reported by the linter.

### Dry Run and Template Debugging

See what Kubernetes manifests would be generated:
```bash
helm template webapp webapp/
```

Test the installation without actually deploying:
```bash
helm install webapp webapp/ --dry-run --debug
```

### Validate Against Kubernetes

Test if the manifests are valid for your cluster:
```bash
helm template webapp webapp/ | kubectl apply --dry-run=client -f -
```

### Debug Template Issues

If you have template errors, use `--debug` to see the rendered templates:
```bash
helm install webapp webapp/ --debug
```

## Lab 5: Deployment Strategies and Lifecycle Management

**Ensure you're in the correct directory:**
```bash
cd ~/helm-labs
```

### Package Your Chart

Create a distributable package:
```bash
helm package webapp/
ls -la webapp-*.tgz
```

### Install from Package

```bash
helm install packaged-webapp webapp-0.1.0.tgz
```

### Upgrade Strategies

Update the replica count in `values.yaml`:
```yaml
replicaCount: 3
```

Update the chart version in `Chart.yaml`:
```yaml
version: 0.2.0
```

Upgrade the release:
```bash
helm upgrade webapp webapp/
```

### Override Values During Install/Upgrade

Override values from command line:
```bash
helm upgrade webapp webapp/ --set replicaCount=5 --set image.tag=latest
```

Use a custom values file:
```bash
cat > custom-values.yaml << EOF
replicaCount: 4
environment: "staging"
resources:
  limits:
    cpu: 300m
    memory: 512Mi
EOF

helm upgrade webapp webapp/ -f custom-values.yaml
```

### Release History and Rollbacks

View release history:
```bash
helm history webapp
```

Get detailed information about a specific revision:
```bash
helm get values webapp --revision 1
helm get manifest webapp --revision 1
```

Rollback to a previous revision:
```bash
helm rollback webapp 1
```

Verify the rollback:
```bash
kubectl get pods
helm history webapp
```

### Status and Information Commands

Check release status:
```bash
helm status webapp
```

Get all values for the release:
```bash
helm get values webapp
```

Get the complete manifest:
```bash
helm get manifest webapp
```

### Uninstall Releases

Remove a release:
```bash
helm uninstall webapp
```

Keep release history:
```bash
helm uninstall webapp --keep-history
```

## Lab 6: Working with Dependencies and Repositories

**Ensure you're in the correct directory:**
```bash
cd ~/helm-labs
```

### Add Chart Dependencies

Update `webapp/Chart.yaml` to include dependencies:
```yaml
dependencies:
  - name: redis
    version: "17.3.7"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
  - name: postgresql
    version: "12.1.2"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
```

Update `values.yaml` to configure dependencies:
```yaml
redis:
  enabled: true
  auth:
    enabled: false

postgresql:
  enabled: false
  auth:
    username: webapp
    password: secretpassword
    database: webapp_db
```

Download dependencies:
```bash
cd webapp/
helm dependency update
cd ..
```

Deploy with dependencies:
```bash
helm install webapp-with-deps webapp/
```

### Working with Private Repositories

Add a private repository (example):
```bash
helm repo add mycompany https://charts.mycompany.com --username user --password pass
```

## Advanced Features and Best Practices

### Using Helm Hooks

Create a pre-install hook `webapp/templates/pre-install-job.yaml`:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "webapp.fullname" . }}-pre-install
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      name: {{ include "webapp.fullname" . }}-pre-install
    spec:
      restartPolicy: Never
      containers:
      - name: pre-install
        image: busybox
        command:
        - /bin/sh
        - -c
        - echo "Running pre-install setup..."
```

### Chart Testing

Create tests in `webapp/templates/tests/`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "webapp.fullname" . }}-test"
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "webapp.fullname" . }}:{{ .Values.service.port }}']
```

Run tests:
```bash
helm test webapp
```

### Security Best Practices

Use `--verify` with signed charts:
```bash
helm install webapp webapp/ --verify
```

Check for security issues:
```bash
helm lint webapp/ --strict
```

### Performance and Monitoring

Monitor Helm operations:
```bash
helm list --all-namespaces
helm history webapp --max 10
```

## Troubleshooting Common Issues

### Chart Installation Failures

Debug failed installations:
```bash
helm status webapp
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### Template Rendering Issues

Check template syntax:
```bash
helm template webapp webapp/ | less
```

### Resource Conflicts

Check for existing resources:
```bash
kubectl get all -l app.kubernetes.io/instance=webapp
```

### Permission Issues

Verify service account permissions:
```bash
kubectl auth can-i create deployments --as=system:serviceaccount:default:webapp
```

## Cleanup

Remove all resources:
```bash
helm uninstall webapp
helm uninstall webapp-with-deps
helm uninstall packaged-webapp
```

Remove repositories:
```bash
helm repo remove bitnami
```

Delete the k3d cluster:
```bash
k3d cluster delete helm-lab
```

## Summary

In this comprehensive lab, you learned:

1. **Helm Fundamentals**: Installation, basic concepts, and chart exploration
2. **Chart Creation**: Building custom charts from scratch
3. **Advanced Features**: Templates, helpers, dependencies, and hooks
4. **Debugging**: Linting, dry-runs, and troubleshooting techniques
5. **Deployment Strategies**: Install, upgrade, rollback, and lifecycle management
6. **Best Practices**: Security, testing, and monitoring

### Key Commands Reference

```bash
# Chart Management
helm create <chart-name>
helm package <chart-path>
helm lint <chart-path>

# Installation and Upgrades
helm install <release-name> <chart>
helm upgrade <release-name> <chart>
helm rollback <release-name> <revision>

# Information and Debugging
helm list
helm status <release-name>
helm history <release-name>
helm get values <release-name>
helm template <release-name> <chart>

# Repository Management
helm repo add <name> <url>
helm repo update
helm search repo <keyword>

# Testing
helm test <release-name>
helm install <release-name> <chart> --dry-run --debug
```

You now have comprehensive knowledge of Helm and are ready to manage complex Kubernetes applications effectively!