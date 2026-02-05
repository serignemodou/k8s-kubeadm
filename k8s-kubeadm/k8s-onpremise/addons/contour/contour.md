### Install contour (as ingress controller) 

```
values-contour.yaml

envoy:
  service:
    type: LoadBalancer
    loadBalancerIP: 172.16.144.110 
    ports:
      http: 80
      https: 443
```

```
helm repo add contour https://projectcontour.github.io/helm-charts/
helm repo update
helm install contour contour/contour --version 0.2.1 --namespace kube-system -f values-contour.yaml
```