# Lab 09: Remote Development with Telepresence on EKS

In this lab, you will learn how to use Telepresence for remote development and debugging of microservices. While most development is done locally using k3d clusters, this lab demonstrates Telepresence's power when working with remote cloud clusters like Amazon EKS for advanced debugging scenarios where local development isn't sufficient.

## Prerequisites
- Windows VM with VS Code and Remote Development extension
- Ubuntu VM with Docker, kubectl, and AWS CLI installed
- VS Code connected to Ubuntu VM via Remote Development extension
- AWS account with appropriate permissions for EKS

## Environment Setup
Ensure you're connected to your Ubuntu VM through VS Code Remote Explorer extension (see Lab 01 for SSH setup). All commands will be executed in the VS Code integrated terminal connected to your Ubuntu VM. Use VS Code's file explorer to navigate and manage directories and files.

**Note**: This lab uses EKS (a remote cloud cluster) instead of k3d because Telepresence is most valuable when debugging applications that need to interact with actual cloud services, databases, and production-like environments that can't be easily replicated locally.

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
sudo apt install -y unzip
unzip awscliv2.zip
sudo ./aws/install
```

Verify installations:
```bash
eksctl version
aws --version
kubectl version --client
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
export AWS_DEFAULT_REGION=us-west-1
```

## Create EKS Cluster

Create an EKS cluster using eksctl:

```bash
eksctl create cluster \
  --name telepresence-demo \
  --region us-west-1 \
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

# How to Debug Kubernetes Services With Telepresence

Use Telepresence connected to a remote Kubernetes cluster to locally debug Node.js microservices

Many organizations using Node.js adopt cloud native development practices with the goal of shipping features faster. The technologies and architectures may change when we move to the cloud, but the fact remains that we all still add the occasional bug to our code. The challenge here is that many of your existing local debugging tools and practices can't be used when everything is running in a container or on the cloud. A change in approach is required!

## Prerequisites

Before starting this lab, you will need:

1. **Kubernetes cluster access** - Either a remote cluster or something like k3d running locally
2. **kubectl configured** - To communicate with your cluster
3. **Node.js and npm** - We'll install these in the next section
4. **Code editor** - VS Code recommended for debugging features, but any editor will work
5. **curl or similar tool** - For testing HTTP endpoints
6. **Terminal access** - For running commands

## Step 1: Deploy a Sample Microservice Application

Here is an architecture diagram showing a high-level overview of the dependencies between services:

![img](https://miro.medium.com/v2/resize:fit:1400/0*7z4okLAi88Hli3lF)

In this architecture diagram, you'll notice that requests from users are routed through an ingress controller to our services. 

First, let's deploy the sample application to your Kubernetes cluster:

```
kubectl apply -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml
```

If you run `kubectl get service` you should see something similar to this:

```
kubectl get service
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
dataprocessingservice  ClusterIP   10.3.255.141    <none>        3000/TCP   5s
kubernetes             ClusterIP   10.3.240.1      <none>        443/TCP    17m
verylargedatastore     ClusterIP   10.3.241.162    <none>        8080/TCP   4s
verylargejavaservice   ClusterIP   10.3.243.172    <none>        8080/TCP   4s
```

Wait for all pods to be running:
```
kubectl get pods
```

## Step 2: Set up your Local Node.js Development Environment

You will need to configure your local development environment to debug the `DataProcessingService` Kubernetes service. As you can see in the architecture diagram above, the `DataProcessingService` is dependent on both the `VeryLargeJavaService` and the `VeryLargeDataStore`, so to make a change to this service, we'll have to interact with these other services as well. You can imagine that the web page generating monolith "VeryLargeJavaService" and "VeryLargeDataStore" are too resource hungry to run on your local machine.

So, let's get started with using our new approach to debugging!

1. Clone the repository for this application from GitHub.

```
mkdir -p ~/k3d/telepresence
cd ~/k3d/telepresence
git clone https://github.com/datawire/edgey-corp-nodejs.git
cd edgey-corp-nodejs/DataProcessingService
```

2. Install Node.js and NPM

   **For Ubuntu/Debian systems**, install Node.js and npm:

   ```bash
   # Method 1: Install from Ubuntu repositories (may be older version)
   sudo apt update
   sudo apt install nodejs npm
   
   # Verify installation
   node --version
   npm --version
   ```

   **For macOS:**
   ```bash
   brew install node
   ```

3. Install the project dependencies via NPM

   ```bash
   npm install
   ```

4. Test the application locally

   **Option A: Run without debugging (Command Line)**
   
   Start the application directly from the terminal:
   ```bash
   node app.js
   ```
   
   You should see output similar to:
   ```
   Welcome to the DataProcessingNodeService!
   Server running on port 3000
   ```

   **Option B: Run with VS Code debugging (Recommended for debugging)**
   
   If you want to use debugging features:
   
   a. Open VS Code and select "Open" from the "Welcome" screen. Navigate to the `DataProcessingService` directory and click "Open"
   
   ![img](https://miro.medium.com/v2/resize:fit:1400/0*9f72XcXgdMp-bK2N)
   
   b. Open the `app.js` file in the editor by double-clicking on it in the Explorer window
   
   ![img](https://miro.medium.com/v2/resize:fit:1400/0*Hdsq-Q53P-Ky_5Uj)
   
   c. Start the application in debug mode by clicking on 'Run and Debug' icon in the left side navigation pane, then click 'Run and Debug' and select 'Node.js'
   
   ![img](https://miro.medium.com/v2/resize:fit:1400/0*-e7wI8h24h8zAzM_)
   
   d. The application will start in debug mode with the debug control panel, debug console, and call stack explorer visible
   
   ![img](https://miro.medium.com/v2/resize:fit:1400/0*J0c-L8IXcEMApnxu)

5. Verify the local service is working by testing the color endpoint:

```
$ curl localhost:3000/color
"blue"
```

You now have your local service loaded into your IDE and running in debug mode! Now connect this to the remote Kubernetes cluster.

## Step 3: Install and Configure Telepresence

Instead of fiddling with remote debugging protocols and exposing ports via `kubectl port-forward` to access services running in our remote Kubernetes cluster we are going to use Telepresence, an open source Cloud Native Computing Foundation project. Telepresence creates a bidirectional network connection between your local development and the Kubernetes cluster to enable [fast, efficient Kubernetes development](https://www.getambassador.io/use-case/local-kubernetes-development/).

1. Install Telepresence CLI for your platform:

**For Linux (AMD64):**
```bash
sudo curl -fL https://github.com/telepresenceio/telepresence/releases/latest/download/telepresence-linux-amd64 -o /usr/local/bin/telepresence
```

2. Make the binary executable

```
sudo chmod a+x /usr/local/bin/telepresence
```

3. Verify installation

```
telepresence version
```

4. Install the Traffic Manager in your cluster

**IMPORTANT**: Before connecting, you must install the Traffic Manager:

```
telepresence helm install
```

5. Test Telepresence by connecting to the remote cluster

```
telepresence connect
```

You should see output similar to:
```
Connected to context your-cluster-context, namespace default
```

6. Send a request to the remotely running `DataProcessingService`:

```
curl http://dataprocessingservice.default.svc.cluster.local:3000/color
```

Expected output:
```
"green"
```

7. Notice two things here:

A. You can refer to the remote Service directly via its internal cluster name as if your development machine is inside the cluster

B. The color returned by the remote `DataProcessingService` is "green", versus the local result you saw above of "blue"

Great! You've successfully configured Telepresence. Right now, Telepresence is "intercepting" (discussed below) the request you're making to the Kubernetes API server, and routing over its direct connection to the cluster instead of over the Internet.

## Step 4: Intercept Remote Traffic and Debug Your Local Kubernetes Service

An [intercept](https://www.getambassador.io/docs/latest/telepresence/reference/intercepts/) is a routing rule for Telepresence. You can [create an intercept](https://www.getambassador.io/docs/latest/telepresence/howtos/intercepts/) to route all traffic intended for the `DataProcessingService` in the cluster to the local version of the `DataProcessingService` running in debug mode on port 3000.

**IMPORTANT**: Make sure your local DataProcessingService is running before creating the intercept. If you haven't started it yet:

```bash
cd ~/k3d/telepresence/edgey-corp-nodejs/DataProcessingService
node app.js
```

Keep this running in the background or in a separate terminal.

Create the intercept:

```
telepresence intercept dataprocessingservice --port 3000
```

You will see something similar to the following output:

```
Using Deployment dataprocessingservice
   Intercept name    : dataprocessingservice
   State             : ACTIVE
   Workload kind     : Deployment
   Intercepting      : 192.168.60.190 -> 127.0.0.1
       3000 -> 3000 TCP
   Volume Mount Error: remote volume mounts are disabled: sshfs is not installed on your local machine
```

Note: The Volume Mount Error is normal and doesn't affect the functionality.

Access the application directly with Telepresence. Visit [http://verylargejavaservice:8080](http://verylargejavaservice:8080/) in your browser. Again, Telepresence is intercepting requests from your browser and routing them directly to the Kubernetes cluster. You should see a web page that displays the architecture of the system you have deployed into your cluster:

![img](https://miro.medium.com/v2/resize:fit:1400/0*DsV_Ej9v5cpbZ1KF)

Note that the color of the title and `DataProcessingService` box is blue. This is because the color is being determined by the locally running copy of the `DataProcessingService` as the Telepresence intercept is routing the remote cluster traffic to this.

You can also test the intercept from command line:
```bash
curl http://dataprocessingservice.default.svc.cluster.local:3000/color
```

This should now return `"blue"` instead of `"green"`, confirming the intercept is working.

## Advanced Debugging (Optional)

If you're using VS Code with the debugging option, you can set breakpoints and inspect variables in real-time:

**Setting a Breakpoint:**

1. In VS Code, scroll the editor window with the `app.js` file content showing so that you can see line 42
2. Click once in the margin to the left of the line number to set a breakpoint
3. A red dot should appear indicating a breakpoint has been set

![img](https://miro.medium.com/v2/resize:fit:1400/0*dWerzEInUybca7fw)

**Testing the Breakpoint:**

1. In your browser, visit [http://verylargejavaservice:8080](http://verylargejavaservice:8080/) 
2. VS Code will jump to the foreground when the breakpoint is hit
3. You can view the call stack and current variables in the debug panels

![img](https://miro.medium.com/v2/resize:fit:1400/0*86q2P4pdr8XYqITb)

**Modifying Variables During Debug:**

1. In the Variables pane, expand the "Closure" section 
2. Locate the color variable and double-click on the value 'blue'
3. Change this value to 'orange' and press enter
4. Click "Continue" in the debug control panel

![img](https://miro.medium.com/v2/resize:fit:1400/0*QYhhguvaGmXfWDaC)

**Alternative: Simple Code Modification**

If you're not using VS Code debugging, you can still modify the behavior by:

1. Opening `app.js` in any text editor
2. Finding line 42 where the color is defined
3. Changing `'blue'` to `'orange'` 
4. Saving the file and restarting the Node.js application with `node app.js`

![img](https://miro.medium.com/v2/resize:fit:1400/0*QYhhguvaGmXfWDaC)

Your browser window should complete reloading and display an orange color for the title and `DataProcessingService` box:

![img](https://miro.medium.com/v2/resize:fit:1400/0*NoTQ31JUpA4cTps1)

Success! You have successfully made a request to the remote `VeryLargeJavaService` and Telepresence has intercepted the call this service has made to the remote `DataProcessingService` and rerouted the traffic to your local copy running in debug mode!

In addition to rapidly inspecting and changing variables locally, you can also step through the execution of the local service as if it were running in the remote cluster. You can view data passed into the local service from the service running in the remote cluster and interact with other services running in the cluster as if you were also running here.

## Cleanup

When you're done with the lab, clean up the intercept:

```bash
telepresence leave dataprocessingservice
telepresence quit
```

To remove the sample application from your cluster:

```bash
kubectl delete -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml
```

Delete the EKS cluster:
```bash
eksctl delete cluster --name my-cluster telepresence-demo
```