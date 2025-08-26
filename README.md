# 🚀 Kubernetes On-Premise Cluster Deployment Guide

---

## ✨ Purpose

This guide provides a concise, step-by-step approach to deploying a **fully functional, production-ready Kubernetes cluster on-premises** with **only the essential components** required to get up and running. The goal is to deliver a minimal, maintainable, and robust baseline suitable for enterprise environments, while keeping configuration complexity low and avoiding unnecessary extras.

You will use:

- **containerd** as the container runtime (CRI)
- **kubeadm** for Kubernetes bootstrapping and management
- **kube-vip** for a highly available virtual IP (VIP) endpoint for the control plane
- **Cilium** as the CNI (Container Network Interface), providing advanced, kube-proxy-free networking and security

---

## 💡 Why This Approach Is Best Practice

- **Minimalism:** Installs only what is strictly necessary for a working, highly available Kubernetes cluster.
- **Cloud-Native:** Uses Kubernetes-native tools (kubeadm, kube-vip, Cilium) that are widely adopted and actively maintained.
- **Simplicity:** Manual, explicit configuration steps ensure transparency and make troubleshooting straightforward.
- **High Availability:** Leverages kube-vip for simple, cloud-agnostic HA of the API server endpoint, without needing external load balancers or complex legacy tools.
- **Modern Networking:** Employs Cilium for high-performance, eBPF-powered networking and security, enabling features like direct routing, WireGuard encryption, and L7 policy enforcement.
- **Vendor Neutral:** No reliance on cloud or commercial vendor components; everything is open source and portable.
- **Production-Ready:** This setup is suitable for running production workloads, and forms a solid baseline for further customization (monitoring, backup, security hardening, etc.) as needed.

---

## 👤 Who Is This Guide For?

- Anyone who needs a **reproducible, enterprise-friendly, and minimal Kubernetes cluster** on bare metal or in a private datacenter.
- Teams who want to understand and control every component in their cluster.
- Environments where you want **high availability** without external dependencies.

---

> ⚠️ **Note:** This guide is intentionally focused on the minimal requirements for a working HA Kubernetes cluster.  
> You should supplement it with your own security, monitoring, and backup solutions as appropriate for your environment.

---

# 🏗️ Deploying the Kubernetes Cluster

---

## 1️⃣ Configuring Nodes

Before installing Kubernetes, configure each node for performance and compatibility.

### 1.1. Update `/etc/hosts`

> **Edit `/etc/hosts` on every node to include all nodes and the VIP:**

```sh
192.168.1.200    cluster-endpoint
192.168.1.100    k8s-master-01
192.168.1.101    k8s-worker-01
# ...add more nodes as needed
```

---

### 1.2. Disable Swap & Configure Kernel Modules

> **Disable swap and configure required kernel modules and sysctl params:**

```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Enable kernel modules for containerd networking
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf 
overlay 
br_netfilter
EOF

sudo modprobe overlay 
sudo modprobe br_netfilter

# Set sysctl params required by Kubernetes
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1 
net.bridge.bridge-nf-call-ip6tables = 1 
EOF

sudo sysctl --system   # Apply sysctl params
sudo reboot now        # Reboot to ensure all changes take effect
```

---

## 2️⃣ Install containerd (Container Runtime)

> **Install containerd and generate its config:**

```sh
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker's repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt install -y containerd.io

# Generate default containerd configuration file
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
```

> **Edit `/etc/containerd/config.toml` and set `SystemdCgroup = true`:**

```sh
sudo vim /etc/containerd/config.toml
# Find the line "SystemdCgroup = false" and change it to "SystemdCgroup = true"
```

<p align="center">
  <img src="https://raw.githubusercontent.com/singudotdev/k8s-install/refs/heads/master/img/cgroup.png" alt="SystemdCgroup = true"/>
</p>

```sh
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 3️⃣ Install kubeadm, kubelet, and kubectl

> **Install Kubernetes tools on every node:**

```sh
export K8S_VERSION=v1.33
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Import the Kubernetes APT repository key and repo
curl -fsSL https://pkgs.k8s.io/core:/stable:/$K8S_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$K8S_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

---

## 4️⃣ Install Kube-VIP LoadBalancer (VIP/HA)

> **🛑 All steps in this section must be run as `root`.**

Set the required environment variables:

```sh
export VIP=192.168.1.200      # Set to your cluster VIP
export INTERFACE=ens1         # The network interface to use
export USERNAME=your_user_non_root  # Change as appropriate
```

Assign the VIP to the network interface:

```sh
ip addr add $VIP/24 dev $INTERFACE
```

Create the manifest directory and file:

```sh
mkdir -p /home/${USERNAME}/k8s
touch /home/${USERNAME}/k8s/kube-vip.yaml
```

Download the latest kube-vip version and prepare the manifest generator:

```sh
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")

# Prepare kube-vip alias using containerd's ctr tool
alias kube-vip="ctr image pull ghcr.io/kube-vip/kube-vip:$KVVERSION; ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:$KVVERSION vip /kube-vip"
```

Generate the DaemonSet manifest for kube-vip:

```sh
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

> **Note:**  
> All the above commands must be executed as `root`.  
> This is required for assigning IP addresses, managing system network interfaces, and running containerd commands.

---

## 5️⃣ Create the Cluster

> **On the first control-plane (master) node, run:**

```sh
sudo kubeadm init --skip-phases=addon/kube-proxy --control-plane-endpoint cluster-endpoint --upload-certs 
```

> **Configure kubectl for your user:**

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 6️⃣ Deploy Kube-VIP to the Cluster

> **Apply the kube-vip RBAC and manifest (use the path from previous step):**

```sh
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
kubectl apply -f /home/${USERNAME}/k8s/kube-vip.yaml
```

---

## 7️⃣ Install Cilium CNI (Container Network Interface) and Remove Kube-Proxy

> **Install Cilium CLI and deploy Cilium with WireGuard encryption and kube-proxy replacement:**

```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
CILIUM_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium/main/stable.txt)
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install --version $CILIUM_VERSION \
  --set encryption.nodeEncryption=true \
  --set encryption.type=wireguard \
  --set kubeProxyReplacement=true

cilium status --wait

# Enable Cilium for L7 policies and proxying:
cilium config set enable-l7-proxy true
```

---

## 🎉 Done!

You now have a **minimal, highly available, and production-grade Kubernetes cluster**  
with modern, secure networking and no unnecessary complexity.

---

> 🔒 **Next Steps:**  
> Add monitoring, backup, and security hardening tailored to your workload and environment.

---
