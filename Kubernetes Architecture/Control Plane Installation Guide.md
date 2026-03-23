# Control Plane Setup (Master Node)

**Tags:** #kubernetes #control-plane #master-node #kubeadm #installation #networking #centos
**Status:** Implementation Guide

---

## 1. Environment Preparation and Network Setup

Proper network configuration is critical before installing Kubernetes components to ensure node communication and reachability.

### Port Verification
Check if the default Kubernetes API Server port (6443) is free to avoid installation conflicts.

```bash
nc 127.0.0.1 6443 -zv -w 2
````

- **Purpose:** Ensures no other process is utilizing the port required by the API Server.
    

### Static IP Configuration

Kubernetes nodes require static IPs to maintain communication after restarts.


```Bash
sudo nmcli con mod ens33 ipv4.addresses 172.16.10.10/24 ipv4.method manual
sudo nmcli con mod ens33 ipv4.gateway 172.16.10.2
sudo nmcli con mod ens33 ipv4.dns "8.8.8.8,1.1.1.1"
sudo nmcli con up ens33
```

- **Purpose:** Prevents the Control Plane IP from changing, which would otherwise invalidate the generated SSL certificates.
    

### Hostname Mapping

Map the static IP to the local hostname to ensure the API server can resolve the node name.


```Bash
echo "172.16.10.10 creativehere" | sudo tee -a /etc/hosts
```

### IP Forwarding

Enable IPv4 forwarding to allow traffic to be routed between pods and external networks.


```Bash
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

- **Effect:** Required for the Pod Network Interface (CNI) to function correctly.
    

---

## 2. System Optimization and Swap Management

Kubernetes is designed to manage resources at the container level and does not support system swap.

### Disable Swap


```Bash
sudo swapoff -a
sudo vim /etc/fstab # Comment out any line containing the word 'swap'
```

- **Purpose:** Ensures the Kubelet can accurately manage resource limits and guarantees performance stability.
    

### SELinux Configuration

Set SELinux to permissive mode to allow containers to access the host filesystem.

**Edit File:** `/etc/selinux/config`


```Bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

---

## 3. Container Runtime Interface (CRI): Containerd

Containerd acts as the engine that pulls images and manages the lifecycle of containers.

### Installation


```Bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y containerd.io
```

### Configuration and Systemd Integration

Generate the default configuration and enable `SystemdCgroup` to align with Kubernetes cgroup management.

**File Created:** `/etc/containerd/config.toml`


```Bash
sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml

# Edit /etc/containerd/config.toml: set SystemdCgroup = true
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

- **Purpose:** Using Systemd as the cgroup driver ensures system stability under heavy load.
    

---

## 4. Kubernetes Tools Installation (Kubeadm, Kubelet, Kubectl)

- **Kubeadm:** The tool used to bootstrap the cluster.
    
- **Kubelet:** The agent that runs on all nodes to manage containers.
    
- **Kubectl:** The command-line utility to interact with the cluster.
    

### Repository Setup and Installation

**File Created:** `/etc/yum.repos.d/kubernetes.repo`


```Bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=[https://pkgs.k8s.io/core:/stable:/v1.35/rpm/](https://pkgs.k8s.io/core:/stable:/v1.35/rpm/)
enabled=1
gpgcheck=1
gpgkey=[https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key](https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key)
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes 
sudo systemctl enable --now kubelet
```

---

## 5. Cluster Initialization (Kubeadm Init)

This stage creates the Control Plane components (API Server, etcd, Scheduler, Controller Manager).

### Execution


```Bash
sudo kubeadm init \
  --apiserver-advertise-address=172.16.10.10 \
  --pod-network-cidr=192.168.0.0/16 \
  --ignore-preflight-errors=Mem
```

- **`--apiserver-advertise-address`**: Sets the specific IP the API server will use to communicate with other nodes.
    
- **`--pod-network-cidr`**: Defines the IP range for pods, required by the Calico network plugin.
    
- **`--ignore-preflight-errors=Mem`**: Allows bypass of the 2GB RAM requirement for testing environments.
    

### Kubectl Access Configuration

Set up the admin configuration for the non-root user to manage the cluster.


```Bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 6. Networking and Add-ons

A cluster is not "Ready" until a Pod Network Add-on (CNI) is installed.

### Calico CNI Installation


```Bash
# Install Tigera Operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Install Custom Resources
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
#
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources-bpf.yaml

```


```

- **Result:** Once the Calico pods are running, the CoreDNS pods will transition from `Pending` to `Running`, and the Control Plane node status will change to `Ready`.
    

### Verification


```Bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl get pods -n calico-system
```