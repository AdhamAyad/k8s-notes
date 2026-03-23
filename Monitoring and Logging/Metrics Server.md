# Kubernetes Architecture: Metrics Server & Logging

**Tags:** #kubernetes #monitoring #metrics-server #logging #cka #troubleshooting
**Status:** Implementation Guide
**Reference:** [Metrics Server GitHub Repo](https://github.com/kubernetes-sigs/metrics-server/tree/master)

---

## 1. Core Concept: What is Metrics Server?

The **Metrics Server** is a scalable, efficient source of container resource metrics for Kubernetes built-in autoscaling pipelines. It collects resource metrics (CPU and Memory) from the `kubelets` running on each Node and exposes them in the Kubernetes API server through the Metrics API.



* **Data Source:** It relies on `cAdvisor`, which is integrated directly into the `kubelet` on every node.
* **Architecture (Deployment vs. DaemonSet):** It runs as a standard **Deployment** (a central aggregator), not a DaemonSet. It does not need to run physically on every node because it pulls data over the network via API calls to the kubelets.
* **Dependencies:** It is strictly required for the `HorizontalPodAutoscaler (HPA)` to function and for the `kubectl top` commands to yield results.

---

## 2. Installation & Fixing the TLS Issue (kubeadm / Test Environments)

In training environments or clusters built with `kubeadm`, the Metrics Server often fails with a `Metrics API not available` error. This occurs because the Metrics Server attempts to verify the TLS certificates of the kubelets, which are usually self-signed in test environments. 

### Step 1: Install Metrics Server
Download and apply the official components manifest:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
````

### Step 2: Edit the Deployment

Open the deployment configuration in your default editor to bypass the TLS verification:

Bash

```
kubectl edit deployment metrics-server -n kube-system
```

### Step 3: Add the Insecure TLS Flag

Locate the `containers` section, specifically the `args` array, and append the `--kubelet-insecure-tls` flag.

**YAML Configuration:**

YAML

```
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  # <==== Add this line here
        image: registry.k8s.io/metrics-server/metrics-server:v0.7.2
```

> [!INFO] Dynamic Reloading
> 
> Once you save and exit the editor (`:wq`), Kubernetes will automatically terminate the old Metrics Server Pod and spin up a new one with the updated arguments.

> [!IMPORTANT] Data Gathering Delay
> 
> Do not run `kubectl top` immediately. Wait approximately 15 to 30 seconds for the new Pod to transition to the `Running` state and actively aggregate the initial data from the nodes.

---

## 3. Resource Monitoring Commands

Once the Metrics Server is fully operational, you can monitor the hardware utilization across your cluster.

Bash

```
# Display CPU and Memory usage for all Nodes
kubectl top node

# Display CPU and Memory usage for all Pods in the current Namespace
kubectl top pod

# Display CPU and Memory usage for all Pods across all Namespaces
kubectl top pod -A
```

---

## 4. Viewing Application Logs

Monitoring logs is critical for troubleshooting applications and verifying container behavior.

Bash

```
# Fetch the static logs of a single-container Pod
kubectl logs <pod-name>

# Stream the logs continuously (similar to 'tail -f' in Linux)
kubectl logs <pod-name> -f

# Stream the logs of a specific container inside a Multi-container Pod
kubectl logs <pod-name> -f -c <container-name>
```

> [!WARNING] Multi-Container Pod Execution
> 
> If you run `kubectl logs <pod-name>` on a Pod that contains more than one container, Kubernetes will reject the command and output an error. You must explicitly specify which container's logs you want to view using the `-c` flag.