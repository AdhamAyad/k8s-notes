# Kubernetes Cluster Upgrades (kubeadm)

**Tags:** #kubernetes #upgrade #kubeadm #architecture #cka
**Status:** Core Operations Guide

---

## 1. Core Architectural Concepts & Rules

Upgrading a Kubernetes cluster using `kubeadm` follows a strict, sequential architectural flow: **From the top down.** You must always upgrade the Control Plane components first, followed by the Worker Nodes. 

> [!WARNING] The Version Skew Policy (Golden Rules)
> 1. **No Skipping:** You cannot skip minor versions (e.g., upgrading directly from 1.32 to 1.34 is strictly forbidden). You must go 1.32 -> 1.33 -> 1.34.
> 2. **Component Match:** The project recommends matching versions across all components, but `kubelet` is allowed to be older than `kubeadm`/`kube-apiserver` (up to 3 minor versions behind). It can **never** be newer.



---

## 2. Phase 1: Repository & OS Preparation (All Nodes)

Kubernetes migrated its package repositories to a community-owned structure (`pkgs.k8s.io`), where each minor version has its own dedicated URL path. Before any upgrade, you must repoint your Linux package manager.

**Step 1: Verify Current OS and Repository**
```bash
# Check your OS distribution
cat /etc/*release*

# Verify current repository path
cat /etc/apt/sources.list.d/kubernetes.list
# Expected format: deb ... [https://pkgs.k8s.io/core:/stable:/v1.33/deb/](https://pkgs.k8s.io/core:/stable:/v1.33/deb/) /
````

**Step 2: Update Repository URL for Target Version (e.g., v1.34)**

Edit the file and change the version string in the URL to your target minor release:

Bash

```
# Example for changing to v1.34
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.34/deb/](https://pkgs.k8s.io/core:/stable:/v1.34/deb/) /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Refresh the package cache
sudo apt update
```

**Step 3: Find the Exact Package Revision (Crucial for CKA)**

Do not guess the version number. Always query the OS for the exact build string (e.g., `1.34.0-1.1`).

Bash

```
apt-cache madison kubeadm
```

---

## 3. Phase 2: Upgrading the Primary Control Plane

This is the most critical step. The first Control Plane node dictates the upgrade for the cluster's "brain" and the underlying `etcd` database.

**1. Unhold, Upgrade, and Hold `kubeadm`**

Bash

```
# Replace '1.34.x-*' with the exact version string from apt-cache madison
sudo apt-mark unhold kubeadm && \
sudo apt-get update && \
sudo apt-get install -y kubeadm='1.34.x-*' && \
sudo apt-mark hold kubeadm

# Verify version
kubeadm version
```

**2. Verify the Upgrade Plan (Dry Run)**

Bash

```
sudo kubeadm upgrade plan
```

_Architecture Note:_ This checks if the cluster is healthy, fetches target versions, and confirms `etcd` and static pod compatibility.

**3. Apply the Upgrade (The Heavy Lifting)**

Bash

```
# Use only the major.minor.patch here, NOT the OS build suffix (-1.1)
sudo kubeadm upgrade apply v1.34.x
```

_What happens under the hood?_

- Pulls new container images for API Server, Scheduler, Controller Manager, and etcd.
    
- Renews control plane certificates (unless disabled).
    
- Upgrades the static Pod manifests in `/etc/kubernetes/manifests`.
    
- Backs up the old state to `/etc/kubernetes/tmp`.
    

**4. Drain the Node**

Bash

```
kubectl drain <control-plane-name> --ignore-daemonsets
```

**5. Upgrade `kubelet` and `kubectl`**

Bash

```
# Use the exact version string with suffix here
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && \
sudo apt-get install -y kubelet='1.34.x-*' kubectl='1.34.x-*' && \
sudo apt-mark hold kubelet kubectl

# Restart the local worker agent
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

**6. Uncordon the Node**

Bash

```
kubectl uncordon <control-plane-name>
```

---

## 4. Phase 3: Upgrading Additional Control Planes (HA Clusters)

If you have multiple Control Plane nodes, the process is slightly different. **Never run `upgrade apply` more than once per cluster upgrade.**

1. Update repository and `kubeadm` package (Same as Phase 2, Step 1).
    
2. **Execute the Secondary Node Upgrade Command:**
    
    Bash
    
    ```
    sudo kubeadm upgrade node
    ```
    
    _Architecture Note:_ This fetches the already-upgraded `ClusterConfiguration`, updates local static pod manifests, and renews local certificates without touching the global `etcd` state.
    
3. Drain the node (`kubectl drain <node> --ignore-daemonsets`).
    
4. Upgrade `kubelet` and `kubectl` (Same as Phase 2, Step 5).
    
5. Uncordon the node.
    

---

## 5. Phase 4: Upgrading Worker Nodes

Workers are the simplest to upgrade, as they only require configuration syncs and binary updates. Do this one node at a time to maintain application availability.

1. Update repository and `kubeadm` package (Same as Phase 2, Step 1).
    
2. **Execute the Node Config Sync:**
    
    Bash
    
    ```
    sudo kubeadm upgrade node
    ```
    
    _Architecture Note:_ On a worker, this command solely upgrades the local `kubelet` configuration file to match any new policies defined by the upgraded Control Plane.
    
3. Drain the node from a control plane (`kubectl drain <node> --ignore-daemonsets`).
    
4. Upgrade `kubelet` and `kubectl` on the worker.
    
5. Restart the `kubelet` service.
    
6. Uncordon the node.
    

---

## 6. Advanced Scenarios & Troubleshooting

### Considerations for Heavy `etcd` Traffic

During `kubeadm upgrade apply`, the `kube-apiserver` static pod restarts. In-flight requests might stall or fail. To minimize disruption:

Bash

```
# Gracefully terminate the API server slightly before running 'upgrade apply'
killall -s SIGTERM kube-apiserver
sleep 20
sudo kubeadm upgrade apply v1.34.x
```

### Recovering from a Failure State

The `kubeadm upgrade apply` command is idempotent. If it fails midway (e.g., unexpected reboot), you can simply run it again.

If automatic rollback fails, you can manually restore the `etcd` data and static pod manifests from the backup directories automatically created during the upgrade process:

- `/etc/kubernetes/tmp/kubeadm-backup-etcd-<date>-<time>`
    
- `/etc/kubernetes/tmp/kubeadm-backup-manifests-<date>-<time>`
    

> [!INFO] Post-Upgrade Clean Up
> 
> Always run `kubectl get nodes` after completing all nodes to ensure they show `Ready` with the new version. The backup directories in `/etc/kubernetes/tmp` are not auto-deleted and should be cleared manually to save disk space once the cluster is confirmed stable.