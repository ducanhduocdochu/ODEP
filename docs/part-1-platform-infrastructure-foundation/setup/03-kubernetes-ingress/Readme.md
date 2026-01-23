# README.md – Kubernetes HA (3 Control Planes) + Calico + NGINX Ingress (NodePort)

This guide installs a Kubernetes cluster (kubeadm) with **3 control-plane nodes**:

* `k8s-master-1`: `192.168.100.105`
* `k8s-master-2`: `192.168.100.106`
* `k8s-master-3`: `192.168.100.107`

Components:

* Container runtime: **containerd**
* CNI plugin: **Calico**
* Ingress Controller: **ingress-nginx**

  * Service type: **NodePort**
  * HTTP NodePort: **30080**
  * HTTPS NodePort: **30443**

> Run the same prerequisites on all 3 masters.
> Only `kubeadm init` is executed on `k8s-master-1`.

---

## 0) Add hosts entries (All masters)

Run on **all 3 servers**:

```bash
echo -e "192.168.100.105 k8s-master-1\n192.168.100.106 k8s-master-2\n192.168.100.107 k8s-master-3" | sudo tee -a /etc/hosts
```

---

## 1) Update OS packages (All masters)

```bash
sudo apt update -y && sudo apt upgrade -y
```

---

## 2) Create `devops` user + sudo permission (All masters)

Create user:

```bash
sudo adduser devops
```

Grant sudo:

```bash
sudo visudo
```

Add this line:

```bash
devops ALL=(ALL) NOPASSWD:ALL
```

Switch to user:

```bash
su devops
cd /home/devops
```

---

## 3) Disable swap (All masters)

```bash
sudo swapoff -a
sudo sed -i '/swap.img/s/^/#/' /etc/fstab
```

---

## 4) Load kernel modules (All masters)

Create module load config:

```bash
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/containerd.conf > /dev/null
```

Load modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

---

## 5) Apply sysctl networking settings (All masters)

```bash
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
```

Apply:

```bash
sudo sysctl --system
```

---

## 6) Install containerd (All masters)

Install dependencies:

```bash
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates
```

Add Docker repository:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

Install containerd:

```bash
sudo apt update -y
sudo apt install -y containerd.io
```

Configure containerd:

```bash
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

Restart + enable:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 7) Install Kubernetes packages (All masters)

Add Kubernetes repository:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
```

Install kube packages:

```bash
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 8) Cluster reset (optional, when re-installing)

Run when you want to reset a node:

```bash
sudo kubeadm reset -f
sudo rm -rf /var/lib/etcd
sudo rm -rf /etc/kubernetes/manifests/*
```

---

# 9) Initialize the cluster (ONLY on k8s-master-1)

### 9.1 kubeadm init

```bash
sudo kubeadm init --control-plane-endpoint "192.168.100.105:6443" --upload-certs
```

### 9.2 Configure kubeconfig

```bash
mkdir -p $HOME/.kube 
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 9.3 Install Calico CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

---

# 10) Join other control-plane nodes (k8s-master-2 and k8s-master-3)

Run the join command (from master-1 output):

```bash
sudo kubeadm join 192.168.100.105:6443 --token your_token --discovery-token-ca-cert-hash your_sha --control-plane --certificate-key your_cert
```

Configure kubeconfig:

```bash
mkdir -p $HOME/.kube 
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Apply Calico if needed:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

---

## 11) Install Helm

Run on master-1 (or any node that has kubectl access):

```bash
wget https://get.helm.sh/helm-v3.16.2-linux-amd64.tar.gz
tar xvf helm-v3.16.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/bin/
helm version
```

---

## 12) Install NGINX Ingress Controller (NodePort)

### 12.1 Add Helm repo

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

### 12.2 Download chart

```bash
helm pull ingress-nginx/ingress-nginx --version 4.11.3
tar -xzf ingress-nginx-4.11.3.tgz
```

### 12.3 Edit values.yaml (IMPORTANT)

```bash
vi ingress-nginx/values.yaml
```

Change:

**(1) controller.service.type**

* `LoadBalancer` → `NodePort`

**(2) controller.service.nodePorts**

* http: `""` → `"30080"`
* https: `""` → `"30443"`

---

### 12.4 Create namespace

```bash
kubectl create ns ingress-nginx
```

---

## 13) Remove taint from control-plane nodes (Allow scheduling)

```bash
kubectl taint nodes k8s-master-1 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes k8s-master-2 node-role.kubernetes.io/control-plane:NoSchedule-
kubectl taint nodes k8s-master-3 node-role.kubernetes.io/control-plane:NoSchedule-
```

---

## 14) Install ingress-nginx via Helm

```bash
helm -n ingress-nginx install ingress-nginx -f ingress-nginx/values.yaml ingress-nginx
```

---

## 15) Fix "cannot get nodes" issue (kubeconfig)

If kubectl cannot access the cluster:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 16) Verify installation

Check nodes:

```bash
kubectl get nodes -o wide
```

Check ingress controller:

```bash
kubectl get pods -n ingress-nginx -o wide
kubectl get svc -n ingress-nginx
```

Expected NodePorts:

* HTTP: `30080`
* HTTPS: `30443`

---

DONE ✅

* Kubernetes HA (3 masters) installed
* Calico CNI applied
* NGINX Ingress Controller installed using NodePort

