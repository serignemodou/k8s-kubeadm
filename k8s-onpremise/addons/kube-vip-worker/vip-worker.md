### Install kube-vip for Loadbalancer service
```
values-lb.yaml

nameOverride: "kube-vip-lb"

config:
  address: "172.16.144.110"


env:
  vip_interface: ens30
  vip_arp: "true"
  lb_enable: "true"
  vip_cidr: "32"
  svc_enable: "true"
  svc_election: "true"
  cp_enable: "false"
  vip_leaderelection: "true"
  vip_leaderelection: "true"

envValueFrom:
  vip_nodename:
    fieldRef:
      fieldPath: spec.nodeName
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: node-role.kubernetes.io/control-plane
          operator: DoesNotExist

```

```
helm repo add kube-vip https://kube-vip.github.io/helm-charts
helm upgrade --install kube-vip-lb kube-vip/kube-vip --version 0.6.6 -n kube-system -f values-lb.yaml
```