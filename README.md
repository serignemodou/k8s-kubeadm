# Deploy kubernetes cluster with kubeadm
## Infrastructure
1. Compute

| Server Name | VLAN (VLanId 106; CIDR 172.16.144.0/28) | Roles | OS |
|:-------- |:--------:| --------:| --------:|
| master01     | 172.16.144.34  | Control Plane  | Ubuntu22.04 |
| master02     | 172.16.144.44  | Control Plane  | Ubuntu22.04 |
| master03     | 172.16.144.54  | Control Plane  | Ubuntu22.04 |
| worker01     | 172.16.144.66  | Worker Node    | Ubuntu22.04 |
| worker02     | 172.16.144.76  | Worker Node    | Ubuntu22.04 |
| worker03     | 172.16.144.86  | Worker Node    | Ubuntu22.04 |


| VIP Name | Private IP | Public IP | Potrs |
|:-------- |:--------:| --------:| --------:|
|  apiserver  |  172.16.144.92  | -  | 8443 => 6443 |
|  workers  |  172.16.144.98  |  41.77.30.10 | 443 => 8443 / 80 => 8080 |

2. Network

| Cluster | VLAN ID | VLAN CIDR | Opening Port |
|:--------|:-------- |:--------:| --------:| 
| Dev | Vlan 106     | 172.16.144.0/28  | 8443, 6443, 8080, 80 |

3. DNS Resolution
Record dns for vip: api.proxima.swap.io => 172.16.144.92

3. Storage
Comming soon ! 

## Installation (bootstrap k8s cluster)
### Install prerequisites (install ubuntu package ; desable swap ; install modules ; configure iptables)
#### Install ubuntu package
```
apt-get update && apt-get upgrade -y
apt-get install unzip tar apt-transport-https libseccomp2 util-linux ca-certificates curl gpg nfs-common -y
```
#### Disable swap configuration
```
swapoff -a
sed -e '/swap/s/^/#/g' -i /etc/fstab
```

#### Configure required modules
```
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF

modprobe overlay
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprob ip_vs_wrr
modprob ip_vs_sh

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```

### Local DNS
```
In /etc/hosts ; hostname -> IP address
```
### Install and configure containerd runtime (1.7.30)
```
mkdir -p /opt/cni/bin/
mkdir -p /etc/cni/net.d/
mkdir -p /etc/containerd

wget https://github.com/containerd/containerd/releases/download/v1.7.30/cri-containerd-cni-1.7.30-linux-amd64.tar.gz
tar Cxzvf /opt/cni/bin/ cri-containerd-cni-1.7.30-linux-amd64.tar.gz
rm -f cri-containerd-cni-1.7.30-linux-amd64.tar.gz

systemctl daemon-reload
systemctl start containerd
systemctl enable containerd
systemctl status containerd

cat <<EOF | sudo tee /etc/containerd/config.toml
version = 2
imports = ["/etc/containerd/conf.d/*.toml"]
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    sandbox_image = "registry.k8s.io/pause:3.10"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true
EOF

systemctl restart containerd
systemctl status containerd

```

### Install kubernetes components binaire (Add dpkg packages on ubuntu and install)
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet=1.31.5-1.1 kubeadm=1.31.5-1.1 kubectl=1.31.5-1.1
apt-mark hold kubelet kubeadm kubectl

```

### Insert this line below with vi in /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf file after section [Service] (it's kubelet attributes parameters)
```
Environment="KUBELET_EXTRA_ARGS= --runtime-cgroups=/system.slice/containerd.service --container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```

### Install kubernetes
#### Only on the first master
##### Initialize cluster
```
mkdir /opt/kubernetes

cat <<EOF | sudo tee /opt/kubernetes/kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta4
clusterName: swap-sandbox
kubernetesVersion: v1.31.5
apiServer:
  extraArgs:
    - name: "audit-log-path"
      value: "/var/log/kubernetes/audit/audit.log"
  certSANs:
    - "api.proxima.swap.io"
    - "172.16.144.92" #vip-private ip
controllerManager:
  extraArgs:
  - name: node-cidr-mask-size
    value: "24"
networking:
  serviceSubnet: "192.168.128.0/17"
  podSubnet: "192.168.0.0/17"
controlPlaneEndpoint: "api.proxima.swap.io:8443"

---

kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd

EOF

kubeadm init --config /opt/kubernetes/kubeadm-config.yaml --skip-phases=addon/kube-proxy --upload-certs | tee /opt/kubernetes/kubeadm-init.out

```

##### Install cilium cni (version 1.18.5 with helm)
```
wget https://get.helm.sh/helm-v3.17.0-linux-amd64.tar.gz
tar -zxvf helm-v3.17.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
rm helm-v3.17.0-linux-amd64.tar.gz

helm repo add cilium https://helm.cilium.io/

export KUBECONFIG=/etc/kubernetes/admin.conf

helm install cilium cilium/cilium --version 1.18.5 \
--set ipam.operator.clusterPoolIPv4PodCIDRList={"192.168.0.0/17"} \
--set ipam.operator.clusterPoolIPv4MaskSize=24 \
--namespace kube-system

export API_SERVER_IP="api.proxima.swap.io"
export API_SERVER_PORT="8443"

helm upgrade --install cilium cilium/cilium --version 1.18.5 --namespace kube-system \
--set k8sServiceHost=${API_SERVER_IP} \
--set k8sServicePort=${API_SERVER_PORT} \
--set kubeProxyReplacement=true \
--set hubble.relay.enabled=true \
--set ipam.operator.clusterPoolIPv4PodCIDRList={"192.168.0.0/17"} \
--set ipam.operator.clusterPoolIPv4MaskSize=24 \
--set tunnelProtocol="geneve" \
--set devices="ens192" \
--set hubble.ui.enabled=true

kubectl get pods --all-namespaces -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,HOSTNETWORK:.spec.hostNetwork --no-headers=true | grep '<none>' | awk '{print "-n "$1" "$2}' | xargs -L 1 -r kubectl delete pod
```

##### Join master (connect to the second then the thirty master vm)
| Variable | Description | Location | 
|:--------|:-------- |:--------:| 
| JOIN_TOKEN | Token to join the control plan  | /opt/kubernetes/kubeadm-init.out  | 
| JOIN_TOKEN_CACERT_HASH | Token Certificate Authority hash to join the control plan  | /opt/kubernetes/kubeadm-init.out  | 
| JOIN_TOKEN_CERT_KEY | Token Certificat Key to join the control plan | /opt/kubernetes/kubeadm-init.out  | 

```
kubeadm join api.proxima.swap.io:8443 --token <JOIN_TOKEN> \
--discovery-token-ca-cert-hash <JOIN_TOKEN_CACERT_HASH> \
--control-plane --certificate-key <JOIN_TOKEN_CERT_KEY>
```

#### Join workers (connect on each worker and make the command below)
```
kubeadm join api.proxima.swap.io:8443 --token <JOIN_TOKEN> \
--discovery-token-ca-cert-hash <JOIN_TOKEN_CACERT_HASH>
```

### Check Nodes status
```
kubectl get node
```

### Check Pods
```
kubectl get pod -A
```

### Check Cilium connectivity status
```
kubectl -n kube-system exec ds/cilium -- cilium-health status

kubectl -n kube-system exec ds/cilium -- cilium-dbg status
```
