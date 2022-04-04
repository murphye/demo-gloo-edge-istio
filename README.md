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

kubectl patch deployment gateway-proxy -n gloo-system --patch '{"spec": {"template": {"metadata": {"labels": {"sidecar.istio.io/inject": "true"}}}}}'

## Auto-inject Sidecars

```
kubectl label namespace petstore istio-injection=enabled --overwrite
kubectl rollout restart deploy petstore -n petstore
```

## Add mTLS

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


```
cat <<EOF | kubectl apply -n petstore -f -
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: petstore-mtls
spec:
  host: petstore.petstore.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF
```

## (Optional) Istio Ingress Gateway

```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

kubectl apply -n petstore -f petstore-istio-vs.yaml

curl -i $INGRESS_HOST/api/pets
```

### Gloo Gateway Istio Sidecar Proxy Logs

```
k logs -n gloo-system gateway-proxy-58c7bd476-7d9jh -c istio-proxy -f
```

### Gloo Gateway Proxy Logs

```
glooctl proxy logs -f
```








https://discuss.istio.io/t/mtls-not-working-when-service-port-and-targetport-are-different/10681

https://docs.aws.amazon.com/app-mesh/latest/userguide/troubleshooting-connectivity.html

No matching filter chain found – This is most likely caused when a request is sent to a virtual service on an invalid port. Make sure that the requests from the application are using the same port specified on the virtual router.





k edit -n gloo-system deployments.apps gateway-proxy
traffic.sidecar.istio.io/excludeInboundPorts: 8080,8443
