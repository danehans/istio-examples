# EDS for Headless Services

### Table of Contents

- [Introduction](#introduction)
- [Verification](#verification)

## Introduction
By default, Istiod will use the [ORIGINAL_DST][1] cluster discovery type for headless services.
Istio provides the [PILOT_ENABLE_HEADLESS_SERVICE_POD_LISTENERS][2] env var to change the type
to [EDS][3].

## Verification
Follow the [Readme](../README.md) to create a cluster, Istio mesh, and example client/workload
for testing. Verify the client sidecar proxy uses `ORIGINAL_DST` as the cluster type for the
"httpbin.default.svc.cluster.local" cluster:
```
$ istioctl pc clusters sleep-557747455f-tjth8.default --fqdn httpbin.default.svc.cluster.local
SERVICE FQDN                          PORT     SUBSET     DIRECTION     TYPE             DESTINATION RULE
httpbin.default.svc.cluster.local     80       -          outbound      ORIGINAL_DST
```

Verify client<>server mesh connectivity:
```
$ kubectl exec -it deploy/sleep -c sleep -- curl -I http://httpbin.default.svc.cluster.local/status/200
HTTP/1.1 200 OK
server: envoy
...
```

Have Pilot use `EDS` for headless services by adding the `PILOT_ENABLE_HEADLESS_SERVICE_POD_LISTENERS=true`
env var to the Istiod deployment:
```
$ kubectl edit deploy/istiod -n istio-system
...
    spec:
      containers:
      - args:
        - discovery
        - --monitoringAddr=:15014
        - --log_output_level=default:info
        - --domain
        - cluster.local
        - --keepaliveMaxServerConnectionAge
        - 30m
        env:
...
        - name: PILOT_ENABLE_EDS_FOR_HEADLESS_SERVICES
          value: "true"
```

Verify the deployment recreates the Istiod pod with the env var set:
```
$ kubectl get po -n istio-system -l app=istiod
NAME                      READY   STATUS    RESTARTS   AGE
istiod-7d48d4f764-5fbd2   1/1     Running   0          1m

$ kubectl get po -n istio-system -l app=istiod -o yaml | grep PILOT_ENABLE_EDS_FOR_HEADLESS_SERVICES
      - name: PILOT_ENABLE_EDS_FOR_HEADLESS_SERVICES
```

Verify that the client proxy shows `EDS` as the discovery type for "httpbin.default.svc.cluster.local":
```
$ istioctl pc clusters sleep-557747455f-tjth8.default --fqdn httpbin.default.svc.cluster.local
SERVICE FQDN                          PORT     SUBSET     DIRECTION     TYPE     DESTINATION RULE
httpbin.default.svc.cluster.local     80       -          outbound      EDS
```

Verify client<>server mesh connectivity:
```
$ kubectl exec -it deploy/sleep -c sleep -- curl -I http://httpbin.default.svc.cluster.local/status/200
HTTP/1.1 200 OK
server: envoy
...
```

[1]: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery#original-destination
[2]: https://istio.io/latest/docs/reference/commands/pilot-discovery/
[3]: https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery#endpoint-discovery-service-eds

