## Install server NFS
#### Install NFS server packages on (NFS-Server: 172.16.144.40)
```
apt-get update && apt-get upgrade -y
apt-get install nfs-kernel-server nfs-common

systemctl daemon-reload
systemctl start nfs-kernel-server
systemctl enable nfs-kernel-server
systemctl status nfs-kernel-server

systemctl daemon-reload
systemctl start rpcbind
systemctl enable rpcbind
systemctl status rpcbind
```

#### Configuration 
```
mkdir -p /srv/nfs/k8s
chmod 777 /srv/nfs/k8s
chown nobody:nogroup /srv/nfs/k8s

cat <<EOF | sudo tee -a /etc/exports
/srv/nfs/k8s  172.16.144.0/24(rw,sync,no_subtree_check,no_root_squash)
EOF

exportfs -rav
systemctl restart nfs-kernel-server
systemctl status nfs-kernel-server

systemctl restart rpcbind
systemctl status rpcbind
```

#### Install NFS Client on Worker Node
```
apt-get update && apt-get upgrade -y
apt-get install nfs-common
```