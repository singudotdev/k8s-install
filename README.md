# Deploy Kubernetes cluster
Configuring nodes
---

Before proceed with the installation, we have to configure the nodes in order to be able to perform well.

<br/>

On each node, we need to disable swap, update "/etc/hosts" and install the CRI (Container Runtime Interface). In this case, containerd.

In the '/etc/hosts' we need to add all nodes IP with his hostname. 'cluster-endpoint' is the Virtual IP (VIP) that the cluster is going to have. Ex:

```sh
192.168.1.200    cluster-endpoint
192.168.1.100    k8s-master-01
192.168.1.101    k8s-worker-01
[...]
```

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
```

Make the changes take effect. A restart is needed:

```sh
sudo sysctl --system && sudo reboot now
```

Then, install containerd and generate configuration file:

```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install containerd
sudo apt install containerd.io

# Generate default containerd configuration file
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
```

Update the 'config.toml' for containerd:

```sh
sudo vim /etc/containerd/config.toml
```

<p align="center">
  <img src="https://raw.githubusercontent.com/singudotdev/k8s-install/refs/heads/master/img/cgroup.png" />
</p>

Save and exit. Then, restart containerd:

```sh
sudo systemctl restart containerd
sudo systemctl enable containerd
```

After doing this in all the nodes, we can proceed with the 'kubeadm' installation.

Installing kubeadm, kubelet and kubectl
---

```sh
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install tools
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet
sudo systemctl enable --now kubelet
```

Install Kube-VIP LoadBalancer
---

Create the necessary environment variables for make the setup faster (run as root):


```sh
export VIP=192.168.0.0
export INTERFACE=ens1
```

Assing to the current Network Interface the desired IP for the Cluster Endpoint (has to be free and not used by another system or service), install Kube-VIP and run it for generate the config file (run as root):

```sh
ip addr add $VIP/24 dev $INTERFACE

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
    --leaderElection | tee /home/singu/k8s/kube-vip.yaml
```

Create cluster
---

On 'master' node:

```sh
sudo kubeadm init --skip-phases=addon/kube-proxy --control-plane-endpoint cluster-endpoint --upload-certs 

# Configuring kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Then, add the control-planes to the cluster and execute:

```sh
kubectl apply -f https://kube-vip.io/manifests/rbac.yaml
kubectl apply -f /home/singu/k8s/kube-vip.yaml
```

Installing Cilium CNI (Container Network Interface) and remove Kube-Proxy:
---

```sh
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
CILIUM_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium/main/stable.txt)
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Enable WireGuard encryption, Kube-Proxy replacement and install Cilium
cilium install --version $CILIUM_VERSION \
  --set encryption.nodeEncryption=true \
  --set encryption.type=wireguard \
  --set kubeProxyReplacement=true

cilium status --wait

# Enable Cilium for L7
cilium config set enable-l7-proxy true
```
