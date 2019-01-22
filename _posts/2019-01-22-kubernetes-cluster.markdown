---
title: "Using kubeadm to create a Kubernetes 1.13 cluster on VirtualBox with Ubuntu"
layout: post
date: 2019-01-22 20:36
headerImage: false
tag:
- kubernetes
- ubuntu
- kubeadm
- cluster
- virtualbox
category: blog
author: kosyanyanwu
description: Create a Kubernetes 1.13.2 cluster using kubeadm on VirtualBox virtual machines running Ubuntu 18.04 LTS.
---

## Introduction
[kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) is a toolkit that helps you bootstrap a best-practice Kubernetes cluster in an easy, reasonably secure and extensible way.

This article explains how you can use kubeadm to set up a Kubernetes cluster on three VirtualBox virtual machines (one master and two workers) running Ubuntu 18.04 LTS. To follow through with this article, you will need to have [Virtualbox](https://www.virtualbox.org/) and [Vagrant](https://www.vagrantup.com/) installed.

If you just want a quick setup without wanting to know how it is done, you can skip the rest of this article and use this [Github repo](https://github.com/kosyfrances/kubeclust).

## Create Virtual Machines
Let us set up 3 VMs, one being the master (kubemaster) and the remaining two being the workers (worker1 and worker2). All VMs will be set up with 2 CPUs and 2GB RAM each.  You can use this [Vagrantfile](https://github.com/kosyfrances/kubeclust/blob/master/Vagrantfile) to spin up the VMs by simply running `vagrant up`.

The hostnames and IP addresses of the machines are as follows:

**kubemaster** — 192.168.99.20

**worker1** – 192.168.99.21

**worker2** – 192.168.99.22


## Install Docker
The following steps should be done on all of the running VMs.

Install Docker v18.06 from Ubuntu’s repositories:
```
apt update
apt install -y docker.io=18.06.1-0ubuntu1~18.04.1
```
Create the `docker` group and add the `Vagrant` user:

```
sudo groupadd docker
sudo usermod -aG docker $USER
```

Log out and log back in so that your group membership is re-evaluated.

Configure docker to start on boot.
```
sudo systemctl enable docker
```

Refer to the [official Docker installation](https://docs.docker.com/install/linux/linux-postinstall/) guides for more information.

## Install kubeadm, kubelet and kubectl
Install these packages on all of the running VMs:

* **kubeadm**: the command to bootstrap the cluster.

* **kubelet**: the component that runs on all of the machines in your cluster and does things like starting pods and containers.

* **kubectl**: the command line util to talk to your cluster.

```
apt update && apt install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt update
apt install -y kubelet=1.13.2-00 kubeadm=1.13.2-00 kubectl=1.13.2-00
```

## Initialise Master node
On kubemaster, start kubelet:
```
systemctl daemon-reload
systemctl restart kubelet
```

This is where you decide what Container Network Interface (CNI) plugin to use as some of them require you to specify parameters when initialising the master node. In this article we will be using [Calico](https://docs.projectcalico.org/v3.1/getting-started/kubernetes/) and in order for Network Policy to work correctly with it, you need to pass `--pod-network-cidr=192.168.0.0/16` to `kubeadm init`. We also need to specify the advertise address of the API server because we have multiple network adapters on the host and we need it to advertise on the host-only network IP.

Now run
```
sudo kubeadm init --apiserver-advertise-address=192.168.99.20 --pod-network-cidr=192.168.0.0/16
```

After the initialisation finishes, make a record of the `kubeadm join` command that `kubeadm init` outputs. You will need it in a moment.

The [token](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/) is used for mutual authentication between the master and the joining nodes. The token included here is secret, keep it safe as anyone with this token can add authenticated nodes to your cluster. These tokens can be listed, created and deleted with the `kubeadm token` command.

To start using your cluster, you need to run the following as a regular user:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

You can now [deploy a pod network](https://kubernetes.io/docs/concepts/cluster-administration/addons/) to the cluster. Recall that we are using Calico, so let's run:
```
kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml
```

Confirm that the pod network is installed by running:
```
kubectl get pods --all-namespaces
```
You should get something like this:
```
vagrant@kubemaster:~$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   calico-node-6fkt5                    2/2     Running   0          67s
kube-system   calico-node-n972r                    2/2     Running   0          61s
kube-system   calico-node-v965s                    2/2     Running   0          61s
kube-system   coredns-86c58d9df4-9lb8f             1/1     Running   0          67s
kube-system   coredns-86c58d9df4-ht6f8             1/1     Running   0          67s
kube-system   etcd-kubemaster                      1/1     Running   0          14s
kube-system   kube-apiserver-kubemaster            1/1     Running   0          9s
kube-system   kube-controller-manager-kubemaster   1/1     Running   0          31s
kube-system   kube-proxy-vv78z                     1/1     Running   0          61s
kube-system   kube-proxy-x2mn9                     1/1     Running   0          61s
kube-system   kube-proxy-ztq8q                     1/1     Running   0          67s
kube-system   kube-scheduler-kubemaster            1/1     Running   0          21s
```

If your network is not working or kube-dns is not in the Running state, check out [kubeadm troubleshooting docs](https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/).


## Join cluster
Once the kube-dns pod is up and running, you can continue by joining your nodes. Remember the `kubeadm join` command that `kubeadm init` gave us ... Get that command and run on worker1 and worker2. It should be something like this:
```
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```
You can run `watch kubectl get nodes` on the kubemaster to see the worker machines join.

Eventually you should see something like this:
```
NAME         STATUS     ROLES    AGE   VERSION
kubemaster   Ready      master   54s   v1.13.2
worker1      Ready      <none>   27s   v1.13.2
worker2      Ready      <none>   27s   v1.13.2
```

## Tear down
To undo what kubeadm did, you should first [drain the node](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#drain) and make sure that the node is empty before shutting it down.

On kubemaster, with `<node name>` being `kubemaster`, `worker1`, and `worker2`, run:
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
