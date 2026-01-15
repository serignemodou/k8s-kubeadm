# 🔒 Install Cert-Manager to request SSL Certificate
```
helm repo add jetstack https://charts.jetstack.io 
helm repo update

```

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.2 \
  --set crds.enabled=true
```