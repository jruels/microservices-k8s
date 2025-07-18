# Using Tilt to automate development

This lab walks through installing and using Tilt for Kubernetes development automation.

## Prerequisites
- Docker and k3d installed
- VS Code
- kubectl installed

## Step 1: Create a k3d Cluster
```bash
k3d cluster create tilt --api-port 6550 --agents 2
```

## Step 2: Install Tilt
```bash
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash
```

## Step 3: Run the Tilt Demo
Tilt Avatars is a demo app to try Tilt quickly.
```bash
tilt demo
```

## Step 4: Explore Tilt Avatars
- The demo launches a Python backend and JS frontend.
- Tilt manages builds and deploys via Dockerfiles and YAML.
- You can view logs and status in the Tilt UI.

## Step 5: Build Your Own Project
- Once familiar, author your own `Tiltfile` for your project.
- See: https://docs.tilt.dev/tiltfile_concepts.html

## Cleanup
- Exit Tilt and delete the cluster:
```bash
k3d cluster delete tilt
```

## Next Steps
Proceed to the Telepresence lab for remote debugging workflows.





