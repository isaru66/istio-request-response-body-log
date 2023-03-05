# Introduction
This is how to implement istio envoy filter to enable http request/response body logging.

This based on the content of 
- https://themartian.hashnode.dev/http-request-body-logging-with-istio-and-envoy
- https://github.com/envoyproxy/envoy/issues/4724

# Pre-requisite
Setup istio and install some sample application. 
```
# Download and install istio
# then cd into the downloaded folder for example ~/istio-1.17.1
bin/istioctl install --set profile=demo -y

# Enable Namespace injection
kubectl label namespace default istio-injection=enabled

# Install bookinfo sample application
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
```

# Apply Envoy Filter

## Configure istio
```
# edit mesh config 
kubectl edit cm -n istio-system isti

# in mesh Config, configure requestBody and responseBody based on DYNAMIC_METADATA that we will set from envoy filter
  meshConfig:
    accessLogFile: /dev/stdout
    accessLogEncoding: JSON
    accessLogFormat: |
      {
        "protocol": "%PROTOCOL%",
        "method": "%REQ(:METHOD)%",
        "path": "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%",
        "responseCode": "%RESPONSE_CODE%",
        "clientDuration": "%DURATION%",
        "responseCodeDetails": "%RESPONSE_CODE_DETAILS%",
        "connectionTerminationDetails": "%CONNECTION_TERMINATION_DETAILS%",
        "targetDuration": "%RESPONSE_DURATION%",
        "upstreamName": "%UPSTREAM_CLUSTER%",
        "traceId": "%REQ(X-B3-Traceid)%",
        "responseFlags": "%RESPONSE_FLAGS%",
        "routeName": "%ROUTE_NAME%",
        "downstreamRemoteAddress": "%DOWNSTREAM_REMOTE_ADDRESS%",
        "upstreamHost": "%UPSTREAM_HOST%",
        "downstreamLocalURISan": "%DOWNSTREAM_LOCAL_URI_SAN%",
        "requestHeaders": "%DYNAMIC_METADATA(envoy.lua:request_headers)%",
        "requestBody": "%DYNAMIC_METADATA(envoy.lua:request_body)%",
        "responseHeaders": "%DYNAMIC_METADATA(envoy.lua:response_headers)%",
        "responseBody": "%DYNAMIC_METADATA(envoy.lua:response_body)%"
      }

```

## Apply envoy filter

```
# cd to this repository folder
kubectl apply -f istio-request-response-filter.yaml

```

wait for envoy proxy to apply... this might took arround 30 sec

## Test
then, monitor the log on bookinfo/productpage

```
kubectl logs -f $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c istio-proxy
```

in another terminal, send the request to the productpage
```
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
```

we should see the the **responseBody** in envoyproxy log
```
{"path":"/details/0","targetDuration":17,"responseBody":"{\"id\":0,\"author\":\"William Shakespeare\",\"year\":1595,\"type\":\"paperback\",\"pages\":200,\"publisher\":\"PublisherA\",\"language\":\"English\",\"ISBN-10\":\"1234567890\",\"ISBN-13\":\"123-1234567890\"}","method":"GET","requestBody":"","routeName":"default","upstreamHost":"10.1.0.106:9080","traceId":"d2c585fe28d729b84dce0cfd0162040",... }
```