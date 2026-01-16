## Install ArgoCD 
```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```
NB: Configuration not complete
```
values-argocd.yaml


oidc.config: |
  name: AzureAD
  issuer: https://login.microsoftonline.com/TENANT_ID/v2.0
  clientID: aaaabbbbccccddddeee
  clientSecret: sgejjdyedjj
```

```
helm install argocd argo/argo-cd --create-namespace --namespace argocd-system -f values-argocd.yaml
```

### Deploy ArgoCD Application bootstrap
```
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: argocd-system
spec:
  description: "Argocd project"
  sourceRepos:
  - 'https://github.com/serignemodou/demo-app.git'
  destinations:
  - namespace: "*"
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  namespaceResourceWhitelist:
  - group: '*'
    kind: '*'
  roles:
  - name: developer
    description: Developer privileges to default project
    policies:
    - p, proj:default:developer, applications, get, project/*, allow
    - p, proj:default:developer, applications, sync, project/*, allow
    - p, proj:default:developer, clusters, get, project/*, allow
    - p, proj:default:developer, repositories, get, project/*, allow
    - p, proj:default:developer, logs, get, project/*, allow
    groups:
    - OIDC_GROUP_DEVELOPER
  - name: admin
    description: Admin privileges to default project
    policies:
    - p, proj:default:developer, applications, *, project/*, allow
    - p, proj:default:developer, applications, *, project/*, allow
    - p, proj:default:developer, clusters, *, project/*, allow
    - p, proj:default:developer, repositories, *, project/*, allow
    - p, proj:default:developer, logs, get, project/*, allow
    groups:
    - OIDC_GROUP_ADMIN  # Prerequisite configure Argocd Helm with OIDC Provider

---

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: argocd-system
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: demo-app
  sources:
    - repoURL: https://github.com/serignemodou/demo-app.git
      targetRevision: "main"
      path: "./manifest
    - repoURL: https://github.com/serignemodou/demo-app-chart.git
      chart: demo-app
      targetRevision: 1.0.0

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```