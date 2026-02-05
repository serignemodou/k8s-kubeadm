## Exploit NFS Server
#### Create NFS Provisionner Class (Provionnement dynamique)
```
apiVersion: v1
kind: Pod
metadata:
  name: nfs-provisioner


---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-provisioner
  namespace: nfs-provisioner

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  namespace: nfs-provisioner
  name: nfs-provisioner-runner
rules:
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch", "create", "delete"]
- apiGroups: [""]
  resources: ["persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "update"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: ServiceAccount
  name: nfs-provisioner
  namespace: nfs-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-provisioner
      containers:
      - name: nfs-client-provisioner
        image: quay.io/external_storage/nfs-client-provisioner:latest
        env:
        - name: PROVISIONER_NAME
          value: nfs-provisioner
        - name: NFS_SERVER
          value: 172.16.144.40
        - name: NFS_PATH
          value: /srv/nfs/k8s
        volumeMounts:
        - name: nfs-volume
          mountPath: /persistentvolumes
      volumes:
      - name: nfs-volume
        nfs:
          server: 172.16.144.40
          path: /srv/nfs/k8s

---

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs-provisioner
reclaimPolicy: Delete
volumeBindingMode: Immediate

```