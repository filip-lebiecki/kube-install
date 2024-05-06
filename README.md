# kube-install
Installing Kubernetes 1.30 on Ubuntu 24.04 LTS

### NODE1, NODE2, NODE3 (as root) ###
```
lsb_release -a
uname -a
ip -br a

cat /etc/netplan/50-cloud-init.yaml

vi /etc/sysctl.conf
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
sysctl -p

vi /etc/fstab
comment swap
swapoff -a
free -m

apt update
apt upgrade -y
apt install apt-transport-https ca-certificates curl jq -y
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
cat /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list
cat /etc/apt/sources.list.d/docker.list

apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

containerd config default > /etc/containerd/config.toml
vi /etc/containerd/config.toml
SystemdCgroup = true

systemctl restart containerd
apt install kubelet kubeadm kubectl -y
kubeadm config images pull

vi kubeadm-config.yaml
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
networking:
  podSubnet: 10.244.0.0/16,fc00:10:244::/56
  serviceSubnet: 10.96.0.0/16,fc00:10:96::/108
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.10.10"
  bindPort: 6443
nodeRegistration:
  kubeletExtraArgs:
    node-ip: 192.168.10.10,2001:470:61bb:10::10

kubeadm init --config=kubeadm-config.yaml
Ctrl+D
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
systemctl status kubelet
curl -OL https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
vi kube-flannel.yml (net-conf.json)

     "EnableIPv6": true,
      "IPv6Network" : "fc00:10:244::/56"

kubectl apply -f kube-flannel.yml
kubectl get all -n kube-flannel
kubectl get pods -A
kubectl describe node node1 | grep Taints
kubectl taint node node1 node-role.kubernetes.io/control-plane:NoSchedule-

NODE1
kubeadm token create --print-join-command

NODE2, NODE3
vi kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: 192.168.10.10:6443
    token: "6bdi75.5t3uzvbzvyz2sv71"
    caCertHashes:
      - "sha256:cec5535cc5777486ff2dc8f1347546a0f6e9e4555b60bc942d8c0aaab9731c2c"
nodeRegistration:
  kubeletExtraArgs:
   node-ip: 192.168.10.11,2001:470:61bb:10::11

kubeadm join --config=kubeadm-config.yaml

NODE1
kubectl get nodes
kubectl get nodes -o json | jq '.items[].spec.podCIDRs'
kubectl run alpine --image=alpine -it
ip a
```
