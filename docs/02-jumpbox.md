# Set Up The Jumpbox

In this lab you will set up one of the four machines to be a `jumpbox`. This machine will be used to run commands throughout this tutorial. While a dedicated machine is being used to ensure consistency, these commands can also be run from just about any machine including your personal workstation running macOS or Linux.

Think of the `jumpbox` as the administration machine that you will use as a home base when setting up your Kubernetes cluster from the ground up. Before we get started we need to install a few command line utilities and clone the Kubernetes The Hard Way git repository, which contains some additional configuration files that will be used to configure various Kubernetes components throughout this tutorial.

Log in to the `jumpbox`:

```bash
$ multipass shell jumpbox
JB$ sudo su
```

All commands will be run as the `root` user. This is being done for the sake of convenience, and will help reduce the number of commands required to set everything up.

### Install Command Line Utilities

Now that you are logged into the `jumpbox` machine as the `root` user, you will install the command line utilities that will be used to preform various tasks throughout the tutorial.

```bash
JB$ apt update
JB$ apt upgrade
JB$ reboot
$ multipass shell jumpbox
JB$ apt-get -y install wget curl vim openssl git
```

![note] From this point all shell commands will be assumed to be in the jumpbox VM.

### Sync GitHub Repository

Now it's time to download a copy of this tutorial which contains the configuration files and templates that will be used build your Kubernetes cluster from the ground up. Clone the Kubernetes The Hard Way git repository using the `git` command:

```bash
# don't do this in a mount from the host - you won't have permissions
git clone https://github.com/georgesims21/kubernetes-the-hard-way.git
```

Change into the `kubernetes-the-hard-way` directory:

```bash
cd kubernetes-the-hard-way
```

This will be the working directory for the rest of the tutorial. If you ever get lost run the `pwd` command to verify you are in the right directory when running commands on the `jumpbox`:

```bash
pwd
```

```text
/home/ubuntu/kubernetes-the-hard-way
```

### Download Binaries

In this section you will download the binaries for the various Kubernetes components. The binaries will be stored in the `downloads` directory on the `jumpbox`, which will reduce the amount of internet bandwidth required to complete this tutorial as we avoid downloading the binaries multiple times for each machine in our Kubernetes cluster.

The binaries that will be downloaded are listed in either the `downloads-amd64.txt` or `downloads-arm64.txt` file depending on your hardware architecture, which you can review using the `cat` command:

```bash
cat downloads-$(dpkg --print-architecture).txt
```

Going to go for the latest stable releases of the tools and k8s, the ones included are quite outdated.

#### What are we actually installing?

We will get to each one in more detail as we go along, but as an overview here are the components we will be installing during this phase. You can see how some of these tie together in the [k8s cluster architecture diagram](https://kubernetes.io/docs/concepts/architecture/):

- kubectl: used to communicate with your cluster. Requests from kubectl are sent to the cluster's Control Plane via the kube api. This is done via CLI commands which get translated into kubernetes API actions against your given cluster. Through the kubeconfig file you can communicate with multiple contexts with a single kubectl binary [tool ref](https://kubernetes.io/docs/reference/kubectl/) and [conceptual overview](https://kubernetes.io/docs/concepts/overview/kubectl/).

- kube-apiserver: the heart of the control plane. Allows communication to the cluster from inside and out. It exposes the HTTP API [tool ref](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/) and [conceptual overview](https://kubernetes.io/docs/concepts/overview/kubernetes-api/).

- kube-controller-manager: If the API server is the heart, the controller manager would be the pulse. The controller manager runs a loop to reconcile and keep the cluster in the desired state via the replication controller, namespace controller and so on. These loops are done through the api server [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/).

- kube-scheduler: The sole job for the scheduler is to find a node for a pod to run on, based on any constraints defined in the spec. The scheduler collects a list of 'feasible' nodes for a pod and then based on underlying logic scores each one and selects the one with the best score [kube-scheduler](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/).

- kube-proxy: Deployed to nodes to allow requests to be forwarded to the pods on that node via Service objects. This isn't a required tool as it can be replaced by a network plugin [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/). 

- kubelet: Another agent which is deployed to each node in the cluster, it's job is to ensure containers defined in the PodSpecs are running and healthy. That means only kubernetes-defined containers are managed by the kubelet. The kubelet requires a Container Runtime Interface (CRI) plugin to be installed so that there is a working Container Runtime on each node, and the kubelet communicates with the CRI for any container actions [tool ref](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/).

- crictl: a CLI tool used to help debug the CRI on the kubelet [crictl](https://github.com/kubernetes-sigs/cri-tools).

- containerd: a simple container runtime, managing things like image transfer/storage, container execution and network attachments. This is what the kubelet will communicate with via the CRI [containerd](https://github.com/containerd/containerd/tree/main).

- runc: The Open Container Initiative (OCI) was created to standardize container formats and runtimes. runC is a CLI tool which allows spawning and running of containers according to the OCI spec. This was donated by Docker to the Linux Foundation. [OCI](https://opencontainers.org/about/overview/) and [runC](https://github.com/opencontainers/runc/).

- cni-plugins: Linux-based networking plugins to be used by the kubelet [cni-plugins](https://github.com/containernetworking/plugins/).

- etcd: At it's core etcd is a distributed key-value store. In the context of Kubernetes we will use etcd to store all cluster data [etcd](https://github.com/etcd-io/etcd/) and [kubernetes docs](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/).

#### Checking compatibility of the versions

Given that I want to use more recent releases than the original fork, there were some tools which required checking compatibility for.

- containerd and runc: containerd have a file which they use on each release to mark which runc version (and above) is compatible. For example if I wanted to use the v2.2.7 of containerd I would check the [runc version file](https://github.com/containerd/containerd/blob/v2.2.7/script/setup/runc-version) which in this case is >v1.3.6.

Download the binaries into a directory called `downloads` using the `wget` command:

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-$(dpkg --print-architecture).txt
```

Depending on your internet connection speed it may take a while to download over `500` megabytes of binaries, and once the download is complete, you can list them using the `ls` command:

```bash
ls -oh downloads
```

Extract the component binaries from the release archives and organize them under the `downloads` directory.

```bash
{
  ARCH=$(dpkg --print-architecture)
  mkdir -p downloads/{client,cni-plugins,controller,worker}
  tar -xvf downloads/crictl-v1.36.0-linux-${ARCH}.tar.gz \
    -C downloads/worker/
  tar -xvf downloads/containerd-2.2.7-linux-${ARCH}.tar.gz \
    --strip-components 1 \
    -C downloads/worker/
  tar -xvf downloads/cni-plugins-linux-${ARCH}-v1.9.1.tgz \
    -C downloads/cni-plugins/
  tar -xvf downloads/etcd-v3.6.14-linux-${ARCH}.tar.gz \
    -C downloads/ \
    --strip-components 1 \
    etcd-v3.6.14-linux-${ARCH}/etcdctl \
    etcd-v3.6.14-linux-${ARCH}/etcd
  mv downloads/{etcdctl,kubectl} downloads/client/
  mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
    downloads/controller/
  mv downloads/{kubelet,kube-proxy} downloads/worker/
  mv downloads/runc.${ARCH} downloads/worker/runc
}
```

```bash
rm -rf downloads/*gz
```

Make the binaries executable.

```bash
chmod +x downloads/{client,cni-plugins,controller,worker}/*
```

### Install kubectl

In this section you will install the `kubectl`, the official Kubernetes client command line tool, on the `jumpbox` machine. `kubectl` will be used to interact with the Kubernetes control plane once your cluster is provisioned later in this tutorial.

Use the `chmod` command to make the `kubectl` binary executable and move it to the `/usr/local/bin/` directory:

```bash
cp downloads/client/kubectl /usr/local/bin/
```

At this point `kubectl` is installed and can be verified by running the `kubectl` command:

```bash
kubectl version --client
```

```text
Client Version: v1.36.4
Kustomize Version: v5.8.1
```

At this point the `jumpbox` has been set up with all the command line tools and utilities necessary to complete the labs in this tutorial.

Next: [Provisioning Compute Resources](03-compute-resources.md)
