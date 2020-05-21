+++
title = "Install kata-containers on K3S on aarch64"
date = 2020-05-21
tags = ["kubernetes", "raspberry"]
draft = false
+++

To install `kata-containers` on a `raspberry-pi` and integrating it into
`kubernetes` a few steps are currently required:

-   Install `kata-containers`
-   Setup integration into `containerd`
-   Setup integration into `kubernetes`


## Install `kata-containers` {#install-kata-containers}

Currently the easiest (and apparently only) officially supported way to
install `kata-containers` on an `aarch64` system is to use `snaps`.

If you are using the `Ubuntu` `arm` version this is already included in the
installation and you can install `kata-containers` with a single command[^fn:1]:

```sh
sudo snap install kata-containers --classic
```


## Integration into containerd {#integration-into-containerd}

When running `k3s` there is no `containerd` binary as it is embedded into
`k3s`, so configuration must be performed in the `k3s` configuration[^fn:2].

The whole process includes the following steps:

-   A `runtime` must be configured for `kata-containers` in `containerd`.
-   `shims` must be created for `containerd` to use `kata-containers`.
-   The node must be configured to be able to run `kata-container` workloads.
-   The kubernetes cluster must be aware that `kata` is a valid runtime.

To configure a `runtime`, the configuration file for `k3s` is found under:
`/var/lib/rancher/k3s/agent/etc/containerd/config.toml`, however it can not
be edited as it will be regenerated every time `k3s` is restarted. Instead it
[should be copied](https://rancher.com/docs/k3s/latest/en/advanced/) to
`/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl` and that file
can be modified.

```sh
cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml{,.tmpl}
```

The content should be:

```toml
[plugins.cri.containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"

[kata.options]
  ConfigPath = "/etc/kata-containers/configuration.toml"
```

The configuration files for `/etc/kata-containers/*.toml` must be copied from
the snap, because the default configuration will not be able to start on
Raspberry Pis with < 4GB of memory, as `qemu` requests too much memory by
default:

```sh
mkdir -p /etc/kata-containers/
cp /snap/kata-containers/current/usr/share/defaults/kata-containers/*.toml /etc/kata-containers/
```

Inside these configuration files (depending on the Pi) make sure to adjust the
value of `default_memory` to something the hardware can handle. You should at
least adjust the default `configuration.toml` and `configuration-qemu.toml`.

In addition `shim` files must be created in `/usr/local/bin` to point to the
`shim` binaries of `kata-containers` for `containerd` to pick it up.

The following script will create all the shims (including some that might not
actually be supported) - it was copied out of the `kata-deploy` project.

```sh
#!/bin/bash
shims=(
    "fc"
    "qemu"
    "qemu-virtiofs"
    "clh"
)

for shim in "${shims[@]}"; do
    shim_binary="containerd-shim-kata-${shim}-v2"
    shim_file="/usr/local/bin/${shim_binary}"
    shim_backup="/usr/local/bin/${shim_binary}.bak"

    if [ -f "${shim_file}" ]; then
        echo "warning: ${shim_binary} already exists" >&2
        if [ ! -f "${shim_backup}" ]; then
            mv "${shim_file}" "${shim_backup}"
        else
            rm "${shim_file}"
        fi
    fi
    cat << EOT | tee "$shim_file"
#!/bin/bash
KATA_CONF_FILE=/etc/kata-containers/configuration.toml /snap/kata-containers/current/usr/bin/containerd-shim-kata-v2 \$@
EOT
    chmod +x "$shim_file"
done

# On the PI a default shim was also needed
cp /usr/local/bin/containerd-shim-kata-qemu-v2 /usr/local/bin/containerd-shim-kata-v2
```

Now the `k3s-agent` must be restarted to reload the `containerd`
configuration and we can test that kata-container runtime is working by
running the following commands:

```sh
systemctl restart k3s-agent
ctr image pull docker.io/library/busybox:latest
ctr run --runtime io.containerd.run.kata.v2 -t --rm docker.io/library/busybox:latest hello sh
```

If the tests succeed the node can now be labeled as supporting `kata-containers`:

```sh
kubectl label node "$NODE_NAME" --overwrite katacontainers.io/kata-runtime=true
```

Now as a last step a new [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/) must be created for `kata` that can be
used to force pods to be run using it:

```yaml
apiVersion: node.k8s.io/v1beta1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
```

Now pods can use this runtime by specifying `runtimeClassName` inside the `spec`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-untrusted
spec:
  runtimeClassName: kata
  containers:
  - name: nginx
    image: nginx
```

[^fn:1]: As documented [here](https://github.com/kata-containers/packaging/blob/master/snap/README.md#initial-setup).
[^fn:2]: Most information in this section is a direct extract from [this PR](https://github.com/kata-containers/packaging/pull/823) adding `k3s` support to `kata-deploy`.
