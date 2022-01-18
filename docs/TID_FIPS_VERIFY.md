# TID FIPS Verification

### Table of Contents

- [Introduction](#introduction)
- [Configuration](#configuration)
- [Verify](#test)

## Introduction
Tetrate provides an Istio Distribution [(TID)][1] that is [FIPS 140-2][2] compliant. This document provides steps for
installing and verifying TID FIPS.

## Configuration

Follow the [Readme](../README.md) to create a cluster and then install Istio:
```shell
istioctl install -f ./manifests/istio-operator-tetrate-nodeport.yaml -y --verify
```
The installation should complete, and the Istio control plane should be in a ready state.

Setup the default namespace for automatic sidecar injection:
```shell
kubectl label ns/default istio-injection=enabled
```

## Verify
Deploy a test client and server workload:
```shell
ISTIO_VERSION=1.12
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-$ISTIO_VERSION/samples/sleep/sleep.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-$ISTIO_VERSION/samples/httpbin/httpbin.yaml
```

Verify the httpbin workload is using the TID FIPS image:
```shell
HTTPBIN=$(kubectl get po -l app=httpbin | grep httpbin | cut -d ' ' -f 1)
kubectl get po/$HTTPBIN -o yaml | grep 1.12.1-tetratefips-v1
```
Repeat the step above for the client workload.

Verify the build of the server workload sidecar proxy, e.g. Envoy:
```shell
kubectl exec -it po/$HTTPBIN -c istio-proxy -- envoy --version | cut -f4 -d" "
```
The output should contain `BoringSSL-FIPS` to indicate Envoy was built using the [BoringSSL FIPS library][3]. Repeat the
step above for the client workload.

Verify the control plane build:
```shell
kubectl cp -c istio-proxy default/$HTTPBIN:/usr/local/bin/pilot-agent pilot-agent
chmod +x pilot-agent
go version pilot-agent | cut -f2 -d" "
```
The go version from the output should include a `b` indicating it was built with BoringSSL.

By default, Istio configures destination workloads using `PERMISSIVE` mode. This mode means a service
can accept both plain text and mTLS connections. To ensure mTLS is being used between sleep and httpbin
workloads, set the mode to `STRICT` for namespace “foo”.
```shell
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

Test mTLS between the workload pods.
```shell
kubectl exec -it deploy/sleep -c sleep -- curl -o /dev/null -s -w '%{http_code}\n' http://httpbin.default:8000/headers
```
A `200` status code should be returned.

Use the [testssl.sh][4] tool to review TLS settings for FIPS 140-2 compliance.
```shell
kubectl create -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: testssl
  namespace: default
spec:
  containers:
  - name: app
    image: drwetter/testssl.sh
    command: [ "/bin/sh", "-c", "--" ]
    args: [ "while true; do sleep 3000; done;" ]
EOF
```

Run the testssl.sh tool against the httpbin workload:
```shell
$ kubectl exec -ti testssl -- testssl.sh https://httpbin.default.svc.cluster.local.:8000/headers

###########################################################
    testssl.sh       3.1dev from https://testssl.sh/dev/

      This program is free software. Distribution and
             modification under GPLv2 permitted.
      USAGE w/o ANY WARRANTY. USE IT AT YOUR OWN RISK!

       Please file bugs @ https://testssl.sh/bugs/

###########################################################

 Using "OpenSSL 1.0.2-chacha (1.0.2k-dev)" [~183 ciphers]
 on testssl:/home/testssl/bin/openssl.Linux.x86_64
 (built: "Jan 18 17:12:17 2019", platform: "linux-x86_64")


 Start 2022-01-12 17:06:50        -->> 10.96.79.160:8000 (httpbin.default.svc.cluster.local.) <<--

 rDNS (10.96.79.160):    httpbin.default.svc.cluster.local.
 Service detected:       certificate-based authentication => skipping all HTTP checks


 Testing protocols via sockets except NPN+ALPN

 SSLv2      not offered (OK)
 SSLv3      not offered (OK)
 TLS 1      not offered
 TLS 1.1    not offered
 TLS 1.2    offered (OK)
 TLS 1.3    offered (OK): final
 NPN/SPDY   not offered
 ALPN/HTTP2 h2, http/1.1 (offered)

 Testing cipher categories

 NULL ciphers (no encryption)                      not offered (OK)
 Anonymous NULL Ciphers (no authentication)        not offered (OK)
 Export ciphers (w/o ADH+NULL)                     not offered (OK)
 LOW: 64 Bit + DES, RC[2,4], MD5 (w/o export)      not offered (OK)
 Triple DES Ciphers / IDEA                         not offered
 Obsoleted CBC ciphers (AES, ARIA etc.)            not offered
 Strong encryption (AEAD ciphers) with no FS       offered (OK)
 Forward Secrecy strong encryption (AEAD ciphers)  offered (OK)


 Testing server's cipher preferences

 Has server cipher order?     yes (OK) -- only for < TLS 1.3
 Negotiated protocol          TLSv1.3
 Negotiated cipher            TLS_AES_256_GCM_SHA384, 256 bit ECDH (P-256)
 Cipher per protocol

Hexcode  Cipher Suite Name (OpenSSL)       KeyExch.   Encryption  Bits     Cipher Suite Name (IANA/RFC)
-----------------------------------------------------------------------------------------------------------------------------
SSLv2
 -
SSLv3
 -
TLSv1
 -
TLSv1.1
 -
TLSv1.2 (server order)
 xc030   ECDHE-RSA-AES256-GCM-SHA384       ECDH 256   AESGCM      256      TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
 xc02f   ECDHE-RSA-AES128-GCM-SHA256       ECDH 256   AESGCM      128      TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
 x9d     AES256-GCM-SHA384                 RSA        AESGCM      256      TLS_RSA_WITH_AES_256_GCM_SHA384
 x9c     AES128-GCM-SHA256                 RSA        AESGCM      128      TLS_RSA_WITH_AES_128_GCM_SHA256
TLSv1.3 (no server order, thus listed by strength)
 x1302   TLS_AES_256_GCM_SHA384            ECDH 256   AESGCM      256      TLS_AES_256_GCM_SHA384
 x1303   TLS_CHACHA20_POLY1305_SHA256      ECDH 256   ChaCha20    256      TLS_CHACHA20_POLY1305_SHA256
 x1301   TLS_AES_128_GCM_SHA256            ECDH 256   AESGCM      128      TLS_AES_128_GCM_SHA256


 Testing robust forward secrecy (FS) -- omitting Null Authentication/Encryption, 3DES, RC4

 FS is offered (OK)           TLS_AES_256_GCM_SHA384 TLS_CHACHA20_POLY1305_SHA256 ECDHE-RSA-AES256-GCM-SHA384
                              TLS_AES_128_GCM_SHA256 ECDHE-RSA-AES128-GCM-SHA256
 Elliptic curves offered:     prime256v1
...
```
As you can see above, Envoy is only allowing TLS1.2 and greater connections, FIPS-compliant ciphers, and the
P-256/prime256v1 elliptic curve algorithm.

[1]: https://istio.tetratelabs.io/
[2]: https://csrc.nist.gov/publications/detail/fips/140/2/final
[3]: https://boringssl.googlesource.com/boringssl/+/master/crypto/fipsmodule/FIPS.md
[4]: https://testssl.sh/
