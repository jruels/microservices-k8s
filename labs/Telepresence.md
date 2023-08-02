# How to Debug Kubernetes Services With VS Code and Telepresence

Use VS Code and Telepresence connected to a remote Kubernetes cluster to locally debug Node.js microservices

Many organizations using Node.js adopt cloud native development practices with the goal of shipping features faster. The technologies and architectures may change when we move to the cloud, but the fact remains that we all still add the occasional bug to our code. The challenge here is that many of your existing local debugging tools and practices can’t be used when everything is running in a container or on the cloud. A change in approach is required!



## Step 1: Deploy a Sample Microservice Application

We assume you have access to a Kubernetes cluster, either a remote cluster or something like k3d running locally that you can pretend is a remote cluster. We also assume you have a [current version of Node installed](https://nodejs.org/en/download/), alongside [VS Code](https://code.visualstudio.com/).

Here is an architecture diagram showing a high-level overview of the dependencies between services:

![img](https://miro.medium.com/v2/resize:fit:1400/0*7z4okLAi88Hli3lF)

In this architecture diagram, you’ll notice that requests from users are routed through an ingress controller to our services. 

First, let’s deploy the sample application to your Kubernetes cluster:

```
kubectl apply -f https://raw.githubusercontent.com/datawire/edgey-corp-nodejs/main/k8s-config/edgey-corp-web-app-no-mapping.yaml
```

If you run `kubectl get service` you should see something similar to this:

```
kubectl get service
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
dataprocessingservice ClusterIP 10.3.255.141 <none> 3000/TCP 5s
kubernetes ClusterIP 10.3.240.1 <none> 443/TCP 17m
verylargedatastore ClusterIP 10.3.241.162 <none> 8080/TCP 4s
verylargejavaservice ClusterIP 10.3.243.172 <none> 8080/TCP 4s
```

## Step 2: Set up your Local Node.js Development Environment and VS Code

You will need to configure your local development environment to debug the `DataProcessingService` Kubernetes service. As you can see in the architecture diagram above, the `DataProcessingService` is dependent on both the `VeryLargeJavaService` and the `VeryLargeDataStore`, so to make a change to this service, we’ll have to interact with these other services as well. You can imagine that the web page generating monolith “VeryLargeJavaService” and “VeryLargeDataStore” are too resource hungry to run on your local machine.

So, let’s get started with using our new approach to debugging!

1. Clone the repository for this application from GitHub.

```
mkdir ~/k3d/telepresence
cd ~/k3d/telepresence
git clone https://github.com/datawire/edgey-corp-nodejs.git
cd edgey-corp-nodejs/DataProcessingService
```

2. Install the dependencies via NPM

   1. If it is not installed use `brew`

      ```bash
      brew install npm
      ```

      

```
npm install
```

3. After NPM has finished downloading the dependencies, start VS Code and select “Open” from the “Welcome” screen. Navigate to the `DataProcessingService` and click the “Open” button

![img](https://miro.medium.com/v2/resize:fit:1400/0*9f72XcXgdMp-bK2N)

4. After the project loads into VS Code, open the `app.js` file in the editor by double-clicking on this in the Explore window

![img](https://miro.medium.com/v2/resize:fit:1400/0*Hdsq-Q53P-Ky_5Uj)

5. Start the application in debug mode by clicking on ‘Run and Debug’ icon in the left side navigation pane, and then clicking the ‘Run and Debug’ button and selecting ‘Node.js’ from the dropdown menu that is shown.

![img](https://miro.medium.com/v2/resize:fit:1400/0*-e7wI8h24h8zAzM_)

6. The application will start in debug mode, and the (1) debug control panel is shown, alongside the (2) debug console (where application logging will be shown) and the (3) call stack explorer

![img](https://miro.medium.com/v2/resize:fit:1400/0*J0c-L8IXcEMApnxu)

7. In a terminal window, type the command `curl localhost:3000/color` to see that your locally running debug service is returning the color `blue`.

```
$ curl localhost:3000/color
“blue”
```

You now have your local service loaded into your IDE and running in debug mode! Now connect this to the remote Kubernetes cluster.

## Step 3: Install and Configure Telepresence

Instead of fiddling with remote debugging protocols and exposing ports via `kubectl port-forward` to access services running in our remote Kubernetes cluster we are going to use Telepresence, an open source Cloud Native Computing Foundation project. Telepresence creates a bidirectional network connection between your local development and the Kubernetes cluster to enable [fast, efficient Kubernetes development](https://www.getambassador.io/use-case/local-kubernetes-development/).

1. Install Telepresence CLI (macOS version). See the [documentation](https://www.getambassador.io/docs/latest/telepresence/quick-start/) for a Linux version.

```
# macOSX 
sudo curl -fL https://app.getambassador.io/download/tel2oss/releases/download/v2.14.2/telepresence-darwin-arm64 -o /usr/local/bin/telepresence
```

2. Make the binary executable

```
sudo chmod a+x /usr/local/bin/telepresence
```

3. Test Telepresence by connecting to the remote cluster

```
telepresence connect
```

4. Send a request to the remotely running `DataProcessingService`:

```
curl http://dataprocessingservice.default.svc.cluster.local:3000/color
“green”
```

5. Notice two things here:

A. You can refer to the remote Service directly via its internal cluster name as if your development machine is inside the cluster

B. The color returned by the remote `DataProcessingService` is “green”, versus the local result you saw above of “blue”

Great! You’ve successfully configured Telepresence. Right now, Telepresence is “intercepting” (discussed below) the request you’re making to the Kubernetes API server, and routing over its direct connection to the cluster instead of over the Internet.

## Step 4: Intercept Remote Traffic and Debug Your Local Kubernetes Service

An [intercept](https://www.getambassador.io/docs/latest/telepresence/reference/intercepts/) is a routing rule for Telepresence. You can [create an intercept](https://www.getambassador.io/docs/latest/telepresence/howtos/intercepts/) to route all traffic intended for the `DataProcessingService` in the cluster to the local version of the `DataProcessingService` running in debug mode on port 3000.

Create the intercept

```
telepresence intercept dataprocessingservice — port 3000
```

You will see something similar to the following output:

```
Using deployment dataprocessingservice
intercepted
Intercept name : dataprocessingservice
State : ACTIVE
Destination : 127.0.0.1:3000
Volume Mount Error: macFUSE 4.0.5 or higher is required on your local machine
Intercepting : all TCP port connections
```

Access the application directly with Telepresence. Visit [http://verylargejavaservice:8080](http://verylargejavaservice:8080/) in your browser. Again, Telepresence is intercepting requests from your browser and routing them directly to the Kubernetes cluster. You should see a web page that displays the architecture of the system you have deployed into your cluster:

![img](https://miro.medium.com/v2/resize:fit:1400/0*DsV_Ej9v5cpbZ1KF)

Note that the color of the title and `DataProcessingService` box is blue. This is because the color is being determined by the locally running copy of the `DataProcessingService`as the Telepresence intercept is routing the remote cluster traffic to this.

Within VS Code, scroll the editor window with the `app.js` file content showing so that you can see line 42. Click once in the margin to the left of the line number to set a breakpoint. A red dot should appear both where you clicked and also within the code on that line to indicate a breakpoint has successfully been set. This breakpoint will be triggered when the “color” endpoint of the `DataProcessingService` is called.

![img](https://miro.medium.com/v2/resize:fit:1400/0*dWerzEInUybca7fw)

In your browser, visit [http://verylargejavaservice:8080](http://verylargejavaservice:8080/) again. Notice how VS Code immediately jumps to the foreground on your desktop with the (1) breakpoint hit. You can view the (2) call stack in the bottom left corner of the Debug window, and you can also see the (3) current variables involved with the execution of this service in the “Variables” panel to the left of the editor window.

At this point, you can perform all of the debug actions you typically can, e.g. inspecting variable values, changing variables, (4) stepping through and over code, and halting execution.

![img](https://miro.medium.com/v2/resize:fit:1400/0*86q2P4pdr8XYqITb)

In the Variables pane on the left side of the screen, click once on the “Closure” section to expand the list of variables shown (3 in the image above).

Locate the color variable and double-click on the value ‘blue’. This should be editable. Change this value to ‘orange’ and press enter (1, below). Next, click the “Continue” icon in the debug control panel (2)

![img](https://miro.medium.com/v2/resize:fit:1400/0*QYhhguvaGmXfWDaC)

Your browser window should complete reloading and display an orange color for the title and `DataProcessingService` box:

![img](https://miro.medium.com/v2/resize:fit:1400/0*NoTQ31JUpA4cTps1)

Success! You have successfully made a request to the remote `VeryLargeJavaService` and Telepresence has intercepted the call this service has made to the remote `DataProcessingService` and rerouted the traffic to your local copy running in debug mode!

In addition to rapidly inspecting and changing variables locally, you can also step through the execution of the local service as if it were running in the remote cluster. You can view data passed into the local service from the service running in the remote cluster and interact with other services running in the cluster as if you were also running here.





