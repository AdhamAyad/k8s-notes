# Kubernetes Node Operations: Drain, Cordon, and Uncordon

**Tags:** #kubernetes #nodes #maintenance #cka #architecture
**Status:** Core Operations Guide

---

## 1. Core Concept: Node Maintenance

When a Kubernetes Node requires physical maintenance, a kernel upgrade, or downscaling, you cannot simply power it off. Abruptly shutting down a Node will cause immediate downtime for the Pods running on it. 

Kubernetes provides three specific commands (`cordon`, `drain`, and `uncordon`) to gracefully handle Node maintenance with **Zero Downtime**, ensuring that applications are safely relocated and traffic is not disrupted.

---

## 2. The Commands Explained

### A. `kubectl cordon <node-name>` (The Gentle Block)
**Purpose:** Marks the Node as `SchedulingDisabled` (Unschedulable).
* **Action:** It applies an invisible `NoSchedule` taint to the Node. The Kube-Scheduler will immediately stop placing any *new* Pods on this Node.
* **Impact on Existing Pods:** **None.** Pods currently running on the Node remain active, healthy, and continue to serve traffic.
* **Use Case:** Soft maintenance, troubleshooting a misbehaving Node without causing an outage, or gradually emptying a Node as temporary jobs finish natively.

### B. `kubectl drain <node-name>` (The Safe Eviction)
**Purpose:** Prepares a Node for immediate shutdown or reboot by removing all workloads safely.
* **Action 1 (Cordon):** It automatically cordons the Node first to prevent new Pods from arriving.
* **Action 2 (Graceful Eviction):** It sends a `SIGTERM` signal to all running Pods, allowing them to save state and shut down gracefully.
* **Action 3 (Rescheduling):** Controllers (like Deployments or ReplicaSets) detect the missing Pods and immediately recreate them on *other* available, healthy Nodes in the cluster.
* **Use Case:** Hard maintenance, kernel upgrades, instance type resizing, or permanent cluster scale-down.

### C. `kubectl uncordon <node-name>` (The Re-entry)
**Purpose:** Returns a previously cordoned or drained Node back to active service.
* **Action:** Removes the `SchedulingDisabled` status. The Kube-Scheduler can now assign new Pods to this Node.
* **Architectural Note:** Uncordoning a Node **does not** automatically move the old Pods back to it. The Node will start empty and only receive newly created Pods.

---

## 3. `kubectl drain` Safety Checks & Required Flags

Because `drain` is a destructive action, Kubernetes implements strict safety checks. If certain conditions exist, the command will halt and throw an error to protect your cluster. You must use specific flags to bypass these protections.

> [!WARNING] The 3 Crucial Flags for `drain` (CKA Exam Focus)
>
> **1. `--ignore-daemonsets`**
> * *The Problem:* DaemonSets (like logging agents or CNI plugins) run a Pod on *every* Node. The drain command cannot permanently evict them because the DaemonSet controller will instantly replace them, bypassing the cordon.
> * *The Solution:* Use this flag to tell the drain process to ignore DaemonSet Pods and proceed with evicting the normal application Pods.
>
> **2. `--delete-emptydir-data`**
> * *The Problem:* If a Pod uses an `emptyDir` volume (local temporary storage on the Node), draining the Pod will permanently erase that data. Kubernetes halts the drain to prevent accidental data loss.
> * *The Solution:* Use this flag to explicitly acknowledge and consent to the deletion of the local `emptyDir` data.
>
> **3. `--force`**
> * *The Problem:* If a Pod is "naked" or "standalone" (created without a controller like a Deployment or ReplicaSet), draining it will delete it forever. It will not be recreated on another Node.
> * *The Solution:* Use this flag to force the deletion of unmanaged, orphaned Pods.

**The Ultimate Production/Exam Command:**
```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force
````

---

## 4. Advanced Architecture: Drain vs. Taints vs. PDBs

### Why use `drain` instead of adding a `NoExecute` Taint?

While a `NoExecute` taint will forcefully kick Pods off a Node, it acts like a bulldozer. `drain` acts like a surgeon because it respects **PodDisruptionBudgets (PDB)**.

> [!IMPORTANT] Respecting the Pod Disruption Budget (PDB)
> 
> A PDB is a policy that guarantees a minimum number of replicas must be available at all times (e.g., `minAvailable: 2`).
> 
> - **If you use a Taint:** It ignores the PDB and instantly kills the Pods, potentially causing downtime.
>     
> - **If you use `drain`:** It reads the PDB. If evicting a Pod would violate the budget (e.g., dropping replicas to 1), the `drain` command will **pause and wait**. It will not kill the Pod until the cluster has spun up a replacement on another Node, guaranteeing zero downtime.
>     

### Parallel Draining

You can execute `kubectl drain` on multiple Nodes simultaneously in different terminal windows. Because the API server processes the eviction API centrally, Kubernetes will coordinate the parallel drains to ensure that your PodDisruptionBudgets are strictly respected across the entire cluster.