---
title: "GKE Ingress with Let’s Encrypt using Cert-Manager"
layout: post
date: 2020-03-16 12:05
headerImage: false
tag:
- kubernetes
- google cloud
- ingress
- letsencrypt
- cert-manager
category: blog
author: kosyanyanwu
description: Setting up GKE Ingress with Let’s Encrypt certificate using Cert-Manager on a Google Kubernetes Cluster
---
## Introduction
[Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine) provides a built-in and managed Ingress controller called GKE Ingress. When you create an Ingress object, the GKE Ingress controller creates a Google Cloud HTTP(S) load balancer and configures it according to the information in the Ingress and its associated Services.

This article describes how to setup Ingress for External HTTP(S) Load Balancing, install cert-manager certificate provisioner and setup up a Let's Encrypt certificate. This was written based on GKE [v1.14.10-gke.17](https://cloud.google.com/kubernetes-engine/docs/release-notes-stable#february_11_2020), [cert-manager](https://cert-manager.io/) v0.13 and [Helm](https://helm.sh/) v3.

### Prerequisites
* [A GKE Kubernetes cluster](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster)
* [Helm](https://helm.sh/docs/intro/install/)
* [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [A global static IP](https://cloud.google.com/compute/docs/ip-addresses/reserve-static-external-ip-address) with [DNS configured](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip#step_4_configure_your_domain_name_records) for your domain for example, as example.your-domain.com. Regional IP addresses do not work with GKE Ingress.

Note that a Service exposed through an Ingress must respond to health checks from the load balancer. According to the [docs](https://cloud.google.com/kubernetes-engine/docs/concepts/ingress#health_checks), your app must either serve a response with an HTTP 200 status to GET requests on the / path, or you can configure an HTTP readiness probe, serving a response with an HTTP 200 status to GET requests on the path specified by the readiness probe.

### Create a deployment
Here is an example of a sample deployment manifest.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-deployment
  labels:
    app: sampleApp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sampleApp
  template:
    metadata:
      labels:
        app: sampleApp
    spec:
      containers:
      - name: sampleContainer
        image: nginx:1.7.9
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Create a service
Here is an example of a sample service manifest.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sampleApp-service
  labels:
    app: sampleApp
spec:
  type: NodePort
  selector:
    app: sampleApp
  ports:
    - name: http
      protocol: TCP
      port: 8080
      targetPort: 8080
```

## Install cert-manager
cert-manager runs within your Kubernetes cluster as a series of deployment resources. It utilizes [CustomResourceDefinitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to configure Certificate Authorities and request certificates. The following steps [installs cert-manager](https://cert-manager.io/docs/installation/kubernetes/) on your Kubernetes cluster.

* Install the CustomResourceDefinition resources separately.
    ```sh
    kubectl apply --validate=false -f https://raw.githubusercontent.com/jetstack/cert-manager/v0.13.1/deploy/manifests/00-crds.yaml
    ```

* Create the namespace for cert-manager.
    ```sh
    kubectl create namespace cert-manager
    ```

* Add the Jetstack Helm repository.
    ```sh
    helm repo add jetstack https://charts.jetstack.io
    ```

* Update your local Helm chart repository cache.
    ```sh
    helm repo update
    ```

* Install the cert-manager Helm chart.
    ```sh
    helm install \
      cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --version v0.13.1
    ```

* Verify the installation.
    ```sh
    $ kubectl get pods --namespace cert-manager
    NAME                                       READY   STATUS    RESTARTS   AGE
    cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
    cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
    cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
    ```
    You should see the cert-manager, cert-manager-cainjector, and cert-manager-webhook pod in a Running state. It may take a minute or so for the TLS assets required for the webhook to function to be provisioned.

* Create an [Issuer](https://cert-manager.io/docs/concepts/issuer/) to test the webhook works okay.
    ```yaml
    cat <<EOF > test-resources.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: cert-manager-test
    ---
    apiVersion: cert-manager.io/v1alpha2
    kind: Issuer
    metadata:
      name: test-selfsigned
      namespace: cert-manager-test
    spec:
      selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1alpha2
    kind: Certificate
    metadata:
      name: selfsigned-cert
      namespace: cert-manager-test
    spec:
      dnsNames:
        - example.com
      secretName: selfsigned-cert-tls
      issuerRef:
        name: test-selfsigned
    EOF
    ```

* Create the test resources.
    ```sh
    kubectl apply -f test-resources.yaml
    ```

* Check the status of the newly created certificate. You may need to wait a few seconds before cert-manager processes the certificate request.
    ```sh
    $ kubectl describe certificate -n cert-manager-test

    ...
    Spec:
      Common Name:  example.com
      Issuer Ref:
        Name:       test-selfsigned
      Secret Name:  selfsigned-cert-tls
    Status:
      Conditions:
        Last Transition Time:  2020-01-29T17:34:30Z
        Message:               Certificate is up to date and has not expired
        Reason:                Ready
        Status:                True
        Type:                  Ready
      Not After:               2020-04-29T17:34:29Z
    Events:
      Type    Reason      Age   From          Message
      ----    ------      ----  ----          -------
      Normal  CertIssued  4s    cert-manager  Certificate issued successfully
    ```

* Clean up the test resources.
    ```sh
    kubectl delete -f test-resources.yaml
    ```

If all the above steps have completed without error, you are good to go!

## Create issuer
The Let’s Encrypt production issuer has very strict [rate limits](https://letsencrypt.org/docs/rate-limits/). When you are experimenting and learning, it is very easy to hit those limits, and confuse rate limiting with errors in configuration or operation. Start with [Let’s Encrypt staging](https://letsencrypt.org/docs/staging-environment/) environment and switch to Let’s Encrypt production after it works fine. In this article, we will be creating a [ClusterIssuer](https://docs.cert-manager.io/en/release-0.11/reference/clusterissuers.html).

Create a clusterissuer definition and update the email address to your own. This email is required by Let’s Encrypt and used to notify you of certificate expiration and updates.
```yaml
cat <<EOF > clusterissuer.yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: you@youremail.com # Update to yours
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
            class: ingress-gce
EOF
```

Once edited, apply the custom resource:
```sh
kubectl apply -f clusterissuer.yaml
```

Check on the status of the clusterissuer after you create it:
```sh
$ kubectl describe clusterissuer letsencrypt-staging

Name:         letsencrypt-staging
Namespace:
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"cert-manager.io/v1alpha2","kind":"ClusterIssuer","metadata":{"annotations":{},"name":"letsencrypt-staging"},"spec":{"acme":...
API Version:  cert-manager.io/v1alpha2
Kind:         ClusterIssuer
Metadata:
  Creation Timestamp:  2020-02-24T18:33:55Z
  Generation:          1
  Resource Version:    1751518
  Self Link:           /apis/cert-manager.io/v1alpha2/clusterissuers/letsencrypt-staging
  UID:                 3699c43a-5734-11ea-8167-42010a840033
Spec:
  Acme:
    Email:  you@youremail.com
    Private Key Secret Ref:
      Name:  letsencrypt-staging
    Server:  https://acme-staging-v02.api.letsencrypt.org/directory
    Solvers:
      http01:
        Ingress:
          Class:  ingress-gce
Status:
  Acme:
    Last Registered Email:  you@youremail.com
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/123456
  Conditions:
    Last Transition Time:  2020-02-24T18:33:56Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

You should see the issuer listed with a registered account.

## Deploy a TLS Ingress Resource
Create an [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) definition.
```yaml
cat <<EOF > ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: sampleApp-ingress
  annotations:
    # specify the name of the global IP address resource to be associated with the HTTP(S) Load Balancer.
    kubernetes.io/ingress.global-static-ip-name: sampleApp-ip
    # add an annotation indicating the issuer to use.
    cert-manager.io/cluster-issuer: letsencrypt-staging
    # controls whether the ingress is modified ‘in-place’,
    # or a new one is created specifically for the HTTP01 challenge.
    acme.cert-manager.io/http01-edit-in-place: "true"
  labels:
    app: sampleApp
spec:
  tls: # < placing a host in the TLS config will indicate a certificate should be created
  - hosts:
    - example.example.com
    secretName: sampleApp-cert-secret # < cert-manager will store the created certificate in this secret
  rules:
  - host: example.example.com
    http:
      paths:
      - path: sample/app/path/*
        backend:
          serviceName: sampleApp-service
          servicePort: 8080
EOF
```

Once edited, apply ingress resource.
```sh
kubectl apply -f ingress.yaml
```

## Verify
View certificate.
```sh
$ kubectl get certificate
NAME                    READY     SECRET                AGE
sampleApp-cert-secret   True      sampleApp-cert-secret   6m34s
```

Describe certificate.
```sh
$ kubectl describe certificate sampleApp-cert-secret
Name:         sampleApp-cert-secret
Namespace:    default
Labels:        <none>
Annotations:  acme.cert-manager.io/http01-override-ingress-name: sampleApp-ingress
              cert-manager.io/issue-temporary-certificate: true
API Version:  cert-manager.io/v1alpha2
Kind:         Certificate
Metadata:
  Creation Timestamp:  2020-03-02T16:30:01Z
  Generation:          1
  Owner References:
    API Version:           extensions/v1beta1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  sampleApp-ingress
    UID:                   20911256-5231-11ea-8167-42010a80921
  Resource Version:        4941888
  Self Link:               /apis/cert-manager.io/v1alpha2/namespaces/default/certificates/sampleApp-cert-secret
  UID:                     2093fe67-5231-11ea-8167-42010a80921
Spec:
  Dns Names:
    example.example.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-staging
  Secret Name:  sampleApp-cert-secret
Status:
  Conditions:
    Last Transition Time:  2020-03-02T16:30:01Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2020-05-24T17:55:46Z
Events:                    <none>
```

Describe secrets created by cert manager.
```sh
$ kubectl describe secret sampleApp-cert-secret

Name:         sampleApp-cert-secret
Namespace:    default
Labels:       <none>
Annotations:  cert-manager.io/alt-names: example.example.com
              cert-manager.io/certificate-name: sampleApp-cert-secret
              cert-manager.io/common-name: example.example.com
              cert-manager.io/ip-sans:
              cert-manager.io/issuer-kind: ClusterIssuer
              cert-manager.io/issuer-name: letsencrypt-staging
              cert-manager.io/uri-sans:

Type:  kubernetes.io/tls

Data
====
ca.crt:   0 bytes
tls.crt:  3598 bytes
tls.key:  1675 bytes
```

## Switch to Let's Encrypt Prod
Now that we are sure that everything is configured correctly, you can update the annotations in the ingress to specify the production issuer:
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: sampleApp-ingress
  annotations:
    kubernetes.io/ingress.global-static-ip-name: sampleApp-ip
    cert-manager.io/cluster-issuer: letsencrypt-prod
    acme.cert-manager.io/http01-edit-in-place: "true"
  labels:
    app: sampleApp
spec:
  tls:
  - hosts:
    - example.example.com
    secretName: sampleApp-cert-secret
  rules:
  - host: example.example.com
    http:
      paths:
      - path: sample/app/path/*
        backend:
          serviceName: sampleApp-service
          servicePort: 8080
```
```sh
$ kubectl create --edit -f ingress.yaml
ingress.extensions "sampleApp-ingress" configured
```

You will also need to delete the existing secret, which cert-manager is watching. This will cause it to reprocess the request with the updated issuer.
```sh
$ kubectl delete secret sampleApp-cert-secret
secret "sampleApp-cert-secret" deleted
```

This will start the process to get a new certificate. Use `describe` to see the status.
```sh
kubectl describe certificate sampleApp-cert-secret
```

You can see the current state of the ACME Order by running `kubectl describe` on the [Order resource](https://docs.cert-manager.io/en/release-0.11/reference/orders.html) that cert-manager has created for your Certificate:
```sh
$ kubectl describe order sampleApp-cert-secret-889745041
...
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  Created     90s   cert-manager  Created Challenge resource "sampleApp-cert-secret-889745041-0" for domain "example.example.com"
```

You can `describe` the challenge to see the status of events by doing:
```sh
kubectl describe challenge sampleApp-cert-secret-889745041-0
```

Once the challenge(s) have been completed, their corresponding challenge resources will be deleted, and the ‘Order’ will be updated to reflect the new state of the Order. You can `describe` order to verify this.

Finally, the ‘Certificate’ resource will be updated to reflect the state of the issuance process. 'describe' the Certificate and verify that the status is `true` and `type` and `reason` are `ready`.
