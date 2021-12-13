# Use a Different Hostname for Client DNS

### Table of Contents

- [Introduction](#introduction)
- [Configuration](#configuration)

## Introduction
By default, clients will use DNS to resolve a hostname. If clients must connect to a hostname that does not exist
in DNS (either Kubernetes or upstream), then the hostname must be [added to DNS][1]. Kubernetes supports
[HostAliases][2] for adding entries to a pod's "/etc/hosts" file. This document will demonstrate how to use HostAliases
to resolve a hostname that does not exist in DNS.

## Configuration
Follow the [Readme](../README.md) to create a cluster, Istio mesh, and example client (sleep) for testing. The client
should be running in the default namespace which is part of the mesh, e.g.
`kubectl label ns/default istio-injection=enabled`.
```
$ kubectl get po -l app=sleep
NAME                     READY   STATUS    RESTARTS   AGE
sleep-55b69559d8-slc5g   2/2     Running   0          26m
```
__Note__: 2/2 shows that the sidecar was injected into the sleep app pod.

Create the test server (httpbin) that will run outside the mesh:
```
kubectl create ns foo
kubectl apply -n foo -f ./manifests/statefulset.yaml
```

Verify the httpbin pods are running as expected:
```
$ kubectl get po -n foo -l app=httpbin
NAME        READY   STATUS    RESTARTS   AGE
httpbin-0   1/1     Running   0          31m
httpbin-1   1/1     Running   0          31m
```
__Note:__ Pods should show 1/1 since the server runs outside the mesh.

The client should be able to curl the server by DNS name:
```
SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
SVC_NAME=httpbin.foo.svc.cluster.local
kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - http://$SVC_NAME/status/200
```
You should receive a `200` HTTP response code and the client sidecar should show that it serviced the request:
```
kubectl logs $SOURCE_POD -c istio-proxy
...
[2021-12-13T19:27:09.629Z] "GET /status/200 HTTP/1.1" 200 - via_upstream - "-" 0 0 2 1 "-" "curl/7.80.0-DEV" "2c299b97-f4c0-9bd8-b9af-5097c571aa00" "httpbin.foo.svc.cluster.local" "10.244.1.15:80" outbound|80||httpbin.foo.svc.cluster.local 10.244.1.16:38960 10.244.1.15:80 10.244.1.16:38958 - default
```

Kubernetes automatically creates a DNS record for "httpbin.foo.svc.cluster.local" so the client can resolve this name.
If you try curl-ing the server by a name that does not exist in DNS the client will fail resolving this name.
For an example, create a ServiceEntry with hostname that does not reside in DNS ("httpbin.localhost"):
```
kubectl apply -f manifests/httpbin-localhost-serviceentry.yaml
```

Curl the ServiceEntry hostname:
```
SVC_NAME=httpbin.localhost
kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - http://$SVC_NAME/status/200
```
The curl should fail with `Could not resolve host: httpbin.localhost`.

Update the client with a DNS entry for "httpbin.localhost" that resolves to the Cluster IP of the httpbin service.
Since httpbin uses a headless service, the IP should be one of the httpbin endpoints:
```
EP_IP=$(kubectl get endpoints/httpbin -n foo -o=jsonpath='{.subsets[0].addresses[0].ip}')
kubectl patch deploy/sleep -p '{"spec":{"template":{"spec":{"hostAliases":[{"ip":"'$EP_IP'","hostnames":["httpbin.localhost"]}]}}}}'
```
__Note:__ The deployment will create a new pod.

Verify the configuration change:
```shell
kubectl get deploy/sleep -o yaml | grep -A 3 hostAliases
```
Test client>server connectivity using "httpbin.localhost":
```
SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
SVC_NAME=httpbin.localhost
kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - http://$SVC_NAME/status/200
```

The DNS error no longer appears but Envoy will return a 503 error because no "httpbin.localhost" cluster exists:
```
$ kubectl logs $SOURCE_POD -c istio-proxy
...
[2021-12-13T20:48:12.759Z] "GET /status/200 HTTP/1.1" 503 NC cluster_not_found - "-" 0 0 0 - "-" "curl/7.80.0-DEV" "2d14c097-84b3-93ab-aaa0-4d5c1f85ba91" "httpbin.localhost" "-" - - 10.244.1.14:80 10.244.1.18:40586 - default
```

Remove the ServiceEntry so that Envoy uses the PassthroughCluster:
```
kubectl delete se/httpbin
```

Retry the curl:
```
kubectl exec "$SOURCE_POD" -c sleep -- curl -sSL -o /dev/null -D - http://$SVC_NAME/status/200
```
You should now receive a `200` and the client sidecar should show the successful connection:
```
kubectl logs $SOURCE_POD -c istio-proxy
...
[2021-12-13T20:53:10.469Z] "GET /status/200 HTTP/1.1" 200 - via_upstream - "-" 0 0 1 1 "-" "curl/7.80.0-DEV" "8fd8579d-45a8-920f-877b-1089ccb25072" "httpbin.localhost" "10.244.1.14:80" PassthroughCluster 10.244.1.18:44006 10.244.1.14:80 10.244.1.18:44004 - allow_any
```

[1]: https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/
[2]: https://kubernetes.io/docs/tasks/network/customize-hosts-file-for-pods/
