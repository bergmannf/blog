+++
title = "Deploying Teamspeak on a Raspberry PI kubernetes cluster"
date = 2020-05-08T22:59:00+02:00
tags = ["kubernetes", "arm", "raspberry"]
draft = false
+++

While this post will end up with a running [Teamspeak](https://www.teamspeak.com/en/) server, it is very hard
on the resources of the Raspberry Pi and might not be suitable for everyday
use.

To deploy a teamspeak server on raspberry pi a few things need to be done:

-   (optional) Get a Kubernetes cluster up and running (this is not required, if
    you just want to run the `docker` container directly).
-   Get Teamspeak to run on `ARM`.
-   Setup an ingress controller to make the Teamspeak server accessible from the
    outside world (in this case this will be the [Nginx Ingress](https://kubernetes.github.io/ingress-nginx/)).


## Setting up a kubernetes cluster {#setting-up-a-kubernetes-cluster}

To setup a kubernetes cluster on Raspberry Pis [K3S](https://k3s.io/) is very good approach, as
the cluster will be more lightweight that simply installing upstream
kubernetes.

As a base I recommend using [HypriotOS](https://blog.hypriot.com/) or [Ubuntu Server](https://ubuntu.com/download/server/arm)[^fn:1] as those allow
configuring the images using [Cloud-Init](https://cloudinit.readthedocs.io/en/latest/).

Make sure to follow the instructions to use the the `legacy` backend for
`iptables` if installing `kubernetes` v1.17 or lower: [kubeadm instructions](https://v1-17.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#ensure-iptables-tooling-does-not-use-the-nftables-backend).

When installing `k3s` to run `Teamspeak` `Traefik` should not be installed,
as only the 2.x version supports `UDP` ingresses - so instead `nginx-ingress`
will be installed later:

```sh
curl -sfL https://get.k3s.io | sh -s - --no-deploy=traefik
```


## Deploy Teamspeak {#deploy-teamspeak}


### Build an ARM image for Teamspeak {#build-an-arm-image-for-teamspeak}

Teamspeak does not provide a binary for `ARM`.

It is however possible to run it using [Qemu](https://www.qemu.org/) - I have already prepared an
`ARM` image that will run the Teamspeak server through `qemu` that you can
find on my [Dockerhub](https://hub.docker.com/repository/docker/monadt/teamspeak3-server) - or if you want to see the source checkout the
repository on [Github](https://github.com/bergmannf/teamspeak3-server-arm).

The image is using the same `entrypoint.sh` as the [official image](https://hub.docker.com/%5F/teamspeak) - so if
you are already using that one you should be able to use it exactly the same
way (if not - feel free to open an issue).


### Deploy the image {#deploy-the-image}

Now - if you do not want to use `kubernetes`, you can simply use the image
using `docker` and expose the required `Teamspeak` ports as you would with
`docker`:

```sh
docker run -p 9987:9987/udp -p 10011:10011 -p 30033:30033 -e TS3SERVER_LICENSE=accept monadt/teamspeak3-server
```

This way is a lot easier on the resources and will likely run more reliable
on a Raspberry PI with < 2 GB of RAM.


## Setup nginx-ingress {#setup-nginx-ingress}


### Get nginx-ingress to run on ARM {#get-nginx-ingress-to-run-on-arm}

Setting up the `nginx-ingress` on a cluster running on `ARM` needs a few extra
steps when using the [official documentation](https://kubernetes.github.io/ingress-nginx/deploy/).

The images used in the `manifests` are not compatible with `armv7` (that is
used when running a cluster on a bunch of Raspberry Pis).

First the `mandatory.yaml` has to be updated to use the images for the `arm`
architecture[^fn:2]:

```sh
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
sed -i 's/nginx-ingress-controller:0/nginx-ingress-controller-arm:0/' mandatory.yaml
```

The resulting `mandatory.yaml` file can now be applied to the cluster:

```sh
kubectl apply -f mandatory.yaml
```

In my local cluster I am using the `NodePort` approach, so the service for
that can be applied next:

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml
```


## Setup a Teamspeak deployment {#setup-a-teamspeak-deployment}

With all pieces in place the `Teamspeak` container can now be deployed onto
the cluster:

Save the following `yaml` into a file (e.g. `teamspeak.yaml`).

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: teamspeak
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: teamspeak-pvc
  namespace: teamspeak
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 256Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: teamspeak-deployment
  namespace: teamspeak
  labels:
    app: teamspeak
spec:
  replicas: 1
  selector:
    matchLabels:
      app: teamspeak
  template:
    metadata:
      namespace: teamspeak
      labels:
        app: teamspeak
    spec:
      containers:
        - name: teamspeak-server
          image: monadt/teamspeak3-server:3.11.0
          ports:
            - name: ts
              containerPort: 9987
              protocol: UDP
          resources:
          env:
          - name: TS3SERVER_LICENSE
            value: accept
          volumeMounts:
          - mountPath: /var/ts3server/
            name: teamspeak-data
      volumes:
        - name: teamspeak-data
          persistentVolumeClaim:
            claimName: teamspeak-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: teamspeak-service
  namespace: teamspeak
  labels:
    app: teamspeak
spec:
  type: ClusterIP
  ports:
    - port: 9987
      targetPort: ts
      protocol: UDP
      name: ts
  selector:
    app: teamspeak
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: udp-services
  namespace: ingress-nginx
data:
  9987: "teamspeak/teamspeak-service:9987"
```

Apply this using `kubectl`:

```sh
kubectl apply -f teamspeak.yaml
```

The 9987 `udp` port will also need to be added to the `ingress` service.
In the `ports` section of the service add the following snippet:

```sh
kubectl edit svc ingress-nginx -n ingress-nginx
```

```yaml
- name: teamspeak
  port: 9987
  protocol: UDP
  targetPort: 9987
```


## Forwarding traffic to the ingress {#forwarding-traffic-to-the-ingress}

The final step depends a lot on the setup you are deploying the cluster in.

If it is behind your local router, you have to check which port was bound to
the 9987 `udp` port and forward this to one of your cluster-nodes:

```sh
kubectl describe svc ingress-nginx -n ingress-nginx
```

```text
Name:                     ingress-nginx
Namespace:                ingress-nginx
Labels:                   app.kubernetes.io/name=ingress-nginx
                          app.kubernetes.io/part-of=ingress-nginx
Annotations:              Selector:  app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx
Type:                     NodePort
IP:                       10.43.33.70
Port:                     http  80/TCP
TargetPort:               80/TCP
NodePort:                 http  32224/TCP
Endpoints:
Port:                     https  443/TCP
TargetPort:               443/TCP
NodePort:                 https  31955/TCP
Endpoints:
Port:                     teamspeak  9987/UDP
TargetPort:               9987/UDP
NodePort:                 teamspeak  31222/UDP
Endpoints:
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

In this case the port that needs to be forwarded is the `31222` port (the
`NodePort` for the 9987 `UDP` port).

[^fn:1]: For Ubuntu you will have to edit the file `/boot/firmware/cmdline.txt` and add the options `cgroup_memory=1 cgroup_enable=memory` at the end of the line for `k3s` (or `containers` in general) to work.
[^fn:2]: See <https://github.com/kubernetes/ingress-nginx/pull/3852>
