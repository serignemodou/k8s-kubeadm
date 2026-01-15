## Create Issuer and Certificate
```
cat <<EOF > issuer.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-demo
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: issuer-test
  namespace: app-demo
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: app-demo
spec:
  dnsNames:
    - app.demo.io
  secretName: selfsigned-cert-tls
  issuerRef:
    name: issuer-test
EOF
```

```
kubectl apply -f issuer.yaml
```