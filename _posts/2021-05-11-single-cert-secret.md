---
title: Syncing cert-manager certificate secret across Namespaces using kubed
layout: post
date: 2021-05-11 12:05
headerImage: false
tag:
- kubernetes
- cert-manager
- ingress
- letsencrypt
category: blog
author: kosyanyanwu
description: Using same cert-manager certificate secret across Namespaces to prevent creating duplicates
---
## Introduction
[cert-manager](https://cert-manager.io/docs/) is a native Kubernetes certificate management controller. It can help with issuing certificates from a variety of sources, such as [Let’s Encrypt](https://letsencrypt.org/).

A common use-case for cert-manager is requesting TLS signed certificates to secure your ingress resources. This can be done by simply adding this annotation `cert-manager.io/cluster-issuer: nameOfClusterIssuer` to your Ingress resources and cert-manager will facilitate creating the Certificate resource for you.

When you have multiple ingresses in separate namespaces, this annotation ends up creating certificate resources in each of the namespaces. If you use Let's encrypt, the limit of [Duplicate Certificates](https://letsencrypt.org/docs/rate-limits/) is 5 per week, and it is easy to hit this limit if you have a lot of namespaces. You can decide to create the TLS secret for the certificate in one namespace and copy manually across all namespaces, but that will be a pain to manage and maintain. [cert-manager docs](https://cert-manager.io/docs/faq/kubed/) recommends using [kubed](https://appscode.com/products/kubed/v0.12.0/welcome/) with its [secret syncing feature](https://appscode.com/products/kubed/v0.12.0/guides/config-syncer/intra-cluster/) to solve this problem.

This article was written based on [cert-manager v1.3](https://cert-manager.io/docs/) and [kubed v0.12.0](https://appscode.com/products/kubed/v0.12.0/welcome/). This article does not address wildcard certificate usecase.

## Sync ingress certificate secret across namespaces using kubed
In order for the target Secret to be synced, the Secret resource must first be created with the correct annotations before the creation of the Certificate, else the Secret will need to be edited instead. When I tried this out, I created the Certificate first and edited the secret annotation afterwards. 

The example below creates a certificate resource in `sandbox` namespace. This in turn creates a secret `cert-secret`.
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: cert-secret
  namespace: sandbox
spec:
  secretName: cert-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - domain.example.com
```

Now apply the `kubed.appscode.com/sync: ""` annotation to the `cert-secret` created by the cert-manager Certificate resource. 

```sh
kubectl annotate secret cert-secret kubed.appscode.com/sync=""
```

This annotation will cause Kubed operator to copy the secret to all existing and new namespaces. If you want to synchronize this secret to some selected namespaces instead of all namespaces, you can do that by specifying namespace label-selector in the annotation. For example: `kubed.appscode.com/sync: "app=kubed"`.

You can now create your ingress resources to use the copied secret and you will no longer need this cert-manager annotation `cert-manager.io/cluster-issuer: nameOfClusterIssuer` in your ingress.
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    -  domain.example.com
    secretName: cert-secret
  rules:
  ...
```
