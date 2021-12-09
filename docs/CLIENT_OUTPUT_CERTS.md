# Store Client Cert/Key Pair for Istio mTLS

### Table of Contents

- [Introduction](#introduction)
- [Configuration](#configuration)
- [Verification](#verification)

## Introduction
By default, Istio Agent stores the client TLS cert/key pair in memory. The agent can be
configured to store these assets to disk using the [OUTPUT_CERTS][1] env var.

## Configuration
The configuration is provided through annotations of a workload resource. The following example
configures Istio Agent to store client TLS assets in directory "/etc/certs". This directory is
mounted as an [emptryDir][2] volume.
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
  namespace: default
spec:
  template:
    metadata:
      annotations:
        proxy.istio.io/config: |
          proxyMetadata:
            OUTPUT_CERTS: /etc/certs
        sidecar.istio.io/userVolume: '[{"name": "istio-certs", "emptyDir": {"medium":
          "Memory"}}]'
        sidecar.istio.io/userVolumeMount: '[{"name": "istio-certs", "mountPath": "/etc/certs"}]'
...
```
__Note:__ The server-side proxy cert directory is not used since it's created by the sidecar
injector as read-only.

## Verification
Follow the [Readme](../README.md) to create a cluster, Istio mesh, and example client/workload
for testing. Verify client<>server internal mesh connectiivity:
```
kubectl exec -it deploy/sleep -c sleep -- curl -I http://httpbin-0.httpbin.default.svc.cluster.local/status/200
```
You should receive a `200` HTTP response code. Repeat this step for the `httpbin-1` Pod.

Update the `deploy/sleep` resource according to the configuration above. Verify the deployment
has updated the pod and both containers are running:
```
kubectl get po -l app=sleep
```

Verify the Istio Agent is now storing the TLS assets to the configured directory:
```
$ kubectl logs deploy/sleep -c istio-proxy -f
2021-12-09T20:17:38.568217Z     info    Apply proxy config from env {"proxyMetadata":{"OUTPUT_CERTS":"/etc/certs"}}

2021-12-09T20:17:38.568689Z     info    Apply proxy config from annotation proxyMetadata:
  OUTPUT_CERTS: /etc/certs

2021-12-09T20:17:38.569156Z     info    Effective config: binaryPath: /usr/local/bin/envoy
concurrency: 2
configPath: ./etc/istio/proxy
controlPlaneAuthPolicy: MUTUAL_TLS
discoveryAddress: istiod.istio-system.svc:15012
drainDuration: 45s
parentShutdownDuration: 60s
proxyAdminPort: 15000
proxyMetadata:
  OUTPUT_CERTS: /etc/certs
...
```

Verify the client-side TLS assets exist on disk:
```
$ kubectl exec -it deploy/sleep -c istio-proxy -- ls -al /etc/certs
total 24
drwxrwxrwt 2 root        root         120 Dec  9 20:17 .
drwxr-xr-x 1 root        root        4096 Dec  9 20:17 ..
-rw-r--r-- 1 istio-proxy istio-proxy 2278 Dec  9 20:17 cert-chain.pem
-rw-r--r-- 1 istio-proxy istio-proxy 1675 Dec  9 20:17 key.pem
-rw-r--r-- 1 istio-proxy istio-proxy 1093 Dec  9 20:17 root-cert.pem
```

Verify the details of the cert chain:
```
kubectl exec deploy/sleep -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text -noout
```

Optionally, you can use the following command to simply verify the cert identity is
as expected:
```
$ kubectl exec deploy/sleep -c istio-proxy -- cat /etc/certs/cert-chain.pem | openssl x509 -text -noout  | grep 'Subject Alternative Name' -A 1
            X509v3 Subject Alternative Name: critical
                URI:spiffe://cluster.local/ns/default/sa/sleep
```

Verify mTLS between workload pods:
```
kubectl exec -it deploy/sleep -c sleep -- curl -I http://httpbin-0.httpbin.default.svc.cluster.local/status/200
...
```
You should receive a `200` HTTP response code. Repeat this step for the `httpbin-1` Pod.

[1]: https://istio.io/latest/docs/reference/commands/pilot-agent/
[2]: https://kubernetes.io/docs/concepts/storage/volumes/#emptydir

