# Kubernetes Cluster Architecture and Components

**Tags:** #kubernetes #architecture #control-plane #nodes #cka #networking

**Status:** Core Reference

## 1. High-Level Architecture

A Kubernetes cluster is a distributed system consisting of two main categories of components:

1. **The Control Plane:** The "Brain" of the cluster. It manages the worker nodes and the Pods. It makes global decisions (scheduling), detects events, and responds to changes.
    
2. **The Worker Nodes:** The "Muscles" of the cluster. These are the machines (VMs or physical servers) that run the actual containerized applications.
    


![The control plane (kube-apiserver, etcd, kube-controller-manager, kube-scheduler) and several nodes. Each node is running a kubelet and kube-proxy.](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

---

## 2. Control Plane Components

The control plane components can run on any machine, but setup scripts usually start them on the same machine (referred to as the "Master Node").

### A. kube-apiserver

- **Function:** The front end for the Kubernetes control plane. It exposes the Kubernetes API.
    
- **Role:**
    
    - Acts as the central "hub" for all communication. No component talks to another directly; they all go through the API Server.
        
    - Authenticates and validates requests (User -> API Server).
        
    - The only component that reads from and writes to `etcd`.
        
- **Scalability:** Designed to scale horizontally (active-active) to balance traffic.
    
- **Default Port:** 6443.
    

### B. etcd

- **Function:** Consistent and highly-available key-value store.
    
- **Role:**
    
    - Kubernetes' backing store for **all** cluster data.
        
    - **Rule:** "If it is not in etcd, it did not happen."
        
- **Critical Note:** Requires a robust backup plan.
    
- **Ports:**
    
    - **2379:** Client communication (API Server talking to etcd).
        
    - **2380:** Peer communication (etcd nodes syncing with each other).
        

### C. kube-scheduler

- **Function:** Watches for newly created Pods with no assigned `nodeName` and selects a node for them.
    
- **Scheduling Logic (2 Phases):**
    
    1. **Filtering:** Removes nodes that do not meet resource requirements (CPU/Memory) or constraints (Taints/Affinity).
        
    2. **Ranking (Scoring):** Scores the remaining nodes (0-10) to find the optimal fit.
        
- **Role:** It _decides_ where the pod goes but does _not_ execute the running of the pod.
    

### D. kube-controller-manager

- **Function:** Runs controller processes (The "Reconciliation Loops").
    
- **Architecture:** Logically, each controller is a separate process, but they are compiled into a single binary to reduce complexity.
    
- **Key Controllers:**
    
    - **Node Controller:** Monitors node health and responds when nodes go down.
        
    - **Replication Controller:** Ensures the correct number of pods are running.
        
    - **EndpointSlice Controller:** Links Services and Pods.
        
    - **ServiceAccount Controller:** Creates default accounts.
        

### E. cloud-controller-manager

- **Function:** Embeds cloud-specific control logic.
    
- **Role:** Separates components that interact with the cloud platform (AWS, Azure, GCP) from generic cluster components.
    
- **Sub-controllers:** Node Controller (cloud deletion checks), Route Controller (cloud routing tables), Service Controller (cloud load balancers).
    

---

## 3. Node Components

These components run on **every** worker node to maintain running pods.

### A. kubelet

- **Function:** The primary agent running on each node. The "Captain of the Ship".
    
- **Role:**
    
    - Registers the node with the cluster.
        
    - Takes `PodSpecs` from the API Server and ensures the containers described are running and healthy.
        
    - Communicates directly with the Container Runtime (CRI).
        
- **Installation:** Usually installed as a systemd binary service, not a Pod.
    

### B. kube-proxy

- **Function:** Network proxy running on each node.
    
- **Role:**
    
    - Implements the **Service** concept.
        
    - Maintains network rules (via **iptables** or IPVS) on the node.
        
    - **Responsibility:** It handles Layer 4 (TCP/UDP) forwarding. It ensures that traffic destined for a virtual Service IP is redirected to the actual backend Pod IP.
        
- **Relationship with CNI:** It works in tandem with the CNI Plugin. They complement each other to form the full networking stack.
    

> [!INFO] Comparison: CNI vs. Kube-Proxy
> 
> These two components work at different layers to enable networking:
> 
> |**Feature**|**CNI Plugin (e.g., Calico, Flannel)**|**Kube-Proxy**|
> |---|---|---|
> |**Primary Role**|**Connectivity (The Road).**|**Services (The Traffic Signs).**|
> |**Layer**|Mostly **Layer 3** (IP Routing).|Mostly **Layer 4** (Load Balancing/NAT).|
> |**Function**|Assigns IP addresses to Pods and creates routes so Pod A can ping Pod B on a different node.|Manipulates `iptables` rules to intercept traffic meant for a Service IP and forward it to a Pod.|
> |**Dependency**|Essential for Pod-to-Pod communication.|Essential for Service discovery and load balancing.|

### C. Container Runtime

- **Function:** The software that effectively runs the containers.
    
- **Examples:** containerd, CRI-O, Docker Engine (via CRI-dockerd).
    

---

## 4. The Pod Creation Workflow

This sequence illustrates how components interact to schedule and run a Pod:

1. **User:** Sends `kubectl run` (POST request) -> **API Server**.
    
2. **API Server:** Authenticates, Validates, and writes to **etcd**. (Status: Pending).
    
3. **Scheduler:** Watches API Server. Sees unassigned Pod.
    
    - Filters & Ranks nodes.
        
    - Assigns node (Bind) -> **API Server**.
        
4. **API Server:** Updates **etcd** with `nodeName`.
    
5. **Kubelet:** Watches API Server. Sees Pod assigned to its Node.
    
    - Instructs **Container Runtime** to start container.
        
    - Updates status -> **API Server**.
        

---

## 5. Container Runtimes & CLI Tools

### Evolution

- **Docker Shim:** Removed in k8s v1.24+.
    
- **CRI:** Kubernetes now communicates directly with CRI-compliant runtimes (containerd/CRI-O).
    

### CLI Tools Cheat Sheet

|**Tool**|**Target Audience**|**Purpose**|
|---|---|---|
|**crictl**|**Admins / CKA Exam**|**Debugging.** Used to inspect pods/containers at the runtime level on the node. Pre-installed with k8s.|
|**nerdctl**|**Developers**|**Docker Replacement.** Provides `docker run`, `docker build`, `docker compose` functionality for containerd.|
|**ctr**|**Low-Level Admins**|**Hardcore Debugging.** Bundled with containerd. Not user-friendly.|

---

## 6. Addons (Cluster Features)

Addons use Kubernetes resources (DaemonSet, Deployment) to implement cluster features. They reside in the `kube-system` namespace.

- **DNS (CoreDNS):** **Mandatory.** Serves DNS records for Services.
    
- **Web UI (Dashboard):** Web-based management.
    
- **Network Plugins (CNI):** Software that implements the Container Network Interface specification (allocating IPs).
    

---

## 7. Ports Reference Table

|**Component**|**Port**|**Description**|
|---|---|---|
|**kube-apiserver**|6443|API Server (HTTPS)|
|**etcd**|2379|Client Request|
|**etcd**|2380|Peer Communication|
|**kubelet**|10250|Kubelet API|
|**kube-scheduler**|10259|Scheduler API|
|**kube-controller-manager**|10257|Controller Manager API|
|**NodePort Services**|30000-32767|Service Range|
