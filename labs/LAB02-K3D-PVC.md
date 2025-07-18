# Lab 02: K3d and Persistent Volumes

In this lab, you will learn how to work with persistent volumes in k3d clusters by mounting local directories.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker and k3d installed
- VS Code connected to Ubuntu VM via Remote Development extension
- Completed Lab 01: K3d Getting Started

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension (see Lab 01 for SSH setup). All commands will be executed in the VS Code integrated terminal connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories.

## Create Local Directory for Persistent Volume

First, create a local directory that will be mounted into the k3d containers:

```bash
mkdir -p /tmp/k3dvol
```

## Create k3d Cluster with Volume Mount

Create the cluster with a volume mount to enable persistent storage:

```bash
k3d cluster create "k3d-pvc" --volume /tmp/k3dvol:/tmp/k3dvol@server:0 --port "80:80@loadbalancer" --agents 2 --api-port 6551 --wait
```

**Note**: The `@server:0` specification ensures the volume is mounted on the server node where persistent volumes can be properly accessed.

You should see output similar to:
```
INFO[0000] Created network 'k3d-k3d-pvc'
INFO[0000] Created volume 'k3d-k3d-pvc-images'
INFO[0001] Creating node 'k3d-k3d-pvc-server-0'
INFO[0001] Creating node 'k3d-k3d-pvc-agent-0'
INFO[0001] Creating node 'k3d-k3d-pvc-agent-1'
INFO[0005] Creating LoadBalancer 'k3d-k3d-pvc-serverlb'
INFO[0013] Cluster 'k3d-pvc' created successfully!
```

## Verify Cluster Creation

Verify your cluster is running:

```bash
k3d cluster list
kubectl cluster-info
```

## Create Application with Persistent Volume

Create a YAML file for deploying an application with persistent storage. You can either:
- Use VS Code to create a new file called `app.yaml` in the file explorer, or
- Use the terminal command below:

```bash
cat > app.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - k3d-k3d-pvc-server-0
  hostPath:
    path: "/tmp/k3dvol"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
spec:
  selector:
    matchLabels:
      app: echo
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: echo
    spec:
      volumes:
      - name: task-pv-storage
        persistentVolumeClaim:
          claimName: task-pv-claim
      containers:
      - image: busybox
        name: echo
        volumeMounts:
        - mountPath: "/data"
          name: task-pv-storage
        command: ["sleep", "3600"]
EOF
```

## Deploy the Application

Deploy the application with persistent volume:

```bash
kubectl apply -f app.yaml
```

You should see:
```
persistentvolume/task-pv-volume created
persistentvolumeclaim/task-pv-claim created
deployment.apps/echo created
```

## Verify Persistent Volume Setup

Check the persistent volume and ensure it's bound:

```bash
kubectl get pv
```
**Expected output**: STATUS should show "Bound"

Check the persistent volume claim and ensure it's bound:

```bash
kubectl get pvc
```
**Expected output**: STATUS should show "Bound"

Check the pod and ensure it's running:

```bash
kubectl get pods
```
**Expected output**: STATUS should show "Running"

**If PV/PVC are not bound, check troubleshooting section below.**

## Test Persistent Storage

1. **Wait for the pod to be ready:**
   ```bash
   kubectl wait --for=condition=Ready pod -l app=echo --timeout=60s
   ```

2. **Get the pod name:**
   ```bash
   export POD_NAME=$(kubectl get pods -l app=echo -o jsonpath='{.items[0].metadata.name}')
   echo "Pod name: $POD_NAME"
   ```

3. **Connect to the running pod:**
   ```bash
   kubectl exec -it $POD_NAME -- sh
   ```

4. **Create a file in the persistent volume:**
   ```bash
   echo $(hostname) > /data/hostname.txt
   cat /data/hostname.txt
   exit
   ```

5. **Verify the file exists on the host:**
   ```bash
   cat /tmp/k3dvol/hostname.txt
   ```

6. **Delete the pod and verify persistence:**
   ```bash
   kubectl delete pod $POD_NAME
   ```

7. **Wait for the new pod to start and verify data persistence:**
   ```bash
   kubectl wait --for=condition=Ready pod -l app=echo --timeout=60s
   export NEW_POD_NAME=$(kubectl get pods -l app=echo -o jsonpath='{.items[0].metadata.name}')
   echo "New pod name: $NEW_POD_NAME"
   kubectl exec -it $NEW_POD_NAME -- cat /data/hostname.txt
   ```

   The file should still contain the hostname from the previous pod, demonstrating data persistence.

## Troubleshooting

**If pods don't start:**
```bash
# Check pod status
kubectl describe pods -l app=echo

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

**If persistent volume doesn't mount:**
```bash
# Check if the directory exists
ls -la /tmp/k3dvol/

# Check persistent volume status
kubectl describe pv task-pv-volume
kubectl describe pvc task-pv-claim
```

## Clean Up

When you're done with the lab:

```bash
kubectl delete -f app.yaml
k3d cluster delete k3d-pvc
```

## Summary

In this lab, you learned how to:
- Create a k3d cluster with volume mounts
- Create persistent volumes and persistent volume claims
- Deploy applications that use persistent storage
- Verify data persistence across pod restarts
- Clean up resources when done

---

**Next Lab:** [LAB03-K3D-Rancher.md](LAB03-K3D-Rancher.md)