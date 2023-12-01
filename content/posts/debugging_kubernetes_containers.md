+++
title = "Debugging kubernetes containers"
author = ["Florian Bergmann"]
date = 2023-12-01
tags = ["kubernetes"]
draft = false
+++

The scenario: you have a running kubernetes cluster, but suddenly some of your
containers start to have problems.

Obviously now there must be a way to find out what exactly is causing these
problems.

In this post I'll highlight two ways to debug a container running on kubernetes:

-   `ephemeral-containers` (which is likely what you should use most of the time)
-   `nsenter` (which is a last resort debugging option that requires access to the
    node the Pod is running on)

> Want to follow along? Setup a test cluster and run the provided command to get
> your very own failing container:
>
> ```yaml
> ---
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: myconfig
>   namespace: default
> data:
>   config.toml: |
>     [default]
>   additional_file: ""
> ---
> apiVersion: v1
> kind: Pod
> metadata:
>   name: my-broken-pod
>   namespace: default
> spec:
>   containers:
>   - name: working-container
>     image: gcr.io/google_containers/pause-amd64:3.0
>   - name: broken-container
>     image: quay.io/fbergman/broken-pod:latest
>     imagePullPolicy: IfNotPresent
>     resources:
>       limits:
>         memory: "128Mi"
>         cpu: "500m"
>     ports:
>       - containerPort: 3000
>     readinessProbe:
>       httpGet:
>         path: /readyz
>         port: 3000
>     volumeMounts:
>       - mountPath: /opt/
>         name: configuration
>   volumes:
>     - name: configuration
>       configMap:
>         name: myconfig
> ```
>
> ````sh
> minikube start
> kubectl apply -f /some-web-url
> ````

In this case one of the `LivenessProbes` starts failing.

In case of one container (`working-container`), there is no way to debug into
it, because it does not contain any executable shell. This is a pretty common
scenario when running minimal container images like [distroless](https://github.com/GoogleContainerTools/distroless) containers.

The other container allows spawning a shell, but is missing most utilities that
would make debugging easier: there is no `strace`, no `gdb` and no default
networking tools.

How can we now proceed to figure out what breaks the container starting?

Obviously we can hope that a `kubectl describe pod` or the logs of our
application has some useful information, but let's say this is not the case and
running `kubectl logs my-broken-pod` simply returns nothing useful.

In this case the two already mentioned approaches can be used - as it should be
your first stop let's start with **ephemeral containers**:


## Using ephemeral containers {#using-ephemeral-containers}

When having to debug a pod, ephemeral containers are the frist tool to use,
because in general access to cluster nodes might not be available.

When using ephemeral containers it is important to have some container images
ready that contain tools you want to use for debugging.

A very good network debugging container is the [netshoot](https://github.com/nicolaka/netshoot) container.

But if all you need is a basic shell (in case of a distroless container) -
busybox might already be enough.


### Starting an ephemeral container {#starting-an-ephemeral-container}

Ephemeral containers are handled as an additional field in the `spec` of a
container: [see the apidocs](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#ephemeralcontainer-v1-core).

Ephemeral containers can only be added to an already running pod - trying to add
them to a Pod during creation will result in an error:

`````yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: ephemeral-container-on-start
  labels:
    name: ephemeral-container-on-start
spec:
  containers:
  - name: main-container
    image: busybox:1.28
    resources:
      limits:
        memory: "64"
        cpu: "250m"
  ephemeralContainers:
  - name: debug-container
    image: busybox:1.28
    resources:
      limits:
        memory: "64Mi"
        cpu: "250m"
`````

`````sh
The Pod "ephemeral-container-on-start" is invalid:
* spec.ephemeralContainers[0].resources: Forbidden: cannot be set for an Ephemeral Container
* spec.ephemeralContainers: Forbidden: cannot be set on create
`````

That leaves two options to add an ephemeral container to a Pod:

1.  The `kubectl debug` command:
    `````sh
       kubectl debug -it [pod-name] --image=busybox:1.28 --target=[container-name]
    `````
2.  Patching the Pod using the API directly using the new resource:
    `````sh
       # Just for this experiment, give the default serviceaccount all permissions
       kubectl create clusterrolebinding --clusterrole cluster-admin --serviceaccount default:default default-all-permissions
       # APIURL
       APIURL=$(kubectl config view --minify --output jsonpath="{.clusters[*].cluster.server}")
       # Impersonate a serviceaccount with the required privileges
       TOKEN=$(kubectl create token default)
       # Call the API using curl as this serviceaccount
       curl --header "Authorization: Bearer $TOKEN" \
           --header "Content-Type: application/strategic-merge-patch+json" \
           --request PATCH \
           -d '
       {
           "spec":
           {
               "ephemeralContainers":
               [
                   {
                       "name": "debugger",
                       "command": ["sh"],
                       "image": "busybox",
                       "targetContainerName": "broken-container",
                       "stdin": true,
                       "tty": true,
                       "volumeMounts": []
                   }
               ]
           }
       }' \
           "${APIURL}/api/v1/namespaces/default/pods/my-broken-pod/ephemeralcontainers" -k
    `````
    Once the container has been added like that it can be used using `kubectl
          exec -ti my-broken-pod -c debugger sh`. Getting rid of the new container can
    be done, by killing the process that was already running when `execing` into
    the container: `kill -9 <pid_of_initial_sh>`.

What will not work is using `kubectl edit`, as this is [explicitly not supported](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/#what-is-an-ephemeral-container):

> Ephemeral containers are created using a special ephemeralcontainers handler in the API rather than by adding them directly to pod.spec, so it's not possible to add an ephemeral container using kubectl edit.

Once the container is launched (and you either already are inside the shell when
using `kubectl debug` or have attached via `kubectl exec`) you can run whatever
commands might be necessary.


### Cleaning up ephemeral containers {#cleaning-up-ephemeral-containers}

Checking the apidocs also shows what might be an issue for some user:

> Ephemeral containers may not be removed or restarted.

So - once a ephemeral container has been added to a Pod, the only way to
completely get rid of it from the Pod spec is by recreating the Pod without it.

It's also strongly recommended to not keep a process running in the ephemeral
container, because it will work against the requests and limits of the entire
pod:

> The kubelet may evict a Pod if an ephemeral container causes the Pod to exceed its resource allocation.


## Using nsenter {#using-nsenter}

Using `nsenter` will not only work for kubernetes pods, so this is something you
can also use for `docker` or `podman` containers.

Generally this is useful if you want to pick-and-choose which **namespaces** of
the container you want to access.

> In kubernetes this will only work, if there is a way to gain access to the
> cluster node the Pod you want to debug is running on.
>
> So either via `ssh` or using a [openshift debug node/NODE](https://www.redhat.com/sysadmin/how-oc-debug-works) container.

Once access to the node is established the process running inside the pod needs
to be found: a lot of these commands depend on the container runtime in use, so
for the remainder I will assume the cluster is using `CRIO`.[^fn:1]

1.  Find the Pod ID for our `my-broken-pod` pod:
    `````sh
       POD_ID=$(crictl pods --name my-broken-pod -o json | jq -r '.items[] | select(.state == "SANDBOX_READY") | .id')
    `````
2.  Find all processes inside this pod:
    `````sh
       CONTAINERS=$(crictl ps --pod d718ccb51d73896035324f6d9d9d12cc6818027ab18e5f78b295b518c62b46bd -o json | jq -r '.containers[].id')
    `````
3.  Find all processes inside these containers (or just choose the one container that is running the crashing process):
    `````sh
       crictl inspect 154c67e44fcc843201aa582214395c82eb0e40acfda1ef1a9e12567f371aa13d | jq -r ".info.pid"
    `````

With the `PID` it is now possible to enter some of the namespaces of this
process, while maintaining access to all binaries installed on the machine we
SSHed to (as long as the `mount` namespace is **not** used with `nsenter` -
indicated by the `-m` flag):

`````sh
PID=1234
nsenter -t $PID -n -p -u
`````

Often it can be useful to add [namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html) incrementally to see if one of the
namespaces might have an impact on the container's behaviour.

In this case it's now possible to inspect the process that is refusing to work
with all tools available on the host - in case of a `minikube` cluster this
includes tools like `lsof` and `strace`.

But first let's check if the `ReadinessProbe` works right now:

`````sh
curl -I localhost:3000/readyz
`````

`````text
HTTP/1.1 500 Internal Server Error
content-type: text/plain; charset=utf-8
content-length: 7
date: Fri, 01 Dec 2023 11:16:12 GMT
`````

So it is returning a **500** error - maybe we can get some more information what
the process is actually doing using strace:

`````sh
strace -f -p "$PID"
`````

`````text
[pid 128904] epoll_wait(3,  <unfinished ...>
[pid 128893] futex(0x7f6985695940, FUTEX_WAIT_PRIVATE, 1, NULL <unfinished ...>
[pid 128904] <... epoll_wait resumed>[], 1024, 202) = 0
[pid 128904] epoll_wait(3, [], 1024, 17) = 0
[pid 128904] statx(AT_FDCWD, "/opt/config/additional_file", AT_STATX_SYNC_AS_STAT, STATX_ALL, {stx_mask=STATX_ALL|STATX_MNT_ID, stx_attributes=0, stx_mode=S_IFREG|0644, stx_size=0, ...}) = 0
[pid 128904] write(1, "app_state bad - configuration at"..., 61) = 61
[pid 128904] write(4, "\1\0\0\0\0\0\0\0", 8) = 8
`````

It seems to `stat` a configuration file at `/opt/config/additional_file` -
that's strange, as the configuration should live in another file
(`config.toml`)[^fn:2].

Time to actually use the mount namespace and see what's in `/opt/config`

`````sh
nsenter -t $PID -n -p -u -m
ls /opt/config
`````

`````sh
additional_file  config.toml
`````

Well the file is there - so let's update our configuration to not contain it and
see if it unbreaks our application:

`````sh
kubectl patch -n default configmaps myconfig --type=json -p '[{"op": "remove", "path": "/data/additional_file"}]'
`````

After removing that key from the `ConfigMap` and a Pod restart it seems our
application is finally happy!

`````text
my-broken-pod   2/2     Running   0          10s
`````

> Obviously in this case the `strace` command could also have been run directly
> from the node, but knowing how to incrementally add container namespaces can
> still be useful - e.g. to check how a mounted filesystem really looks like
> inside the container.


## References {#references}

-   [Ephemeral containers (official documentation)](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
-   [Ephemeral containers (great blog post)](https://iximiuz.com/en/posts/kubernetes-ephemeral-containers/)
-   [nsenter man page](https://man7.org/linux/man-pages/man1/nsenter.1.html)

[^fn:1]: If the process is easy to identify running `ps` and grepping for the
    process commandline will also work.
[^fn:2]: Obviously in this example this file was mounted from the `ConfigMap` - in
    reality this might indicate problems with other software that might inject
    libraries into all processes via `LD_PRELOAD` modifications.