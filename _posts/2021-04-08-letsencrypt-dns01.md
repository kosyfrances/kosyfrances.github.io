---
title: "DNS01 Challenge Provider for Let’s Encrypt Issuer using Google CloudDNS"
layout: post
date: 2021-04-08 12:05
headerImage: false
tag:
- kubernetes
- google cloud
- ingress
- letsencrypt
- cert-manager
category: blog
author: kosyanyanwu
description: Configuring DNS01 Challenge Provider using Google CloudDNS for Let's Encrypt issuer setup with cert-manager
---
## Introduction
This article explains how to set up a ClusterIssuer to use Google CloudDNS to solve [DNS01 ACME challenge](https://letsencrypt.org/docs/challenge-types/#dns-01-challenge). It assumes that your cluster is hosted on Google Cloud Platform (GCP) and that you already have a [domain set up with CloudDNS](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip#step_4_configure_your_domain_name_records). It also assumes that you have [cert-manager installed](/ingress-gce-letsencrypt/#install-cert-manager) on your cluster.

## Create a Service Account
Create a [service account](https://cloud.google.com/iam/docs/creating-managing-service-accounts#creating) with `dns.admin` role. This is required for cert-manager to be able to add records to CloudDNS in order to solve the DNS01 challenge.

## Use Service Account
To use this service account, you can either [create a service account secret](#create-a-service-account-secret), or use [GKE workload identity](#gke-workload-identity).

### Create a Service Account secret
To access the service account you created in the previous step, cert-manager uses a key stored in a Kubernetes Secret. First, create a key for the service account and download it as a JSON file, then [create a Secret](https://kubernetes.io/docs/concepts/configuration/secret/#creating-a-secret-using-kubectl) from this file.

```sh
$ gcloud iam service-accounts keys create service-account.json \
  --iam-account dns01-solver@$PROJECT_ID.iam.gserviceaccount.com

$ kubectl create secret generic clouddns-dns01-solver-svc-acct \
  --from-file=service-account.json
```

## Create ClusterIssuer (using Service Account secret setup)
Here is a sample manifest
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: you@youremail.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        clouddns:
          # The ID of the GCP project
          project: $PROJECT_ID
          # Secret key reference to the service account created above
          serviceAccountSecretRef:
            name: clouddns-dns01-solver-svc-acct
            key: service-account.json
```

### GKE Workload Identity
If your GKE cluster already has [workload identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) enabled, you can leverage workload identity to avoid creating and managing static service account credentials. Follow these steps to do this:
* Link Kubernetes Service Account to Google Service Account in GCP:

  If you followed the [standard methods for deploying cert-manger to Kubernetes](https://cert-manager.io/docs/installation/kubernetes/), run the following command:
  ```sh
  $ gcloud iam service-accounts add-iam-policy-binding \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[cert-manager/cert-manager]" \
    dns01-solver@$PROJECT_ID.iam.gserviceaccount.com
  ```

  If your cert-manager pods are running under a different service account, replace goog[cert-manager/cert-manager] with goog[NAMESPACE/SERVICE_ACCOUNT], where NAMESPACE is the namespace of the service account and SERVICE_ACCOUNT is the name of the service account.

* Link Kubernetes Service Account to Google Service Account in Kubernetes:
  
  To do this, add the proper workload identity annotation to the cert-manager service account.
  ```sh
  $ kubectl annotate serviceaccount --namespace=cert-manager cert-manager \
    "iam.gke.io/gcp-service-account=dns01-solver@$PROJECT_ID.iam.gserviceaccount.com"
  ```

  If your cert-manager pods are running under a different service account, replace --namespace=cert-manager cert-manager with --namespace=NAMESPACE SERVICE_ACCOUNT, where NAMESPACE is the namespace of the service account and SERVICE_ACCOUNT is the name of the service account.

## Create ClusterIssuer (using Workload Identity setup)
Here is a sample manifest
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: you@youremail.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - dns01:
        clouddns:
          # The ID of the GCP project
          project: $PROJECT_ID
```
Note that the issuer does not include a serviceAccountSecretRef property. Excluding this instructs cert-manager to use the default credentials provided by GKE workload identity.

## Verify

View the clusterissuer object:

```sh
$ kubectl get clusterissuer
NAME               READY   AGE
letsencrypt-prod   True    9s

$ kubectl describe clusterissuer
Name:         letsencrypt-prod
...
Status:
  Acme:
    Last Registered Email:  you@youremail.com
    Uri:                    https://acme-v02.api.letsencrypt.org/acme/acct/12345678
  Conditions:
    Last Transition Time:  2020-03-18T15:05:26Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

## Create test certificate
After successful creation of the ClusterIssuer, you can create a test Certificate to verify that everything works.
Here is a sample manifest:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    # The issuer created previously
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
```

It may take a few minutes for the certificate to be issued. While you are waiting, you can view the Certificate object and the associated resources being created:
```sh
$ kubectl get certificate
NAME          READY   SECRET            AGE
example-com   False   example-com-tls   65s
```

```sh
$ kubectl describe certificate
Name:         example-com
...
Status:
  Conditions:
    Last Transition Time:  2020-03-18T15:19:42Z
    Message:               Waiting for CertificateRequest "example-com-123456787" to complete
    Reason:                InProgress
    Status:                False
    Type:                  Ready
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Requested  71s   cert-manager  Created new CertificateRequest resource "example-com-123456787"
```

```sh
$ kubectl get certificaterequest
NAME                     READY   AGE
example-com-123456787   False   88s
```

```sh
$ kubectl describe certificaterequest example-com-123456787
Name:         example-com-123456787
...
Status:
  Conditions:
    Last Transition Time:  2020-03-18T15:19:42Z
    Message:               Waiting on certificate issuance from order default/example-com-123456787-123456787: "pending"
    Reason:                Pending
    Status:                False
    Type:                  Ready
Events:
  Type    Reason        Age   From          Message
  ----    ------        ----  ----          -------
  Normal  OrderCreated  96s   cert-manager  Created Order resource default/example-com-123456787-123456787
```

```sh
$ kubectl get order
NAME                                STATE     AGE
example-com-123456787-123456787   pending   114s
```

```sh
$ kubectl describe order example-com-123456787-123456787
Name:         example-com-123456787-123456787
...
Events:
  Type    Reason   Age   From          Message
  ----    ------   ----  ----          -------
  Normal  Created  118s  cert-manager  Created Challenge resource "example-com-123456787-123456787-123456787" for domain "example.com"
```

From the `order` event, we can see that the challenge resource has been created. When the order creation is complete, you will see this in its event.
```sh
$ kubectl describe order example-com-123456787-123456787
...
Events:
Type    Reason   Age   From          Message
----    ------   ----  ----          -------
...
Normal  Complete  4m2s   cert-manager  Order completed successfully
```

The state of `order` should change to `valid`.
```sh
$ kubectl get order
NAME                                STATE   AGE
example-com-123456787-123456787   valid   7m59s
```

CertificateRequest should have `READY` status as `True`.
```sh
$ kubectl get CertificateRequest
NAME                     READY   AGE
example-com-123456787   True    8m14s
```

```sh
$ kubectl describe CertificateRequest
...
Events:
  Type    Reason             Age    From          Message
  ----    ------             ----   ----          -------
  Normal  OrderCreated       8m37s  cert-manager  Created Order resource default/example-com-123456787-123456787
  Normal  CertificateIssued  5m49s  cert-manager  Certificate fetched from issuer successfully
```

Certificate should also have `READY` status as `True`.
```sh
$ kubectl get certificate
NAME          READY   SECRET            AGE
example-com   True    example-com-tls   9m53s
```

```sh
$ kubectl describe certificate example-com
...
Events:
  Type    Reason     Age    From          Message
  ----    ------     ----   ----          -------
  Normal  Requested  10m    cert-manager  Created new CertificateRequest resource "example-com-123456787"
  Normal  Issued     7m38s  cert-manager  Certificate issued successfully
```
