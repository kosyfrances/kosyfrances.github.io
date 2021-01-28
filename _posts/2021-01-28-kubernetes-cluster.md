---
title: "Using kubeadm to create a Kubernetes 1.20 cluster on VirtualBox with Ubuntu"
layout: post
date: 2021-01-28 20:36
headerImage: false
tag:
- kubernetes
- ubuntu
- kubeadm
- cluster
- virtualbox
category: blog
author: kosyanyanwu
description: Create a Kubernetes 1.20 cluster using kubeadm on VirtualBox virtual machines running Ubuntu 20.04 LTS.
---

## Introduction
[kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is a tool that helps you bootstrap a best-practice Kubernetes cluster in an easy, reasonably secure and extensible way.

This article explains how you can use kubeadm to set up a Kubernetes cluster on three VirtualBox virtual machines (one master and two workers) running Ubuntu 20.04 LTS. To follow through with this article, you will need to have [Virtualbox](https://www.virtualbox.org/) and [Vagrant](https://www.vagrantup.com/) installed.

If you just want a quick setup without wanting to know how it is done, you can skip the rest of this article and use this [Github repo](https://github.com/kosyfrances/kubeclust).

## Create Virtual Machines
Let us set up 3 VMs, one being the master (kubemaster) and the remaining two being the workers (worker1 and worker2). All VMs will be set up with 2 CPUs and 2GB RAM each.  You can use this [Vagrantfile](https://github.com/kosyfrances/kubeclust/blob/master/Vagrantfile) to spin up the VMs by simply running `vagrant up`.

The hostnames and IP addresses of the machines are as follows:

**kubemaster** — 192.168.99.20

**worker1** – 192.168.99.21

**worker2** – 192.168.99.22


## Install Docker
The following steps should be done on all of the running VMs.

Install Docker v19.03 from Ubuntu’s repositories:
```sh
sudo apt-get update
sudo apt-get install containerd=1.3.3-0ubuntu2 # because docker.io depends on it
sudo apt-get install -y docker.io=19.03.8-0ubuntu1.20.04.1
```
Create the `docker` group and add the `Vagrant` user:

```sh
sudo groupadd docker
sudo usermod -aG docker $USER
```

Log out and log back in so that your group membership is re-evaluated.

Configure docker to start on boot.
```sh
sudo systemctl enable docker
```

Refer to the [official Docker installation](https://docs.docker.com/install/linux/linux-postinstall/) guides for more information.

## Install kubeadm, kubelet and kubectl
Install these packages on all of the running VMs:

* **kubeadm**: the command to bootstrap the cluster.

* **kubelet**: the component that runs on all of the machines in your cluster and does things like starting pods and containers.

* **kubectl**: the command line util to talk to your cluster.

```sh
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet=1.20.2-00 kubeadm=1.20.2-00 kubectl=1.20.2-00
sudo apt-mark hold kubelet kubeadm kubectl
```

## Initialise Master node
This is where you decide what Container Network Interface (CNI) plugin to use as some of them require you to specify parameters when initialising the master node.

In this article we will be using [Calico](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/) as our network plugin.

When initialising kubeadm, the value given to it must match the value of `CALICO_IPV4POOL_CIDR` in the [calico manifest](https://docs.projectcalico.org/manifests/calico.yaml). This is required for Network Policy to work correctly with the calico plugin. As at the time of writing this article, the default value is `192.168.0.0/16`.

We also need to specify the advertise address of the API server (our master node) because we have multiple network adapters on the host and we need it to advertise on the host-only network IP.

Now run
```sh
kubeadm init --apiserver-advertise-address=192.168.99.20 --pod-network-cidr=192.168.0.0/16
```

After the initialisation finishes, make a record of the `kubeadm join` command that `kubeadm init` outputs. You will need it in a moment.

The [token](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/) is used for mutual authentication between the master and the joining nodes. The token included here is secret, keep it safe as anyone with this token can add authenticated nodes to your cluster. These tokens can be listed, created and deleted with the `kubeadm token` command.

To start using your cluster, you need to run the following as a regular user:
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Deploy pod network
You can now [deploy a pod network](https://kubernetes.io/docs/concepts/cluster-administration/addons/) (Calico in our case) to the cluster. Deploying Calico is as easy as applying the [manifest](https://docs.projectcalico.org/manifests/calico.yaml). On the master node, download the manifest and apply.
```sh
wget https://docs.projectcalico.org/manifests/calico.yaml

kubectl apply -f calico.yaml
```

Confirm that the pod network is installed by running:
```sh
kubectl get pods --all-namespaces
```

You should get something like this:
```sh
vagrant@kubemaster:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-744cfdf676-8npmm   1/1     Running   0          3m56s
kube-system   calico-node-7v48f                          1/1     Running   0          3m56s
kube-system   coredns-74ff55c5b-lh5xs                    1/1     Running   0          11m
kube-system   coredns-74ff55c5b-tmq8g                    1/1     Running   0          11m
kube-system   etcd-kubemaster                            1/1     Running   0          11m
kube-system   kube-apiserver-kubemaster                  1/1     Running   0          11m
kube-system   kube-controller-manager-kubemaster         1/1     Running   0          11m
kube-system   kube-proxy-dd9zv                           1/1     Running   0          11m
kube-system   kube-scheduler-kubemaster                  1/1     Running   0          11m
```

If there are issues, check out [kubeadm troubleshooting docs](https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/).


## Join cluster
Remember the `kubeadm join` command that `kubeadm init` gave us ... Get that command and run on worker1 and worker2 nodes. It should be something like this:
```
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```
You can run `watch kubectl get nodes` on the kubemaster to see the worker machines join.

Eventually you should see something like this:
```
NAME         STATUS   ROLES                  AGE     VERSION
kubemaster   Ready    control-plane,master   7m2s    v1.20.2
worker1      Ready    <none>                 6m33s   v1.20.2
worker2      Ready    <none>                 6m33s   v1.20.2
```

## Tear down
 if you want to deprovision your cluster more cleanly, you should first [drain the node](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain) and then deconfigure the node.

On kubemaster, with `<node name>` being `kubemaster`, `worker1` and `worker2`, run:
```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
```
After draining all nodes, reset all kubeadm installed state:
```
sudo kubeadm reset
```
If you wish to start over simply run `kubeadm init` or `kubeadm join` with the appropriate arguments.

More options and information about the [kubeadm reset](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/) command.
