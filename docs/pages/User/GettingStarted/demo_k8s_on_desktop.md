---
title: Using Liqo to borrow resources
weight: 5
---

## Scenario

Many times, as students, we would like to use the environment of our PCs but also leverage the hardware capabilities that
the University provided us. This is pretty useful in particular when the university was provided of specific hardware facilities such as GPUs. On the other hand, the system admins want to keep control of resources, limit the amount of consumption of a user, etc.


In this tutorial, we provide a proof-of concept example of how Liqo can be used to offload desktop applications on remote clusters.
More precisely, we offload an instance of [Blender](https://www.blender.org/), a graphical application that runs much faster if the hosting computer includes a powerful GPU.

![](/images/k8s-on-desktop-demo/introduction_schema.svg)

## Introduction

This tutorial shows how to install [Liqo](https://liqo-io) on two [k3s](https://k3s.io/) clusters from scratch and then run a real application on the foreign cluster.
In particular, we use a desktop application _containeraized_ with the [KubernetesOnDesktop](#about-kubernetesondesktop) project, which aims at executing traditional Desktop applications in a remote environment, while showing their graphical user interface locally.

Obviously, for the purpose of this demo it is better (but not required) that the remote machine features an NVIDIA graphic card, while the local machine can be a traditional laptop.
To be more specific, we will execute a Blender `pod` in a *foreign cluster* (that is represented in the *local cluster* as a *virtual node* named `liqo-<...>`) and a viewer `pod` in the *local cluster*.

**Note**: from now on, following the Liqo naming, when we talk about "*local cluster*" we refer to the one that runs the `cloudify` script ([see afterwards](#about-kubernetesondesktop)), and when we talk about "*foreign cluster*" we refer to the other one.


### The KubernetesOnDesktop project
[KubernetesOnDesktop](https://github.com/netgroup-polito/KubernetesOnDesktop) (KoD) aims at developing a cloud infrastructure to run desktop applications in a remote Kubernetes cluster.
In a nutshell, KoD splits traditional desktop applications in a backend (running the actual application) and a frontend, showing the graphical interface and interacting with the (desktop) user.
This enables desktop applications to be executed also on a remote machine, while keeping their GUI locally.

Technically, KoD leverages a client/server VNC+PulseAudio+SSH infrastructure that enables to start the application `pod` in a k8s remote node and redirects its GUI (through VNC) and the audio (through PulseAudio+SSH) in a second `pod` scheduled in the node where the `cloudify` application is running.
The communication between the two components leverages several kubernetes primitives, such as `deployments`, `jobs`, `services` and `secrets`.
For further information see the [KubernetesOnDesktop](https://github.com/netgroup-polito/KubernetesOnDesktop) GitHub page.

So far, KoD supports firefox, libreoffice and blender, the latter with the capability to exploit any NVIDIA GPU (through the NVIDIA CUDA driver) available in the remote node.
In any case, thanks to the massive use of templates, many more applications can be easily supported.

When executing the `cloudify` command, the application will create:
  * a `secret` containing an ssh key that allows the two application components to communicate securely;
  * a `deployment` containing the application (e.g. blender) and the VNC server, whose `pod` will be scheduled on the remote cluster;
  * a `ClusterIP` `service` that allows the remote `pod` to be reachable from other `pod`s in the cluster by using [K8s DNS for services](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/);
  * a `pod` executing the VNC viewer, started in the local machine (i.e., on the same node where you run `cloudify`).

## Installation of the required software

To install all the required software we need to follow these steps:

1. [Install k3s](#install-k3s) in both clusters;
2. [Install Liqo](#install-liqo) in both clusters;
3. [Install KubernetesOnDesktop](#install-kubernetesondesktop) in the *local cluster*. Note that the foreign cluster simply runs a vanilla Liqo, without any other software.

### Install k3s

Assuming you already have two Linux machines (or Virtual Machines) up and running in the same LAN, we can install [k3s](https://k3s.io/) by using the official script as documented in the [K3s Quick-Start Guide](https://rancher.com/docs/k3s/latest/en/quick-start/).
So, you only need to run the following command:

```bash
curl -sfL https://get.k3s.io | sh -
```

A tiny customization of the above default install is required for [Liqo](https://liqo.io) to work.
When the script ends, you need to modify the `/etc/systemd/system/k3s.service` file by adding the `--kube-apiserver-arg anonymous-auth=true` service execution parameter as in the following command:

```bash
sudo sed -i "s#server#server --kube-apiserver-arg anonymous-auth=true#" /etc/systemd/system/k3s.service
```

{{%expand "After this operation, your 'k3s.service' file should look like the following:" %}}

```
[Unit]
Description=Lightweight Kubernetes
Documentation=https://k3s.io
Wants=network-online.target

[Install]
WantedBy=multi-user.target

[Service]
Type=notify
EnvironmentFile=/etc/systemd/system/k3s.service.env
KillMode=process
Delegate=yes
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
TimeoutStartSec=0
Restart=always
RestartSec=5s
ExecStartPre=-/sbin/modprobe br_netfilter
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/k3s \
    server --kube-apiserver-arg anonymous-auth=true \

```
{{% /expand%}}

{{%expand "Note: If you want to exploit any available NVIDIA GPUs in the foreign cluster, you have to follow the additional steps below (in the foreign cluster):" %}}
1. [Install in the required NVIDIA CUDA driver](https://github.com/NVIDIA/nvidia-docker/wiki/Frequently-Asked-Questions#how-do-i-install-the-nvidia-driver);
2. [Install the Docker engine](https://docs.docker.com/engine/install/);
3. [Install the `nvidia-container-runtime`](https://github.com/nvidia/nvidia-container-runtime#installation);
4. Add the `--docker` service execution parameter in the `/etc/systemd/system/k3s.service` file to let k3s using docker instead of containerd as container runtime.
   Indeed, Docker is the only container runtime officially supported by NVIDIA. You can do this by executing the following command:
```bash
sudo sed -i "s#server#server --docker#" /etc/systemd/system/k3s.service
```
{{% /expand%}}

Now you need to apply the changes by executing the following commands to restart the `k3s` service:
```bash
systemctl daemon-reload
systemctl restart k3s.service
```

Finally, to facilitate the interactions with K3s, we suggest to modify the default setup in order to allow the `kubectl` command to interact with the installed cluster without `sudo`.
This can be achieved by copying the `k3s.yaml` config file in a _user_ folder, change its owner and export the `KUBECONFIG` environment variable as follows:
```bash
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/
sudo chown $USER:$USER $HOME/.kube/k3s.yaml
export KUBECONFIG="$HOME/.kube/k3s.yaml"
```

> **NOTE**: You need to export the `KUBECONFIG` environment variable each time you open a new terminal by running, as above, `export KUBECONFIG="$HOME/.kube/k3s.yaml"`. If you want to make `KUBECONFIG` environment variable permanent, you can add it to your shell configuration file by executing the following command:
> ```bash
> echo 'export KUBECONFIG="$HOME/.kube/k3s.yaml"' >> $HOME/.bashrc
> ```

Before proceeding with the [Liqo](https://liqo.io) installation, wait for all pods to be in `Running` status; for this, you can execute the command `kubectl get pod --all-namespaces`.


### Install Liqo
To install [Liqo](https://liqo.io), you have to (1) export manually the required environment variables and (2) use the script provided in the project.
This can be done with the following commands:

```bash
export POD_CIDR=10.42.0.0/16
export SERVICE_CIDR=10.43.0.0/16
curl https://raw.githubusercontent.com/liqotech/liqo/master/install.sh | bash
```

For detailed information see the [Liqo Installation Guide](/user/gettingstarted/install/#custom-install); particularly, check that your Liqo instance works properly.

Before proceding with the installation of [KubernetesOnDesktop](https://github.com/netgroup-polito/KubernetesOnDesktop) in one of the two clusters, wait for all the `pod`s in `liqo` `namespace` to be up and running in both clusters.
You can check it by executing `kubectl get pod -n liqo` in both clusters.

Since both (virtual) machines are connected to the same local area network, each Liqo cluster will automatically join the foreign one thanks the Liqo [Discovery and Peering](/user/gettingstarted/peer/) features.

### Install KubernetesOnDesktop
Now that both [k3s](https://k3s.io/) and [Liqo](https://liqo.io) are up and running, we can install [KubernetesOnDesktop](https://github.com/netgroup-polito/KubernetesOnDesktop) by executing the following command:

```bash
sudo curl -L https://raw.githubusercontent.com/netgroup-polito/KubernetesOnDesktop/master/install.sh | sudo bash -s -- --remote
```

Now we are ready to run the KubernetesOnDesktop `cloudify` script as described in the next section.


## Run the KubernetesOnDesktop demo
To run the demo we need to execute the `cloudify` command with the following command:

```bash
cloudify -t 500 -r pod -s blender
```
{{%expand "Expand here to understand the meaning of the different parameters:" %}}
  * `-t 500`: specifies a timeout in seconds. If the pod does not reach the `Running` status within this timeout, the native application will be automatically executed (if available) by the script. The very first time you execute the _cloudified_ application, you should specify a large value for this parameter because of the time required to pull the required Docker images from the public repository;
  * `-r pod`: specifies the run mode. In this case the viewer will be a k8s `pod` too (as the application one) and will be scheduled on the local node;
  * `-s`: enables secure communication between the application `pod` and the viewer `pod`;
  * `blender`: the (supported) application we want to execute. If you have a NVIDIA graphic card (with the required drivers already installed as specified [above](#install-k3s)) in the remote node, you can use that card with blender!
{{% /expand%}}

The `cloudify` application will create the `kod` `namespace` (if not present) and will apply on it the `liqo.io/enabled=true` `label` so that this `namespace` could be extended to the Liqo *foreign cluster* (See [Exploit foreign cluster resources](/user/gettingstarted/test/#start-hello-world-pod)).

In addition, `cloudify` adds a label to the local `node` to allow k3s to schedule the pods according to the node affinity specified in the `kubernetes/deployment.yaml`, particularly that the application `pod` must be executed in a remote node.
Similarly, `kubernetes/vncviewer.yaml` specifies that the viewer must be executed on the local node.

Since the *foreign cluster* is abstracted by Liqo as a *local cluster* node, the `cloudify` application does not need to directly interact with the foreign cluster to deploy the application components.
Additionally, the communication between pods requires nothing but traditional Kubernetes services and the [service name DNS resolution](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/) works as usual.
Additionally, even though there are two separated clusters and the `pod`s are spread across them, a `ClusterIP` service is enough, being the *foreign cluster* a *virtual node* of the *local cluster*.

### Check the created resources and where the pods are running
When the GUI appears on the machine running the `cloudify` script, you can inspect the created resources by running the following commands:
```bash
kubectl get deployment -n kod    # This will show you the application deployment (blender in this example)
kubectl get jobs -n kod          # This will show you the vncviewer job
kubectl get secrets -n kod       # This will show you the secret containing the ssh key
kubectl get pod -n kod -o wide   # This will show you the running pods and which node they are scheduled on
```

The above commands can be executed in both the clusters, paying attention to the `namespace`.
In fact, the `kod` `namespace` will be reflected in the *foreign cluster* by adding a suffix as follows `kod-<...>`. So, to retrieve that `namespace`, execute the following in the *foreign cluster*:
```bash
kubectl get namespaces | grep 'kod-'
```

Now, you can execute all the `kubectl` commands listed above also in the *foreign cluster*, by replacing the `namespace` with the one obtained with the previous command.
In this case, you should see that only the `secret`, the `deployment` and the application `pod` (in this example blender) are present in this cluster.
This is because the other resources (related to vncviewer) are present only in the *local cluster*, where the GUI is actually displayed.

## Cleanup the KubernetesOnDesktop installation
To clean up the KubernetesOnDesktop installation you need to execute the following command:

```bash
sudo KUBECONFIG=$KUBECONFIG cloudify-uninstall
```

> **Note:** During the uninstall process, you will be asked whether to completely remove the `kod` namespace. Just type "yes" and then press "Enter" to complete the process.

## Teardown k3s and Liqo
To teardown k3s and Liqo just run the following commands on both nodes:

```bash
k3s-uninstall.sh
rm $HOME/.kube/k3s.yaml
```

> **Note**: If you added the `export KUBECONFIG` instruction to your bashrc to make the `KUBECONFIG` environment variable permanent, as described in [Install k3s](#install-k3s) section, then run also the following command:
> ```bash
> sed -i 's#export KUBECONFIG="$HOME/.kube/k3s.yaml"##' $HOME/.bashrc
> ```