# TID Canary Upgrade

### Table of Contents

- [Introduction](#introduction)
- [Configuration](#configuration)

## Introduction
Istio supports [canary upgrades][1] to run multiple Istiod control-planes at different software versions. A canary
version of an upgrade can be started by installing the new control plane version next to the old one, using a
different `revision`. Each `revision` is a full Istio control plane implementation with its own Deployment, Service,
etc. The following document provides instructions for performing a canary upgrade.

## Configuration

Follow the [Readme](../README.md) to create a cluster and then install Istio:
```shell
istioctl install -f ./manifests/istio-operator-tetratefips-1.11.4-nodeport.yaml -y --verify
```
The installation should complete, and the Istio control plane should be in a ready state.

Setup the default namespace for automatic sidecar injection:
```shell
kubectl label ns/default istio-injection=enabled
```

Deploy a test client and server workload:
```shell
ISTIO_VERSION=1.11
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-$ISTIO_VERSION/samples/sleep/sleep.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-$ISTIO_VERSION/samples/httpbin/httpbin.yaml
```

Verify the rollout status of the test workloads:
```shell
for i in sleep httpbin; do kubectl rollout status deploy/$i; done
```

Verify mesh control and data plane versions:
```shell
istioctl version
```

Verify mesh connectivity:
```shell
kubectl exec -it deploy/sleep -c sleep -- curl -o /dev/null -s -w '%{http_code}\n' http://httpbin.default:8000/headers
```

Before starting the canary upgrade, analyze the stio configuration.
```shell
istioctl analyze
```

Install the 1.12.1 Istiod control plane:
```shell
istioctl install -f ./manifests/istio-operator-tetratefips-1.12.1-nodeport.yaml --set revision=1-12-1 -y --verify
```
__Note:__ Ingress and egress gateways get upgraded when the new Istiod control plane is installed.

Verify both Istiod revisions are running:
```shell
kubectl get po -n istio-system
```

Verify the versions:
```shell
istioctl version
```
At this point, both Istiod control planes should be running. Workload sidecars should contionue to use the 1.11.4
control plane and ingress/egress proxies should have been upgraded inline.

Set the 1-12-1 revision label for the workload namespace.
```shell
kubectl label ns default istio-injection- istio.io/rev=1-12-1
```

Restart the workloads to upgrade the sidecar proxies.
```shell
kubectl rollout restart deployment
```

Verify the rolling restart status of the test workloads:
```shell
for i in sleep httpbin; do kubectl rollout status deploy/$i; done
```

Verify the versions:
```shell
istioctl version
```

Verify mesh connectivity:
```shell
kubectl exec -it deploy/sleep -c sleep -- curl -o /dev/null -s -w '%{http_code}\n' http://httpbin.default:8000/headers
```

Remove the 1.11.4 control plane.
```shell
istioctl x uninstall -f ./manifests/istio-operator-tetratefips-1.11.4-nodeport.yaml -y
```

[1]: https://istio.io/latest/docs/setup/upgrade/canary/
