# Secure Ingress with Cert-Manager

### Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Test](#test)
- [Ingress](#ingress)

## Introduction
By default, Istiod acts as a CA to authenticate mTLS and secure ingress connections. This document
provides guidance for integrating Istiod with [cert-manager][1] as an external CA.

## Prerequisites
- Helm
- jq

## Configuration

Follow the [Readme](../README.md) to create a cluster and then install cert-manager.
```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl create namespace cert-manager
helm install -n cert-manager cert-manager jetstack/cert-manager --set installCRDs=true
```

Verify the cert-manager deployments have successfully rolled-out.
```shell
for i in cert-manager cert-manager-cainjector cert-manager-webhook; \
do kubectl rollout status deploy/$i -n cert-manager; done
```

An [Issuer][2] must be created in the istio-system namespace to sign Istiod and mesh workload certificates.
We will use a `SelfSigned` Issuer, though any supported Issuer can be used.
```shell
kubectl create namespace istio-system
kubectl apply -n istio-system -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: istio-ca
spec:
  isCA: true
  duration: 2160h # 90d
  secretName: istio-ca
  commonName: istio-ca
  subject:
    organizations:
    - cert-manager
  issuerRef:
    name: selfsigned
    kind: Issuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: istio-ca
spec:
  ca:
    secretName: istio-ca
EOF
```

Verify the “istio-ca” and “selfsigned” issuers are present and report `READY=True`.
```shell
kubectl get issuers -n istio-system
```

The [istio-csr][3] project installs an agent that is responsible for verifying incoming certificate signing
requests from Istio mesh workloads, and signs them through cert-manager via a configured Issuer. Install istio-csr:
```shell
helm install -n cert-manager cert-manager-istio-csr jetstack/cert-manager-istio-csr
```

Verify the istio-csr deployment has successfully rolled-out.
```shell
kubectl rollout status deploy/cert-manager-istio-csr -n cert-manager
```

Certificates “istio-ca” and “istiod” should be present and report `READY=True`.
```shell
kubectl get certificates -n istio-system
```

Istio must be configured to use cert-manager as the CA server for both workload and Istio control plane components.
The following configuration uses the IstioOperator resource to install Istio with cert-manager integration.
Install Istio 1.12:
```shell
istioctl install -f ./manifests/istio-operator-certmanager-nodeport.yaml -y --verify
```
The installation should complete, and the Istio control plane should become in a ready state.

## Test

All workload certificates will now be requested through cert-manager using the configured Issuer. Let’s run an
example workload to test mTLS. First, create a namespace and configure it for automatic sidecar injection:
```shell
kubectl create ns foo
kubectl label ns/foo istio-injection=enabled
```

Run the sample sleep and httpbin workloads.
```shell
ISTIO_VERSION=1.11
kubectl apply -n foo -f https://raw.githubusercontent.com/istio/istio/release-$ISTIO_VERSION/samples/sleep/sleep.yaml
kubectl apply -n foo -f https://raw.githubusercontent.com/istio/istio/release-$ISTIO_VERSION/samples/httpbin/httpbin.yaml
```

Verify the sleep and httpbin deployments have successfully rolled-out.
```shell
for i in sleep httpbin; do kubectl rollout status -n foo deploy/$i; done
```

Verify the sidecar proxy was injected for each workload. Each workload pod should show 2/2 containers are `READY`.
```shell
for i in sleep httpbin; do kubectl get po -n foo -l app=$i; done
```

By default, Istio configures destination workloads using `PERMISSIVE` mode. This mode means a service
can accept both plain text and mTLS connections. To ensure mTLS is being used between sleep and httpbin
workloads, set the mode to `STRICT` for namespace “foo”.
```shell
kubectl apply -n foo -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

Test mTLS from the sleep pod to the httpbin pod.
```shell
kubectl -n foo exec -it deploy/sleep -c sleep -- curl -o /dev/null -s -w '%{http_code}\n' http://httpbin.foo:8000/headers
```

## Ingress

Istio Gateways support exposing secure HTTPS services using simple or mutual TLS.
cert-manager can be used to generate the gateway's certificate and key. To get started,
configure a Certificate that will be used to securely expose the httpbin workload.
The Certificate must be in the same namespace as the associated ingress gateway
deployment, e.g. `istio-system`.
```sh
kubectl apply -n istio-system -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: httpbin
spec:
  commonName: httpbin.example.com
  dnsNames:
  - httpbin.example.com
  issuerRef:
    group: cert-manager.io
    kind: Issuer
    name: istio-ca
  secretName: httpbin
EOF
```
__Note:__ If desired, a separate Issuer for ingress can be created by following the steps in the
Configuration section of this document.

Verify the Certificate is reporting `READY=True`.
```sh
kubectl get cert/httpbin -n istio-system
```

Create a Gateway and VirtualService for the httpbin workload.
```sh
kubectl apply -n foo -f - <<EOF
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: httpbin
spec:
  servers:
  - port:
      number: 443
      name: tls
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: httpbin
    hosts:
    - httpbin.example.com
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: httpbin.foo.svc.cluster.local
EOF
```
__Note:__ Mode `SIMPLE` secures connections with standard TLS semantics, i.e. the client
authenticates the gateway's certificate.

Copy the CA certificate used to establish a secure connection with the ingress gateway.
```sh
kubectl get secret/istio-ca -n istio-system -o json | jq -r '.data."ca.crt"' | base64 -D > ca.crt
```

Before connecting to the httpbin service, determine your [ingress gateway's IP address][1].
The `--cacert` flag is used to specify the CA root certificate used to authenticate the ingress
gateway's server certificate.
```sh
INGRESS_HOST=127.0.0.1
curl -o /dev/null -s -w '%{http_code}\n' --cacert ca.crt --resolve "httpbin.example.com:443:$INGRESS_HOST" "https://httpbin.example.com/status/200"
```
A `200` status code should be returned.

__Note:__ The `--resolve` flag can be removed if a DNS record exists for "httpbin.example.com".

[1]: https://cert-manager.io/
[2]: https://cert-manager.io/docs/configuration/
[3]: https://github.com/cert-manager/istio-csr
