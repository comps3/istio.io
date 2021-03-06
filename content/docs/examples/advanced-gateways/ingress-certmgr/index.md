---
title: Securing Kubernetes Ingress with Cert-Manager
description: Demonstrates how to obtain Let's Encrypt TLS certificates for Kubernetes Ingress automatically using Cert-Manager.
weight: 70
keywords: [traffic-management,ingress,https,cert-manager,acme,sds]
---

This example demonstrates the use of Istio as a secure Kubernetes Ingress controller with TLS certificates issued by [Let's Encrypt](https://letsencrypt.org/). While more powerful Istio concepts such as [gateway](/docs/reference/config/networking/v1alpha3/gateway) and [virtual service](/docs/reference/config/networking/v1alpha3/virtual-service) should be used for advanced traffic management, optional support of the Kubernetes Ingress is also available and can be used to simplify integration of legacy and third-party solutions into a service mesh and benefit from extensive telemetry and tracing capabilities that Istio provides.

You will start with a clean Istio installation, create an example service, expose it using the Kubernetes `Ingress` resource and get it secured by instructing cert-manager (bundled with Istio) to manage issuance and renewal of TLS certificates that will be further delivered to the Istio ingress [gateway](/docs/reference/config/networking/v1alpha3/gateway) and hot-swapped as necessary via the means of [Secrets Discovery Service (SDS)](https://www.envoyproxy.io/docs/envoy/latest/configuration/secret).

## Before you begin

[Install Istio](/docs/setup/) making sure to enable ingress [gateway](/docs/reference/config/networking/v1alpha3/gateway) with Kubernetes Ingress support, [SDS](https://www.envoyproxy.io/docs/envoy/latest/configuration/secret) and [cert-manager](https://docs.cert-manager.io/) optional dependency during installation. Here's an example of how to do this for the [helm template](/docs/setup/kubernetes/install/helm/#option-1-install-with-helm-via-helm-template) installation path:

{{< text bash >}}
$ helm template $HOME/istio-fetch/istio \
  --namespace=istio-system \
  --set gateways.istio-ingressgateway.sds.enabled=true \
  --set global.k8sIngress.enabled=true \
  --set global.k8sIngress.enableHttps=true \
  --set global.k8sIngress.gatewayName=ingressgateway \
  --set certmanager.enabled=true \
  --set certmanager.email=mailbox@donotuseexample.com \
  > $HOME/istio-fetch/istio.yaml
{{< /text >}}

{{< tip >}}
By default `istio-ingressgateway` will be exposed as a `LoadBalancer` service type. You may want to change that by setting the `gateways.istio-ingressgateway.type` installation option to `NodePort` if this is more applicable to your Kubernetes environment.
{{< /tip >}}

## Configuring DNS name and gateway

Take a note of the external IP address of the `istio-ingressgateway` service:

{{< text bash >}}
$ kubectl -n istio-system get service istio-ingressgateway
{{< /text >}}

Configure your DNS zone so that the domain you'd like to use for this example is resolving to the external IP address of `istio-ingressgateway` service that you've captured in the previous step. You will need a real domain name for this example in order to get a TLS certificate issued. Let's store the configured domain name into an environment variable for further use:

{{< text bash >}}
$ INGRESS_DOMAIN=mysubdomain.mydomain.edu
{{< /text >}}

Your Istio installation contains an automatically generated [gateway](/docs/reference/config/networking/v1alpha3/gateway) resource configured to serve the routes defined by the Kubernetes `Ingress` resources. By default it does not use [SDS](https://www.envoyproxy.io/docs/envoy/latest/configuration/secret), so you need to modify it in order to enable the delivery of the TLS certificates to the `istio-ingressgateway` via [SDS](https://www.envoyproxy.io/docs/envoy/latest/configuration/secret):

{{< text bash >}}
$ kubectl -n istio-system edit gateway
{{< /text >}}

...and modify the `tls` section corresponding to the `https-default` port as follows:

{{< text bash >}}
$ kubectl -n istio-system \
  patch gateway istio-autogenerated-k8s-ingress --type=json \
  -p='[{"op": "replace", "path": "/spec/servers/1/tls", "value": {"credentialName": "ingress-cert-staging", "mode": "SIMPLE", "privateKey": "sds", "serverCertificate": "sds"}}]'
{{< /text >}}

Now it's time to setup a demo application.

## Setting up a demo application

You will be using a simple `helloworld` application for this example. The following command will spin up the `Deployment` and `Service` for the demo application and expose the service using an `Ingress` resource that will be handled by `istio-ingressgateway`.

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
spec:
  ports:
  - port: 5000
    name: http
  selector:
    app: helloworld
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: helloworld
spec:
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
      - name: helloworld
        image: istio/examples-helloworld-v1
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: istio
  name: helloworld-ingress
spec:
  rules:
    - host: "$INGRESS_DOMAIN"
      http:
        paths:
          - path: /hello
            backend:
              serviceName: helloworld
              servicePort: 5000
---
EOF
{{< /text >}}

{{< tip >}}
Notice use of the `INGRESS_DOMAIN` variable you defined earlier
{{< /tip >}}

Now you should be able to access your demo application via HTTP:

{{< text bash >}}
$ curl http://$INGRESS_DOMAIN/hello
Hello version: v1, instance: helloworld-5d498979b6-jp2mf
{{< /text >}}

HTTPS access still won't work as you don't have any TLS certificates. Let's fix that.

## Getting a Let's Encrypt certificate issued using cert-manager

At this point your Istio installation should have cert-manager up and running with two `ClusterIssuer` resources configured (for production and staging ACME-endpoints provided by [Let's Encrypt](https://letsencrypt.org/)). You will be using staging endpoint for this example (feel free to try swapping `letsencrypt-staging` for `letsencrypt` to get a browser-trusted certificate issued).

In order to have a certificate issued and managed by cert-manager you need to create a `Certificate` resource:

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: ingress-cert-staging
  namespace: istio-system
spec:
  secretName: ingress-cert-staging
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  commonName: $INGRESS_DOMAIN
  dnsNames:
  - $INGRESS_DOMAIN
  acme:
    config:
    - http01:
        ingressClass: istio
      domains:
      - $INGRESS_DOMAIN
---
EOF
{{< /text >}}

Notice that the `secretName` matches the `credentialName` attribute value that you previously used while configuring the [gateway](/docs/reference/config/networking/v1alpha3/gateway) resource. The `Certificate` resource will be processed by cert-manager and a new certificate will eventually be issued. Consult the status of the `Certificate` resource to check the progress:

{{< text bash >}}
$ kubectl -n istio-system describe certificate ingress-cert-staging
-> status should eventually flip to 'Certificate issued successfully'
{{< /text >}}

At this point the service should become available over HTTPS as well:

{{< text bash >}}
$ curl --insecure https://$INGRESS_DOMAIN/hello
Hello version: v1, instance: helloworld-5d498979b6-jp2mf
{{< /text >}}

Note that you have to use the `--insecure` flag as certificates issued by the "staging" ACME-endpoints aren't trusted.
