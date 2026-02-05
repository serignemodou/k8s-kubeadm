## Install Gatekeeper for K8s Governance (Apply Constraint to all namespaces except kube-system and default)
```
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update
```

```
values.gatekeeper.yaml


controllerManager:
  exemptNamespaces: ["kube-system", "default"]
mutatingWebhookCustomRules:
  - apiGroups:
      - '*'
    apiVersions:
      - '*'
    operations:
      - CREATE
      - UPDATE
    resources:
      - '*'
      - pods/ephemeralcontainers
      - pods/exec
      - pods/log
```