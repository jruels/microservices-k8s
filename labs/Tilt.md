# Using Tilt to automate development

This lab walks through installing and using Tilt. 



## Create a cluster 

```bash
k3d cluster create tilt --api-port 6550 --agents 2
```



## Install Tilt

```bash
curl -fsSL https://raw.githubusercontent.com/tilt-dev/tilt/master/scripts/install.sh | bash
```



# Launching & Managing Resources

## Tilt Avatars

Tilt Avatars is a demo application, so you can try Tilt without first setting it up for your own project.

Tilt Avatars consists of a Python web API backend to generate avatars and a JavaScript SPA (single page app) frontend. 

![Randomized Tilt avatar generation](https://docs.tilt.dev/assets/docimg/tutorial/tilt-avatars.gif)

> **No two projects are alike**
>
> This project uses `Dockerfile`s with Docker as the build engine and `kubectl`-friendly YAML files. But that’s only a small subset of Tilt functionality.
>
> Tilt supports much more!  (e.g. Helm, CRDs, `podman`).
>
> Once you’re comfortable with how Tilt works, follow the guides on `Tiltfile` authoring to build your own project

## Run `tilt up` for the First Time

Tilt brings consistency to your development not only due to ensuring a reproducible dev-environment-as-code, but launching any Tilt project is the same with the `tilt up` command, so you always have a familiar starting point. `tilt up` starts the Tilt control loop, which we’ll explore in more detail in a moment.

For this tutorial, we’re going to use a special `tilt demo` command, which will perform a few steps:

1. Create a temporary, local Kubernetes development cluster in Docker
2. Clone the [Tilt Avatars](https://github.com/tilt-dev/tilt-avatars) sample project
3. Launch a `tilt up` session for the sample project using the temporary Kubernetes cluster
4. Clean everything up on exit 🧹

Run the following command in your terminal to get started:

```
tilt demo
```



You should see output similar to the following in your terminal:

![Running tilt up in a Terminal window shows "Tilt started on http://localhost:3366/" message](https://docs.tilt.dev/assets/docimg/tutorial/tilt-up-cli.gif)

First, open the sample project directory (path is in the `tilt demo` output) in your preferred editor so that you can make changes in the following steps.

Once the sample project is open, return focus to the terminal window and press `(Spacebar)`, and the Tilt UI will be opened in your default browser.

In the next section, we’ll explain the Tilt UI. But first, let’s dissect what’s happening in the background.



## `Tiltfile`

When you run `tilt up`, Tilt looks for a special file named `Tiltfile` in the current directory, which defines your dev-environment-as-code.

A `Tiltfile` is written in [Starlark](https://docs.bazel.build/versions/main/skylark/language.html), a simplified dialect of Python.

Because your `Tiltfile` is a program, you can configure it with familiar constructs like loops, functions, and arrays. This makes the `Tiltfile` more extensible than a configuration file format like JSON or YAML, which requires hard-coding all possible options upfront.

When Tilt executes the `Tiltfile`:

1. Built-in functions like [`k8s_yaml`](https://docs.tilt.dev/api#api.k8s_yaml) and [`docker_build`](https://docs.tilt.dev/api#api.docker_build) register information with the Tilt engine
2. Tilt uses the resulting configuration to assemble resources to build and deploy
3. Tilt watches **relevant** source code files so it can trigger an update of the associated resource(s)

Within Tilt, the `Tiltfile` is itself a resource, so **you can even modify your `Tiltfile` and see the changes without restarting Tilt**!

![Sample Tiltfile code](https://docs.tilt.dev/assets/docimg/tutorial/tiltfile.png)

You can skip container re-builds and Pod re-deployments entirely via [Smart Rebuilds with Live Update](https://docs.tilt.dev/tutorial/5-live-update).

Open the [`tilt-avatars` Tiltfile](https://github.com/tilt-dev/tilt-avatars/blob/main/Tiltfile) and read through it. 



## Resources

A “resource” is a bundle of work managed by Tilt. For example: a Docker image to build + a Kubernetes YAML to apply.

> **Resources don’t have to be containers!**
>
> Tilt can also [manage locally-executed commands](https://docs.tilt.dev/local_resource) to provide a unified experience no matter how your code runs.

Resource bundling is **automatic** in most cases: Tilt finds the relationship between bits of work (e.g. `docker build` + `kubectl apply`). When that’s not sufficient, `Tiltfile` functions like [`k8s_resource`](https://docs.tilt.dev/api#api.k8s_resource) let you configure resources on top of what Tilt does automatically.

Because Tilt assembles multiple bits of work into a single resource, it’s much easier to determine status and find errors across update (build/deploy) and runtime.

### Update Status

Whenever you run `tilt up` or change a source file, Tilt determines which resources need to be changed to bring them up-to-date.

To execute these updates, Tilt might:

- Compile code locally on your machine (e.g. `make`)
- Build a container image (e.g. `docker build`)
- Deploy a modified K8s object or Helm chart (e.g. `kubectl apply -f` or `helm install`)

![Resource pane showing an update error](https://docs.tilt.dev/assets/docimg/tutorial/tilt-ui-update-status.png)

Tilt knows which files correspond to a given resource and update action. It won’t re-build a container just because you changed a Pod label, or re-compile your backend when you’ve only edited some JSX.

> When you `tilt up`, if your services are already running and haven’t changed, Tilt won’t unnecessarily re-deploy them!

### Runtime Status

Unfortunately, just because it builds does not mean it works.

In Tilt, the runtime status lets you know what’s happening with your code *after* it’s been deployed.

![Resource pane showing a runtime error](https://docs.tilt.dev/assets/docimg/tutorial/tilt-ui-runtime-status.png)

More importantly, Tilt lets you know *why*. 

## The Control Loop

Tilt is based on the idea of a [control loop](https://docs.tilt.dev/controlloop). This gives you real-time, circular feedback: something watches, something reacts, and equilibrium is maintained.

This is intentionally more “hands-free” than other dev tools. Traditional build systems like `make` are oriented around tasks that are invoked on-demand by the user. Even many service-oriented development tools like `docker-compose up` don’t *react* to changes once started. Newer tools, such as Webpack, often include hot module reload, but have limitations. (For example, changes to `webpack.config.js` require a manual restart.)

![Diagram of Tilt's control loop architecture](https://docs.tilt.dev/assets/img/controlloop/06.jpg)

Some examples of what Tilt handles for you:

- Source code file changes → sync to running container
- Dependency changes (e.g. `package.json`) → sync to running container and then run code in the container (e.g. `npm install`)
- Build spec changes (e.g. `Dockerfile`) → re-build container image + re-deploy
- Deployment spec changes (e.g. `app.yaml`) → reconcile deployment state (e.g. `kubectl apply -f ...`)
- `Tiltfile` changes → re-evaluate and create new resources, modify existing, and delete obsolete as needed

**So, once you’ve run `tilt up`, you can focus on your code and let Tilt continuously react to your changes without worrying if they’re the “right” type of changes.**

This has other benefits: for example, when you run `tilt up`, Tilt won’t re-deploy any already running and up-to-date services.

## Launching the Web UI

In your terminal window running `tilt demo`, press `(Spacebar)`. Tilt will open your default browser to the Tilt UI. (Or navigate there directly in your preferred browser using the URL from the terminal.)

## Resource Overview

The Resource Overview is the first thing you see when you open the Tilt UI. You can always return to it by clicking the Tilt logo in the upper left corner.

![Tilt UI resource overview](https://docs.tilt.dev/assets/docimg/tutorial/tilt-ui-table.png)

This view shows all your resources (services) at a glance, grouped by their [resource labels](https://docs.tilt.dev/tiltfile_concepts#resource-groups).

The Resource Overview is essential to get a quick view of your project’s entire state and offers critical info at a glance:

- **Update and Resource Status**

  If you look at the `api` resource row, you’ll see both the update and runtime status. Since this is a “Kubernetes Deploy” type resource, the update included building the image and `kubectl apply`ing it the cluster. The runtime status reflects the Pod’s current state in the cluster. e.g. Is it running and passing readiness checks?

- **Pod ID**

  Copy a Pod ID to your clipboard in one click, so you can interact with it as needed via `kubectl` or other tools.

- **Widgets**

  [Custom buttons](https://docs.tilt.dev/buttons) let you run any one-off tasks (unit tests, lint, etc.) you’ve configured for a resource.

- **Endpoints**

  Remembering port numbers when you’ve got a bunch of services can be a challenge. Endpoints gives you quick access to all your Tilt managed port forwards. You can also define custom endpoints for relevant external references such as a wiki page so that they’re never more than a click away.

- **Trigger Mode**

  By default, resources in Tilt are updated whenever a relevant file changes. It’s possible to change the default behavior on a per-resource basis (or globally) in the Tiltfile with [manual update control](https://docs.tilt.dev/manual_update_control). The trigger mode toggles for each resource in the UI make it easy to quickly pause and resume automatic updates to it.

  Even if a resource is in manual mode, it’s always possible to trigger an update on-demand!

From here, click the endpoint link on the “web” resource to open the frontend for the Tilt Avatars app.

![Using "Endpoints" from Tilt UI Resource Overview to open the Tilt Avatars frontend](https://docs.tilt.dev/assets/docimg/tutorial/tilt-ui-web-endpoint.gif)

Finished making an awesome avatar? 😻

Click on the “api” resource to navigate to the Resource Details view for the respective resource.

## Resource Details

The central focus of the Resource Detail view is logs, but all the information from the Resource Overview such as [custom buttons](https://docs.tilt.dev/buttons), endpoints, and pod IDs are available here as well.

Try clicking the “Trigger Update” (↻) button next to the “web” resource to run a manual update, which will re-build and re-deploy the Pod: ![Triggering an update for the "web" resource in the Tilt UI Resource Detail view](https://docs.tilt.dev/assets/docimg/tutorial/tilt-ui-trigger-update.gif)

> The “All Resources” link in the navbar will show logs for all services at once instead of a single service

### Log Filtering

Tilt provides several mechanisms to focus your logs:

- **Source**

  By default, Tilt shows both build/update and runtime logs interleaved. It’s possible to restrict this to a single source. For example, if you’re trying to fix an error during your resource start, it might be helpful to temporarily hide the build logs to reduce noise as you make changes.

- **Level**

  In addition to unifying your logs, Tilt collects errors and warnings from different tools such as Docker build errors, Kubernetes events, and more. You can quickly filter the view to just these important events including the surrounding context by clicking `... (more)`.

- **Keyword/Regex Filter**

  If you’ve ever tried to catch an error whiz by while tailing logs, you might have found yourself copying the output to a text editor to search through it. Tilt lets you non-destructively filter by keywords or regex match.

![Filtering logs via source, level, and keyword in the Tilt UI Resource Detail view](https://docs.tilt.dev/assets/docimg/tutorial/tilt-ui-logs.gif)

## What Else?

You can extend the Tilt UI with [custom buttons](https://docs.tilt.dev/buttons) to run common tasks such as unit tests or lint with one-click. Buttons support parameterized inputs and the log output goes directly to the relevant resource, so you don’t have to jump back and forth between a terminal and the Tilt UI.

Otherwise, the Tilt UI is designed to be unobtrusive and run in the background, notifying you only when something needs your attention.

Multi-service development might be complex, but we aim for simplicity in the Tilt UI.

## Update Code

Tilt embraces the concept of a [control loop](https://docs.tilt.dev/tutorial/2-tilt-up#the-control-loop), so once you’ve run `tilt up`, it’s a “hands free” development experience.

As you edit your code, Tilt will automatically run update steps such as building an updated container image and deploying it.

Let’s test it out:

1. Navigate to the “web” resource in the Tilt UI and click “Clear Logs”
2. Open `web/vite.config.js` in your favorite editor
3. Find the `logLevel` line and change it from `'error'` to `'info'`
4. Save the file
5. Watch magic happen for the `web` resource in the Tilt UI

![Tilt updating a resource after a code change](https://docs.tilt.dev/assets/docimg/tutorial/tilt-code-change-full-rebuild.gif)

What just happened?

## 1. File Changed

First, Tilt saw a file change and associated it with the “web” resource:

```log
1 File Changed: [web/vite.config.js] • web
```

Let’s take a peek into the `Tiltfile` to understand:

- Why Tilt was watching for changes to `web/vite.config.js`
- How it knew `web/vite.config.js` belonged to the “web” resource

For reference, here’s an abbreviated file hierarchy for [Tilt Avatars](https://github.com/tilt-dev/tilt-avatars):

```log
tilt-avatars/
├── api/
│   └── ...
├── deploy/
│   ├── web.dockerfile
│   └── ...
├── web/
│   ├── vite.config.js
│   └── ...
└── Tiltfile
```

Ready? Okay. In the Tilt Avatars `Tiltfile`, which is at the repo root, the container image build for the “web” resource looks like this:

```
docker_build(
    'tilt-avatar-web',
    context='.',
    dockerfile='./deploy/web.dockerfile',
    only=['./web/'],
    ignore=['./web/dist/'],
    live_update=[...]  # omitted for brevity
)
```



> Path arguments for Tiltfile functions are relative to your `Tiltfile`’s path (refer back to the file hierarchy above if you get confused).

Several of these arguments include paths. Let’s go through them one by one:

- **`context`: build context for image build** (specified here as the current directory, which is the repo root)

  Tilt watches for changes to any modified files in this directory or any subdirectory, recursively. Perfect! We changed `./web/vite.config.js`, which meets this criteria.

  However, let’s keep looking at the other arguments to make sure they don’t negate or alter this somehow…

- **`dockerfile` (optional): path for the `Dockerfile` to be used**

  This is optional and defaults to `./Dockerfile` to mimic `docker build ...` CLI behavior. Tilt will watch this path (`./deploy/web.dockerfile` in our case) and trigger an image re-build if it changes, but it’s not relevant here because we didn’t edit it, so let’s move on…

- **`only` (optional): filters paths included in build context and restricts file watching to the subset of paths**

  Because we have a “mono-repo” (multiple services in a single repository) and the build context is the repo root (`.`), we set this to `['web/']` so that unrelated changes, such as those to the backend (files under `api/`) don’t trigger a re-build of the “web” resource. Since `./web/vite.config.js` *is* under `./web/`, it hasn’t been excluded, which is what we want!

  Just one more argument to go…

- **`ignore` (optional): excludes certain paths from the build context and ignores changes to them**

  We’ve used this to exclude `./web/dist/` for our production web assets, which otherwise match the rules defined by `context` and `only`. This is supplementary to `.dockerignore`, but can be helpful for cases where you want different ignore rules for local dev with Tilt, for example.

  As `./web/vite.config.js` is *not* under `./web/dist/`, it was not ignored, which is what we’d expect.

If we put all this together, Tilt is watching for any file changes in the `web/` directory or any of its subdirectories, recursively, EXCEPT for those in `web/dist` (or any of its subdirectories, recursively).

When a matching file changes, such as `web/vite.config.js`, because it’s watched by the `tilt-avatar-web` container image build configuration, Tilt initiates an update for the “web” resource.

> **How does Tilt know the `tilt-avatar-web` image belongs to the “web” resource?**
>
> You might remember that a resource can be composed of multiple bits of work. In the case of the “web” resource, it has a container image build and a Kubernetes Deployment.
>
> Tilt associated the `tilt-avatar-web` container image with the “web” resource because the container image name is referenced in Kubernetes YAML loaded in the `Tiltfile` with `k8s_yaml`. (This is not the only way that container images can be assembled into a resource, and it’s possible to manually configure where auto-assembly is insufficient.)

## 2. Resource Update

Now, the update process starts:

```log
STEP 1/3 — Building Dockerfile: [tilt-avatar-web]
  ...

STEP 2/3 — Pushing localhost:44099/tilt-avatar-web:tilt-0b9fcdf9cfea47ba
  ...

STEP 3/3 — Deploying
     Injecting images into Kubernetes YAML
     Applying via kubectl:
     → web:deployment
```

First, Tilt built an updated version of the container image. Then, it pushed the image to our local registry so that it can be used by Kubernetes. Tilt adapts its workflow based on your local cluster setup. Finally, it deployed the updated image.

>  **Immutable Image Tags**
>
> Tilt tags every image it builds with a unique `:tilt-<hash>` tag. It then rewrites the Kubernetes YAML (or Helm chart) on the fly during deployment to use this tag.
>
> Why? Using a “rolling” tag (such as `:latest`) can result in hard-to-debug issues depending on factors like image pull policy configuration. With an immutable tag, you’re guaranteed *exactly* what got built is what will run.
>
> It’s just one more thing Tilt takes care of without any extra configuration to save you a headache later.

## 3. Resource Runtime Monitoring

Once deployed, Tilt starts tracking the updated version of the resource:

```log
Tracking new pod rollout (web-7f9b8b65f4-wt97k):
     ┊ Scheduled       - <1s
     ┊ Initialized     - <1s
     ┊ Ready           - 1s
```

As Tilt waits for the resource to become ready, it’ll pass along relevant events, such as image pull status or container crashes, so you don’t need to resort to manually investigating a failed deploy with `kubectl`.

Once the container has started, Tilt will stream the logs. In our case, since we enabled more verbose logging for Vite (the dev server that hosts the frontend), we should see some messages as it starts up:

```log
yarn run v1.22.5
$ vite
Pre-bundling dependencies:
  react
  react-dom
(this will be run only when your dependencies or config have changed)

    ...

  ready in 946ms.
```

If you’re a bit underwhelmed by changing a log level, we do much more soon! The [Tilt Avatars](https://github.com/tilt-dev/tilt-avatars) project is configured to use Live Update for regular development, so we purposefully made a change in a config file that meant the full container would be rebuilt.

Let’s move on to the next section, where we’ll make more interesting code changes with Live Update.



# Smart Rebuilds with Live Update

Tilt’s deep understanding of your resources means the right things get rebuilt at the right times.

Even with Docker layer caching, rebuilding a container image can be slow. For unoptimized Kubernetes-based development, every code change requires:

1. Rebuilding the container image (`docker build ...`)
2. Pushing the built image to a registry (`docker push ...`)
3. Updating the tag in YAML and applying the the Deployment to the cluster (`kubectl apply -f ...`)
4. Waiting for the rollout of new Pods using the updated image (open Reddit, 😴, etc.)

Live Update solves these challenges by performing an **in-place update of the containers in your cluster**.

It works with frameworks that natively support hot reload (e.g., Webpack) and compiled languages.

Time to try it out:

1. Open `api/app.py` in your favorite editor
2. Find the commented-out line `# 'other': ['accessory']`
3. Uncomment it (remove the leading `#`)
4. Save the file
5. Watch magic happen for the `api` resource in the Tilt UI
6. Open the Tilt Avatars web app http://localhost:5735/
7. Dress the character with some stylish glasses (this is important!!! 😎)

![Tilt Live Updating a container after a code change](https://docs.tilt.dev/assets/docimg/tutorial/tilt-code-change-live-update.gif)

To understand what happened, let’s take a look at the `docker_build` configuration for the `tilt-avatar-api` image:

```
docker_build(
    'tilt-avatar-api',
    context='.',
    dockerfile='./deploy/api.dockerfile',
    only=['./api/'],
    live_update=[
        sync('./api/', '/app/api/'),
        run(
            'pip install -r /app/requirements.txt',
            trigger=['./api/requirements.txt']
        )
    ]
)
```

It looks a lot like the `tilt-avatar-web` image configuration we saw in the last section.

What’s not omitted this time is the `live_update` argument value, which defines a series of steps to run (in-order) to Live Update a container.

## `sync()`

We have a single sync step defined:

```
sync('./api/', '/app/api/')
```

The first argument (`./api/`) is the path, relative to the `Tiltfile`, on our machine that we want Tilt to watch for changes to (recursively). The destination path (`/app/api/`) is the absolute path *inside the container* where we want the files copied to.

> Files you sync to the container *must* match paths that Tilt is already watching for the image configuration

In practice, that results in what we saw in the “api” logs in Tilt:

```log
Will copy 1 file(s) to container: 4a9aac5527
- '/Users/quixote/dev/tilt-avatars/api/app.py' --> '/app/api/app.py'
  → Container 4a9aac5527 updated!
```

Since Flask (Python web framework) provides a dev server with hot module support, copying the file is all that was needed! Live Update also supports situations where the framework does not support reloading code at runtime by restarting your process with an updated version of the code in the container, which saves the overhead of image build and deployment. For details, refer to the full [Live Update Reference](https://docs.tilt.dev/live_update_reference#restarting-your-process).

## `run()`

When modifying non-code files, it’s sometimes necessary to run additional command(s) to process them.

For example, our project has a run step to install new or updated Python dependencies using pip (Python package manager):

```
run(
    'pip install -r /app/requirements.txt',
    trigger=['./api/requirements.txt']
)
```

The first argument is a command to run *inside the container*. The `trigger` argument defines a path, relative to the `Tiltfile`, on our machine that, when changed, will result in the command being run in the container.

Now, when we change our project’s dependencies in `./api/requirements.txt`, the updated version of the file will first be synced to the container. Then, because it matches the run step’s `trigger` condition, the command will be run in the container to install new/updated dependencies.

Go ahead and try it out by making a change to `./api/requirements.txt`. (Hint: lines beginning with `#` will be ignored, so add a new line like `# hello from Tilt tutorial!` and save the file.) You’ll see that not only is the file copied as before when we modified `./api/app.py`, but that this time, the run step executes as well:

```log
Will copy 1 file(s) to container: 4a9aac5527
- '/Users/quixote/dev/tilt-avatars/api/requirements.txt' --> '/app/api/requirements.txt'
[CMD 1/1] sh -c pip install -r /app/requirements.txt
   ...
  → Container 4a9aac5527 updated!
```

## 

## Clean up 

In the tilt directory, run `ctrl+c` to shut things down. 



Delete the k3d cluster 

```bash
k3d cluster delete tilt
```





