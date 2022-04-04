This demo explores an integration of Gloo Edge and Istio that goes beyond just the mTLS integration documented here:
https://docs.solo.io/gloo-edge/latest/guides/integrations/service_mesh/gloo_istio_mtls/

## Install Gloo Edge Community (Version 1.10.18)

```
glooctl install gateway
```

## Install Istio (Version 1.13.2)

```
istioctl install --set profile=default
```

## Install PetStore and Route Through Gloo Edge

```
k create ns petstore
k apply -n petstore -f ./petstore.yaml
k get upstream -n gloo-system
k apply -f ./petstore-gloo-vs.yaml
curl -i $(glooctl proxy url)/pets
```

## Add Istio Sidecar to Gloo Edge Proxy

```
kubectl patch deployment gateway-proxy -n gloo-system --patch '{"spec": {"template": {"metadata": {"labels": {"sidecar.istio.io/inject": "true"}}}}}'
```

## Add Exclude Inbound Ports

```
kubectl patch deployment gateway-proxy -n gloo-system --patch '{"spec": {"template": {"metadata": {"annotations": {"traffic.sidecar.istio.io/excludeInboundPorts": "8080,8443"}}}}}'
```

## Auto-inject Sidecars

```
kubectl label namespace petstore istio-injection=enabled --overwrite
kubectl rollout restart deploy petstore -n petstore
```

## Add Strict mTLS

```
kubectl apply -n petstore -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
EOF
```

## Observe 503 Service Unavailable

This is due to direct to pod routing of Gloo Edge.

```
curl -i $(glooctl proxy url)/pets
virtualservice.gateway.solo.io/default configured
upstream.gloo.solo.io/petstore-static-upstream unchanged
HTTP/1.1 503 Service Unavailable
content-length: 95
content-type: text/plain
date: Mon, 04 Apr 2022 04:17:28 GMT
server: envoy
```

## Apply Static Upstream to Service

```
k apply -f ./petstore-gloo-vs-static.yaml
```

Service will be available again, no more 503.

```
curl -i $(glooctl proxy url)/pets
```

### Gloo Gateway Istio Sidecar Proxy Logs

```
k logs -n gloo-system gateway-proxy-58c7bd476-7d9jh -c istio-proxy -f
```

### Gloo Gateway Proxy Logs

```
glooctl proxy logs -f
```