# 🚀 Kubernetes On-Premise Cluster Deployment Guide

---

## ✨ Purpose

This guide is a **minimalist, production-ready manual** for deploying a highly available Kubernetes cluster on bare metal or in your private datacenter. It focuses on **simplicity** and **robustness**—giving you everything you need to get started, and nothing you don’t.

You’ll build a cluster with:

- **containerd** as a secure, lightweight container runtime
- **kubeadm** for easy, reliable cluster bootstrapping
- **kube-vip** for a virtual IP (VIP) and HA control plane
- **Cilium** as the CNI, with eBPF-powered, proxy-free, secure networking

---

## 💡 Why This Approach Is Best Practice

- **Minimalism:** Only the essential components—no bloat, no guesswork.
- **Cloud-Native:** All tools are cloud-native, open source, and widely adopted.
- **Transparency:** Each step is explicit and manual, making troubleshooting and learning easier.
- **High Availability:** kube-vip provides seamless HA for the Kubernetes API, without external load balancers.
- **Modern Networking:** Cilium delivers state-of-the-art, high-performance networking (eBPF, WireGuard encryption, L7 policies).
- **Vendor Neutral:** No lock-in; fully open source.
- **Production-Ready:** Forms a strong, stable foundation for your workloads—easy to extend with monitoring, backup, and security later.

---

## 👤 Who Is This Guide For?

- Engineers seeking a **reproducible, minimal, and enterprise-ready Kubernetes cluster** on bare metal or private cloud.
- Teams wanting **full control and understanding** of their cluster’s components.
- Environments that require **high availability** without external dependencies.

---

> ⚠️ **Note:** This guide covers the minimal requirements for a working HA Kubernetes cluster.  
> Supplement with security hardening, monitoring, and backup as needed.

---

# 🏗️ Deploying the Kubernetes Cluster

---

## 1️⃣ Configuring Nodes

Before installing Kubernetes, configure each node for performance and compatibility.

#### 1.1. Update `/etc/hosts`

Add all nodes' IP addresses and hostnames, including the VIP:

```sh
192.168.1.200    cluster-endpoint
192.168.1.100    k8s-master-01
192.168.1.101    k8s-worker-01
# ...add more nodes as needed
```

#### 1.2. Disable Swap & Configure Kernel Modules

```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
sudo reboot now
```

---

## 2️⃣ Install containerd (Container Runtime)

```sh
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt install containerd.io

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

# Edit config.toml to set SystemdCgroup = true:
sudo vim /etc/containerd/config.toml
# (Find and change to SystemdCgroup = true)

sudo systemctl restart containerd
sudo systemctl enable containerd
```

<p align="center">
  <img src="https://raw.githubusercontent.com/singudotdev/k8s-install/refs/heads/master/img/cgroup.png" alt="SystemdCgroup = true"/>
</p>

---

## 3️⃣ Install kubeadm, kubelet, and kubectl

```sh
export K8S_VERSION=v1.33
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/$K8S_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$K8S_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

---

## 4️⃣ Set Up Kube-VIP LoadBalancer (HA VIP)

**On each control plane node:**

```sh
export VIP=192.168.1.200       # Your desired VIP
export INTERFACE=ens1          # Your network interface
export USERNAME=your_user_non_root

sudo ip addr add $VIP/24 dev $INTERFACE

mkdir -p /home/${USERNAME}/k8s
touch /home/${USERNAME}/k8s/kube-vip.yaml

KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")

alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"

kube-vip manifest daemonset \
    --interface $INTERFACE \
    --address $VIP \
    --inCluster \
    --taint \
    --controlplane \
    --services \
    --arp \
    --leaderElection | tee /home/${USERNAME}/k8s/kube-vip.yaml
```

---

## 5️⃣ Initialize the Cluster

**On the first master node:**

```sh
sudo kubeadm init --skip-phases=addon/kube-proxy --control-plane-endpoint cluster-endpoint --upload-certs

# Configure kubectl for your user:
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Add additional control plane nodes as required.**

---

## 6️⃣ Deploy Kube-VIP to the Cluster

```sh
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
kubectl apply -f /home/${USERNAME}/k8s/kube-vip.yaml
```

---

## 7️⃣ Install Cilium CNI (proxy-free)

```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
CILIUM_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium/main/stable.txt)
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Enable WireGuard encryption, kube-proxy replacement, and install Cilium:
cilium install --version $CILIUM_VERSION \
  --set encryption.nodeEncryption=true \
  --set encryption.type=wireguard \
  --set kubeProxyReplacement=true

cilium status --wait

# Enable L7 proxying:
cilium config set enable-l7-proxy true
```

---

## 🎉 Done!

You now have a **minimal, highly available, production-grade Kubernetes cluster** with cloud-native HA, modern secure networking, and no unnecessary complexity.

---

> 🔒 **Next Steps:**  
> Add monitoring, backup, and security hardening tailored to your workload and environment.

---
