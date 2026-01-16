## Matrix deny default
| Service | Allow | 
|:--------|:-------- |
| DNS (53) |   ✅   | 
| Internet |   ✅   |
| Others  | ❌   |

## Deploy Deny Default network Policy
```
apiVersion: cilium.io/v2
kind: ClusterCiliumNetworkPolicy
metadata:
  name: "default-deny"
spec:
  description: "Block all trafic except DNS and Internet by default on all app namespace"

  endpointSelector:
    matchExpressions:
      - key: io.kubernetes.pod.namespace
        operator: NotIn
        values:
          - kube-system
          - default
          - cert-manager
          - gatekeeper-system
          - prometheus-system
          - argocd-system
  egress:
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
    - toPorts:
        - ports:
            - port: "53"
              protocol: UDP
            - port: "53"
              protocol: TCP
          rules:
            dns:
              - matchPattern: '*'
    - toEntities:
        - world
```