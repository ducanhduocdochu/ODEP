# ☸ Kubernetes HA Cluster Setup (3 Control Plane Nodes)

This guide describes how to build a 3-node High Availability Kubernetes cluster using kubeadm and containerd.

Cluster Architecture:

- k8s-master-1 → 192.168.41.105
- k8s-master-2 → 192.168.41.106
- k8s-master-3 → 192.168.41.107

---

# 1️⃣ Configure Hosts (Run on ALL Nodes)

```bash
echo -e "192.168.41.105 k8s-master-1\n192.168.41.106 k8s-master-2\n192.168.41.107 k8s-master-3" | sudo tee -a /etc/hosts
```
Update system:
```
sudo apt update -y && sudo apt upgrade -y
```
2️⃣ Create DevOps User
```
sudo adduser devops
sudo visudo
```
Add:
```
devops ALL=(ALL:ALL) ALL
```
Switch user:
```
su - devops
```
3️⃣ Disable Swap (Required for Kubernetes)
```
sudo swapoff -a
sudo sed -i '/swap.img/s/^/#/' /etc/fstab
```
Verify:
```
free -h
```
Swap must be 0.

4️⃣ Configure Kernel Modules
```
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/containerd.conf
```
Load modules:
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
5️⃣ Configure Sysctl Networking
```
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
```
Apply:
```
sudo sysctl --system
```
6️⃣ Install Containerd
Add Docker repository
```
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
Install containerd:
```
sudo apt update -y
sudo apt install -y containerd.io
```
Generate default config:
```
sudo containerd config default | sudo tee /etc/containerd/config.toml
```
Enable Systemd Cgroup:
```
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```
Restart containerd:
```
sudo systemctl restart containerd
sudo systemctl enable containerd
```
7️⃣ Install Kubernetes Components

Add Kubernetes repository:
```
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Install:
```
sudo apt update -y
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
8️⃣ Initialize Control Plane (Run on k8s-master-1 ONLY)
```
sudo kubeadm init \
--control-plane-endpoint "192.168.41.105:6443" \
--upload-certs \
--pod-network-cidr=192.168.0.0/16
```
Configure kubectl:
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
9️⃣ Install Calico Network (Run ONCE)
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```
Verify:
```
kubectl get pods -n kube-system
```
🔟 Join Additional Control Plane Nodes

Run the join command generated from master-1:
```
sudo kubeadm join 192.168.41.105:6443 \
--token <TOKEN> \
--discovery-token-ca-cert-hash sha256:<HASH> \
--control-plane \
--certificate-key <CERT_KEY>
```
Configure kubectl on joined nodes:
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
1️⃣1️⃣ Install Helm
```
wget https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz
tar xvf helm-v3.16.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/bin/
helm version
```
1️⃣2️⃣ Install Ingress NGINX

Add repo:
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
Pull chart:
```
helm pull ingress-nginx/ingress-nginx --version 4.11.3
tar -xzf ingress-nginx-4.11.3.tgz
```
Edit values.yaml:
```
type: NodePort

http: 30080

https: 30443
```
Create namespace:
```
kubectl create ns ingress-nginx
```
Remove control-plane taint (allow scheduling):
```
kubectl taint nodes k8s-master-1 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes k8s-master-2 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes k8s-master-3 node-role.kubernetes.io/control-plane:NoSchedule-
```
Install:
```
helm -n ingress-nginx install ingress-nginx -f ingress-nginx/values.yaml ingress-nginx
```
🛠 Fix kubectl Access (If Needed)

If kubectl returns localhost:8080 error:
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
Or configure permanently:
```
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
✅ Verify Cluster
```
kubectl get nodes
kubectl get pods -A
```
All nodes should be Ready.

👤 Author

DucAnh

I am a software engineer on the path to becoming a DevOps engineer.
