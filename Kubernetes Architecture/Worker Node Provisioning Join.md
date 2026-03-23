# Kubernetes Worker Node Join Procedure

**Tags:** #kubernetes #worker-node #kubeadm #installation #networking #centos
**Status:** Implementation Guide

---

## 1. Hostname & DNS Configuration

Since we are setting up a new Worker Node, it must have a unique identity.

### Set Unique Hostname
Execute this on the **Worker Node** to differentiate it from the Master.

```bash
sudo hostnamectl set-hostname <NEW_HOSTNAME>

```

### Update Hosts File (DNS Resolution)

Since there is no DNS server, we must manually map IPs to Hostnames on **BOTH** the Master and the Worker nodes.

**Edit File:** `/etc/hosts`

Add the following lines to the file on both machines:

```text
172.16.10.10  master-node
172.16.10.11  worker-node-01
# Add other nodes as needed

```

---

## 2. Environment Preparation and Network Setup

Proper network configuration is critical before installing Kubernetes components to ensure node communication and reachability.

### Port Verification

Check if the default Kubernetes API Server port (6443) is free to avoid installation conflicts.

```bash
nc 127.0.0.1 6443 -zv -w 2

```

* **Purpose:** Ensures no other process is utilizing the port required by the API Server.

### Static IP Configuration

Kubernetes nodes require static IPs to maintain communication after restarts.

```bash
sudo nmcli con mod ens33 ipv4.addresses 172.16.10.10/24 ipv4.method manual
sudo nmcli con mod ens33 ipv4.gateway 172.16.10.2
sudo nmcli con mod ens33 ipv4.dns "8.8.8.8,1.1.1.1"
sudo nmcli con up ens33

```

* **Purpose:** Prevents the Node IP from changing, which allows stable communication with the Control Plane.

### Hostname Mapping

Map the static IP to the local hostname to ensure the API server can resolve the node name locally.

```bash
echo "172.16.10.10 creativehere" | sudo tee -a /etc/hosts

```

### IP Forwarding

Enable IPv4 forwarding to allow traffic to be routed between pods and external networks.

```bash
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

```

* **Effect:** Required for the Pod Network Interface (CNI) to function correctly.

---

## 3. System Optimization and Swap Management

Kubernetes is designed to manage resources at the container level and does not support system swap.

### Disable Swap

```bash
sudo swapoff -a
sudo vim /etc/fstab # Comment out any line containing the word 'swap'

```

* **Purpose:** Ensures the Kubelet can accurately manage resource limits and guarantees performance stability.

### SELinux Configuration

Set SELinux to permissive mode to allow containers to access the host filesystem.

**Edit File:** `/etc/selinux/config`

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

```

---

## 4. Container Runtime Interface (CRI): Containerd

Containerd acts as the engine that pulls images and manages the lifecycle of containers.

### Installation

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y containerd.io

```

### Configuration and Systemd Integration

Generate the default configuration and enable `SystemdCgroup` to align with Kubernetes cgroup management.

**File Created:** `/etc/containerd/config.toml`

```bash
sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml

# Edit /etc/containerd/config.toml: set SystemdCgroup = true
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

```

* **Purpose:** Using Systemd as the cgroup driver ensures system stability under heavy load.

---

## 5. Kubernetes Tools Installation (Kubeadm, Kubelet)

* **Kubeadm:** The tool used to bootstrap the cluster.
* **Kubelet:** The agent that runs on all nodes to manage containers.

### Repository Setup and Installation

**File Created:** `/etc/yum.repos.d/kubernetes.repo`

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo yum install -y kubelet kubeadm --disableexcludes=kubernetes 

```

### Enable and Start Services

Ensure both the Runtime and the Kubelet are enabled.

```bash
sudo systemctl enable --now containerd
sudo systemctl enable --now kubelet
sudo systemctl start --now containerd
sudo systemctl start --now kubelet

```

---

## 6. Joining the Cluster

### On Control Plane (Master Node)

Run this command to generate the secure token and hash required for joining.

```bash
sudo kubeadm token create --print-join-command

```

### On Worker Node

Paste the output from the previous step here (prepend with sudo).

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>

```