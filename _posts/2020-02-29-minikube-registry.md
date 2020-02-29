---
title: "Using local images with Minikube"
layout: post
date: 2020-02-29 12:05
headerImage: false
tag:
- minikube
- docker
- images
category: blog
author: kosyanyanwu
description: Use Minikube’s built-in Docker daemon as image registry with Kubernetes
---

## Introduction
[Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/) is a tool that makes it easy to run Kubernetes locally. Minikube runs a single-node Kubernetes cluster inside a Virtual Machine (VM) on your laptop for users looking to try out Kubernetes or develop with it day-to-day.

This article explains how you can use Minikube’s built-in Docker daemon without having to push images to a remote registry when trying out things locally, which speeds up local experiments.

## Steps
Start Minikube with:
```sh
minikube start
```

To be able to work with Minikube's docker daemon on your mac/linux host, use the `docker-env` command in your current shell:
```sh
eval $(minikube docker-env)
```

You can see containers in the Minikube registry with:
```sh
docker ps
```

Build your image with the `docker build` command.

On your deployment manifest, set `spec.containers.imagePullPolicy` to `Never`, otherwise kubernetes won't use the images you built locally.
```yaml
...
  spec:
    containers:
    - name: sampleApp
      image: sampleApp:0.0.1
      imagePullPolicy: Never
      ports:
      ...
```

When you do not wish to use Minikube docker registry, you can unset the Minikube variables from your shell with:
```sh
eval $(minikube docker-env -u)
```
