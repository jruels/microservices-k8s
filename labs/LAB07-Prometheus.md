# Lab 07: Prometheus Monitoring on Kubernetes

In this lab, you will learn how to deploy Prometheus for monitoring Kubernetes clusters and applications.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, k3d, kubectl, and Helm installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Completed Lab 06: Skaffold and k3d

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension (see Lab 01 for SSH setup). All commands will be executed in the VS Code integrated terminal connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories and files.

## Create k3d Cluster

Create a k3d cluster for this lab:

```bash
k3d cluster create prometheus-demo --api-port 6556 --agents 2 --wait
```

Verify the cluster is running:

```bash
k3d cluster list
kubectl cluster-info
```

## Create Monitoring Namespace

It's good practice to run your Prometheus containers in a separate namespace:

```bash
kubectl create namespace monitor
```

## Add Prometheus Helm Repository

Add the Prometheus community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Verify the repository was added:

```bash
helm repo list
```

## Install Prometheus Operator

Install the Prometheus operator using the kube-prometheus-stack chart:

```bash
helm install prometheus-operator prometheus-community/kube-prometheus-stack -n monitor
```

This installs:
- Prometheus operator
- Prometheus server
- Alertmanager
- Grafana
- kube-state-metrics
- node-exporter
- Service monitors for Kubernetes components

## Verify Installation

Check that all components are running:

```bash
kubectl get pods -n monitor
```

You should see pods for:
- alertmanager-prometheus-operator-alertmanager-0
- prometheus-operator-grafana-*
- prometheus-operator-kube-state-metrics-*
- prometheus-operator-operator-*
- prometheus-operator-prometheus-node-exporter-*
- prometheus-prometheus-operator-prometheus-0

Wait for all pods to be in Running state before proceeding.

## Access Prometheus Dashboard

Port forward to access the Prometheus dashboard:

```bash
kubectl port-forward -n monitor svc/prometheus-operator-kube-prometheus-prometheus 9090:9090
```

Open your browser and navigate to http://127.0.0.1:9090 to access the Prometheus dashboard.

## Explore Prometheus Metrics

In the Prometheus dashboard, try these queries:

1. **Node CPU usage**:
   ```
   node_cpu_seconds_total
   ```

2. **Pod memory usage**:
   ```
   container_memory_usage_bytes
   ```

3. **Kubernetes API server requests**:
   ```
   apiserver_request_total
   ```

4. **Pod restart count**:
   ```
   kube_pod_container_status_restarts_total
   ```

## Access Grafana Dashboard

Get the Grafana admin credentials:

```bash
kubectl get secret --namespace monitor prometheus-operator-grafana -o jsonpath="{.data.admin-user}" | base64 --decode && echo
kubectl get secret --namespace monitor prometheus-operator-grafana -o jsonpath="{.data.admin-password}" | base64 --decode && echo
```

Port forward to access Grafana:

```bash
kubectl port-forward -n monitor svc/prometheus-operator-grafana 3000:80
```

Open your browser and navigate to http://127.0.0.1:3000 to access Grafana.

Login with the credentials from the previous step (default: admin/prom-operator).

## Explore Grafana Dashboards

Grafana comes with pre-configured dashboards:

1. **Kubernetes / Compute Resources / Cluster** - Overall cluster resource usage
2. **Kubernetes / Compute Resources / Namespace (Pods)** - Pod-level metrics
3. **Kubernetes / Compute Resources / Node (Pods)** - Node-level metrics
4. **Kubernetes / Networking / Cluster** - Network metrics
5. **Node Exporter / Nodes** - Detailed node metrics

Navigate to Dashboards → Browse to see all available dashboards.

## Deploy Sample Application

Let's deploy a sample application to monitor. You can either:
- **Use VS Code**: Create a new file called `sample-app.yaml` in VS Code editor and add the contents below
- **Use terminal**: Run the command below to create the file

```bash
cat > sample-app.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - name: web
    protocol: TCP
    port: 80
    targetPort: 80
  type: NodePort
EOF
```

Deploy the application:

```bash
kubectl apply -f sample-app.yaml
```

## Monitor the Application

1. **View pod metrics in Prometheus**:
   ```
   container_memory_usage_bytes{pod=~"sample-app.*"}
   ```

2. **Check CPU usage**:
   ```
   rate(container_cpu_usage_seconds_total{pod=~"sample-app.*"}[5m])
   ```

3. **Monitor network traffic**:
   ```
   rate(container_network_receive_bytes_total{pod=~"sample-app.*"}[5m])
   ```

## Create Custom ServiceMonitor

Create a custom ServiceMonitor to scrape metrics from your application:

```bash
cat > service-monitor.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app-monitor
  namespace: monitor
spec:
  selector:
    matchLabels:
      app: sample-app
  endpoints:
  - port: web
    interval: 30s
    path: /metrics
EOF
```

Apply the ServiceMonitor:

```bash
kubectl apply -f service-monitor.yaml
```

## Set Up Alerts

Create a simple alert rule:

```bash
cat > alert-rule.yaml << 'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: sample-app-alerts
  namespace: monitor
spec:
  groups:
  - name: sample-app.rules
    rules:
    - alert: HighMemoryUsage
      expr: container_memory_usage_bytes{pod=~"sample-app.*"} > 100000000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage detected"
        description: "Pod {{ $labels.pod }} is using {{ $value }} bytes of memory"
EOF
```

Apply the alert rule:

```bash
kubectl apply -f alert-rule.yaml
```

## View Alerts

Check alerts in Prometheus:
1. Go to http://127.0.0.1:9090/alerts
2. View the alert rules and their current status

## Generate Load for Testing

Generate some load to trigger metrics:

```bash
kubectl run load-generator --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://sample-app; done"
```

Watch the metrics change in both Prometheus and Grafana dashboards.

## Clean Up Load Generator

Remove the load generator:

```bash
kubectl delete pod load-generator
```

## Important Metrics to Monitor

Key metrics for production monitoring:

1. **Node Metrics**:
   - `node_cpu_seconds_total` - CPU usage
   - `node_memory_MemAvailable_bytes` - Available memory
   - `node_filesystem_avail_bytes` - Available disk space

2. **Pod Metrics**:
   - `kube_pod_container_status_restarts_total` - Pod restarts
   - `container_memory_usage_bytes` - Memory usage
   - `container_cpu_usage_seconds_total` - CPU usage

3. **Cluster Metrics**:
   - `kube_node_status_condition` - Node health
   - `kube_deployment_status_replicas` - Deployment status
   - `apiserver_request_total` - API server requests

## Clean Up

Clean up the resources:

```bash
# Delete sample application
kubectl delete -f sample-app.yaml

# Delete monitoring components
kubectl delete -f service-monitor.yaml
kubectl delete -f alert-rule.yaml

# Uninstall Prometheus operator
helm uninstall prometheus-operator -n monitor

# Delete namespace
kubectl delete namespace monitor

# Delete cluster
k3d cluster delete prometheus-demo
```

## Troubleshooting

### Common Issues:

1. **Pods not starting**
   - Check resource limits: `kubectl describe pod -n monitor <pod-name>`
   - Verify sufficient cluster resources: `kubectl top nodes`

2. **Grafana login issues**
   - Regenerate credentials: `kubectl get secret --namespace monitor prometheus-operator-grafana -o yaml`
   - Reset password if needed

3. **Metrics not appearing**
   - Check ServiceMonitor configuration
   - Verify target discovery in Prometheus: Status → Targets

4. **Port forwarding issues**
   - Check if ports are already in use: `netstat -tlnp | grep 9090`
   - Use different local ports if needed

---

**Next Lab:** [LAB08-Tilt.md](LAB08-Tilt.md)