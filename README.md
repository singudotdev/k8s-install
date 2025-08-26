# 🚀 Enterprise Kubernetes Platform: On-Premise/Cloud/Hybrid Architecture & Deployment Guide

---

## ✨ Purpose

This guide provides a full-stack, step-by-step approach for deploying a **highly available, production-ready Kubernetes platform** on-premises, in the cloud, or in a hybrid setup. It layers **essential cluster deployment steps** with **enterprise-grade architecture** for security, observability, resilience, and scalability.

You will build a platform with:

- **containerd** as the container runtime (CRI)
- **kubeadm** for Kubernetes bootstrapping and management
- **kube-vip** for a highly available virtual IP (VIP) endpoint for the control plane
- **Cilium** as the CNI and service mesh, providing eBPF-powered networking, security, and observability
- **Cloudflare** at the edge (security, CDN, global LB, Zero Trust)
- **Traefik OSS** for L7 ingress, API gateway, and observability
- **OPA/Gatekeeper, Falco, Vault** for security and compliance
- **Prometheus, Grafana, Opensearch, Opensearch Dashboards, Fluent Bit, Hubble, Jaeger** for observability
- **ArgoCD/Flux, backup/DR tools** for resilience and operations

---

## 💡 Why This Architecture Is Enterprise Best Practice

- **Minimal, robust cluster foundation**: Simple, reproducible, and reliable Kubernetes deployment using open-source, cloud-native tools.
- **Full-stack security**: From the edge (Cloudflare) to the pod (Cilium, OPA, Vault, Falco), you get Zero Trust, microsegmentation, runtime defense, and compliance.
- **Unified observability**: Centralized logging, tracing, metrics, and real-time network visibility from edge to pod.
- **Operational excellence**: GitOps, automated upgrades, backup/restore, and self-healing for day-2 reliability.
- **Scalable & flexible**: Supports on-premises, cloud, and hybrid; multi-cluster and multi-region ready.
- **Future-proof**: Uses modern standards (e.g., Gateway API, eBPF, GitOps), avoiding vendor lock-in.

---

## 👤 Who Is This Guide For?

- Enterprises and teams needing a **scalable, secure, and compliant Kubernetes platform** on bare metal, private cloud, or hybrid setups.
- Operators who want **transparency, automation, and control** at every layer.
- Organizations with **high-availability, compliance, and observability** needs.

---

> ⚠️ **Note:** This guide is intentionally focused on the minimal and essential requirements for a working enterprise-grade Kubernetes platform.  
> You should further supplement it with organization-specific security, monitoring, and backup/disaster recovery solutions.

---

# 🗂️ Platform Layers Overview

| Layer / Capability            | Main Components                    | Key Features / Benefits                                                                                                     |
|------------------------------|------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| **Edge Load Balancer & Security**    | Cloudflare                         | Global Anycast load balancing, DDoS/WAF, SSL offload, CDN, Argo Tunnel, Bot Management, Zero Trust Access            |
| **Ingress Controller**               | Traefik OSS                        | Dynamic L7 routing, mTLS, API gateway, Gateway API, observability                                                    |
| **Service Mesh & Networking**        | Cilium Service Mesh                | eBPF-powered mesh, sidecarless mTLS, L3/L4/L7 policies, L7 routing, Hubble observability, egress, multi-cluster      |
| **Security & Compliance**            | Cloudflare WAF, Cilium, OPA, Falco, Vault | Edge and internal segmentation, admission control, runtime security, secrets, CIS hardening                   |
| **Observability & Logging**          | Cloudflare Logs, Opensearch, Opensearch Dashboards, Fluent Bit, Hubble, Prometheus, Grafana, Jaeger | Centralized logging, SIEM, real-time network flow, metrics, alerting, tracing |
| **Platform Resilience & Operations** | ArgoCD/Flux, Operators, Self-Heal, Backup | GitOps, automated upgrades, disaster recovery, pod disruption budgets, failover                               |

---

# 🏗️ Deploying the Kubernetes Cluster (Foundation)

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

# 🏛️ Enterprise Architecture Layers (Build on the Foundation)

---

## 1. Edge Load Balancer & Security: **Cloudflare**

- **Global Anycast Load Balancing**
- **DDoS Protection & WAF**
- **SSL Offloading**
- **CDN & Geo Routing**
- **Argo Tunnel (no inbound ports)**
- **Bot Management & Zero Trust Access**

## 2. Ingress Controller: **Traefik OSS**

- **Dynamic L7 Routing**
- **mTLS & Automated TLS**
- **API Gateway Features**
- **Gateway API Support**
- **Prometheus, Jaeger, Hubble integration**

> **HA Setup:**  
> Deploy Traefik OSS with multiple replicas (Deployment or DaemonSet) behind a Kubernetes Service (type `LoadBalancer` or `NodePort`) for high availability.  
> This handles all ingress traffic to workloads running inside Kubernetes.

## 3. Service Mesh & Networking: **Cilium Service Mesh**

- **Sidecarless Service Mesh (eBPF)**
- **L3/L4/L7 Policies, mTLS, DNS-aware policies**
- **Advanced Traffic Control (LB, circuit breaking, canary)**
- **Hubble Observability**
- **Encryption (WireGuard/IPsec)**
- **Egress Gateway, Multi-Cluster Mesh**

## 4. Security & Compliance

- **Cloudflare WAF + Cilium Network Policies + OPA/Gatekeeper**
- **Admission Control**
- **Runtime Security (Falco)**
- **Secrets Management (Vault)**
- **Cluster Hardening (CIS, kube-bench)**
- **Image Scanning (Trivy, Clair or Aqua)**

## 5. Observability & Logging

- **Cloudflare Logs**
- **Fluent Bit** (agent on every node, collects and forwards logs)
- **Opensearch** (search and store logs)
- **Opensearch Dashboards** (view and analyze logs and metrics)
- **SIEM Integration** (optional)
- **Cilium Hubble for Network Flow Visibility**
- **Prometheus + Grafana for Metrics**
- **Jaeger for Tracing**
- **Alerting (policy violations, anomaly detection)**

### Example: Log Collection and Visualization Stack

1. **Fluent Bit** runs as a DaemonSet, collecting container and node logs.
2. **Fluent Bit** forwards logs to **Opensearch**.
3. **Opensearch Dashboards** provides a web UI for search, visualization, and alerting on logs.
4. Optional: export logs to a SIEM as required.

## 6. Platform Resilience & Operations

- **Automated Backup & Disaster Recovery**
- **GitOps (ArgoCD/Flux)**
- **Automated Upgrades via Operators**
- **Self-Healing, Pod Disruption Budgets, Multi-Zone/Cluster Failover**

---

# 🏗️ High-Level Architecture Diagram

```plaintext
[ Users / Clients ]
        |
    Cloudflare (Edge LB, WAF, Zero Trust)
        |
  Traefik OSS (Ingress/API Gateway)
        |
  ┌─────────────────────────────────────────────┐
  |      Kubernetes Cluster (on-prem/cloud)     |
  |---------------------------------------------|
  |   - kube-vip (HA control plane VIP)         |
  |   - containerd (CRI)                        |
  |   - kubeadm                                 |
  |   - Cilium (CNI, Service Mesh, Security)    |
  |   - Platform services, workloads            |
  └─────────────────────────────────────────────┘
        |
  Fluent Bit → Opensearch (+ Opensearch Dashboards), 
  Cilium Hubble, Prometheus, Grafana, etc.
        |
    GitOps Ops & Resilience Layer (ArgoCD, Backup)
```

# 📈 Benefits

- **Scalability:**  
  Global Anycast at the edge, elastic mesh networking, and cloud-native ingress let you scale seamlessly across datacenters, clouds, and regions.
- **Security:**  
  End-to-end protection, from edge (Cloudflare, WAF, Zero Trust) to pod (Cilium, OPA, Vault, Falco), ensures high trust, microsegmentation, and runtime defense.
- **Compliance:**  
  Policy-driven controls, automated audit trails, and supply chain security meet enterprise and regulatory needs.
- **High Availability:**  
  Multi-region failover, HA control plane, self-healing workloads, and resilient GitOps workflows maximize uptime.
- **Unified Observability:**  
  Real-time logs, metrics, and traces from Fluent Bit, Opensearch, Dashboards, Hubble, Prometheus, and Grafana provide actionable insights across the stack.
- **Operational Excellence:**  
  Automated upgrades, backup/recovery, disaster recovery, and unified, GitOps-driven operations reduce toil and risk.
- **Vendor Neutrality & Flexibility:**  
  100% open source, cloud-agnostic, and extensible—portable across any infrastructure.

---

# 🎉 Done!

You now have a **minimal, highly available, and enterprise-grade Kubernetes platform**  
with modern, secure networking, observability, and operational excellence—ready for any workload, anywhere.

> 🔒 **Next Steps:**  
> Add organization-specific monitoring, backup, security, and custom workflows as needed.
