# Kubernetes Controllers: Deployment

**Tags:** #kubernetes #deployment #controller #updates #stateless #CKA

**Status:** Core Concept

---

## 1. Overview

A **Deployment** is a higher-level Kubernetes object that manages **[[ReplicaSet]]** and provides declarative updates to Pods. It is the standard way to deploy stateless applications (e.g., Web Servers, Microservices).

### Core Function
* **Lifecycle Management:** Automates the transition from one version of an application to another.
* **Replica Management:** Ensures the desired number of Pods are running (by orchestrating the underlying **[[ReplicaSet]]**).
* **Rollback Capability:** Maintains a history of changes, allowing you to revert to a previous state instantly.

---

## 2. Update Strategies

Defined in `.spec.strategy.type`. This determines *how* the pods are replaced during an update.



| Strategy | Description | Pros | Cons |
| :--- | :--- | :--- | :--- |
| **RollingUpdate** (Default) | Replaces old Pods with new ones gradually. It scales up a new **[[ReplicaSet]]** while scaling down the old one. | **Zero Downtime.** Service remains available throughout the process. | Complex to tune. App must support running multiple versions simultaneously during the transition. |
| **Recreate** | Terminates **all** existing Pods first. Once they are dead, it creates the new Pods. | **Simple.** Guarantees no two versions run at the same time (good for legacy databases). | **Downtime.** The service goes offline completely between the kill and the create steps. |

### Deep Dive: The `Recreate` Strategy
When selecting `Recreate`, the behavior is strictly sequential:
1.  **Scale Down:** Old ReplicaSet is scaled to 0 immediately.
2.  **Wait:** Controller waits for all pods to terminate.
3.  **Scale Up:** New ReplicaSet is scaled to the desired count.

> [!WARNING] Configuration Note
> When using `type: Recreate`, you **must not** include `rollingUpdate` parameters (like `maxSurge` or `maxUnavailable`). They are incompatible because the strategy does not support surging or partial unavailability—it kills everything at once.

---

## 3. Tuning Rolling Updates (RollingUpdate Only)

When using `RollingUpdate`, you can control the speed and safety of the rollout using two parameters under `.spec.strategy.rollingUpdate`.

### A. `maxSurge` (The Accelerator) 
* **Definition:** The number (or percentage) of Pods that can be created **above** the desired replica count.
* **Function:** Determines how fast the new version starts.
* **Example:** If Replicas=10 and `maxSurge=2`, Kubernetes can temporarily scale up to **12** Pods.

### B. `maxUnavailable` (The Brake) 
* **Definition:** The number (or percentage) of Pods that can be unavailable **below** the desired replica count.
* **Function:** Determines the minimum availability during the update.
* **Example:** If Replicas=10 and `maxUnavailable=1`, Kubernetes ensures at least **9** Pods are always running.

---

## 4. Stability & History Settings 

These settings optimize the "quality" of the deployment and prevent broken updates.

### A. `minReadySeconds` (The Soak Period)
* **Location:** `.spec.minReadySeconds`
* **Default:** `0`
* **Concept:** A "Probation Period" for new Pods.
* **How it works:** It starts counting **AFTER** the Readiness Probe succeeds (Pod is Ready).
    * **Traffic:** Starts immediately (Pod receives users).
    * **Rollout:** Pauses until the timer finishes.
* **Impact:**
    * If the Pod crashes *during* this countdown, the Deployment stops the rollout (Safety Stop).
    * If the Pod survives the countdown, it is marked "Available" and the rollout continues to the next pod.

### B. Failure Behavior (No Auto-Rollback) 
Just like DaemonSets, Deployments do **NOT** automatically rollback if an update fails.
1.  **Stuck State:** If new pods crash, the Deployment pauses. It creates a mix of old (working) and new (broken) pods.
2.  **Manual Fix:** You must explicitly run `kubectl rollout undo deployment/name` to revert to the stable version.

### C. `revisionHistoryLimit` (Cleanup)
* **Location:** `.spec.revisionHistoryLimit`
* **Default:** `10`
* **Purpose:** The number of old **[[ReplicaSet]]**s to retain to allow rollback.
* **Why use it?** To keep `etcd` clean. A value of `2-5` is usually sufficient. Setting this to `0` disables rollback functionality.

---

## 5. Update Methods: Declarative vs. Imperative

There are multiple ways to trigger an update.

| Method | Command | Use Case | Risk |
| :--- | :--- | :--- | :--- |
| **Declarative** (Recommended) | `kubectl apply -f file.yaml` | Production / GitOps. Keeps the file as the "Source of Truth". | None. Ideally, this is the only method used in stable environments. |
| **Imperative** (CLI) | `kubectl set image deployment/app nginx=nginx:1.19` | Testing / Quick Fixes. Very fast. | **High.** It bypasses the YAML file. If you run `apply` later, it will overwrite this change. |
| **Edit** (Interactive) | `kubectl edit deployment/app` | Debugging. Opens live config in `vi`/`nano`. | **Medium.** Easy to make syntax errors. |

---

## 6. Advanced Pattern: Canary Deployment

Kubernetes does not have a native "Canary" object. It is implemented using two Deployments and one Service.

### Architecture
1.  **Stable Deployment:** 90% of Replicas. Image: v1. Label: `app=front`, `track=stable`.
2.  **Canary Deployment:** 10% of Replicas. Image: v2. Label: `app=front`, `track=canary`.
3.  **Service:** Selector: `app=front`.

### Traffic Flow
The Service creates a singular LoadBalancer across **all** pods with the label `app=front`. Traffic is distributed randomly, effectively sending a small percentage of users to the Canary version based on the replica ratio.

---

## 7. Deployment vs. DaemonSet (Quick Comparison)

While both manage pods, their update logic differs fundamentally.

| Feature              | Deployment                         | [[Daemonset]]                                                 |
| :------------------- | :--------------------------------- | :------------------------------------------------------------ |
| **Goal** | Run X copies of an app.            | Run 1 copy on **every** node.                                 |
| **Default Strategy** | `RollingUpdate`                    | `RollingUpdate`                                               |
| **Manual Strategy** | `Recreate` (Kill all, then start). | `OnDelete` (Update definition, wait for user to delete pods). |

---

## 8. Master YAML Reference

This reference includes the **Standard Configuration (RollingUpdate)** and shows exactly how to swap the strategy block for **Recreate**.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 10
  
  # ---------------------------------------------------------
  # 1. HISTORY & STABILITY (Best Practices)
  # ---------------------------------------------------------
  revisionHistoryLimit: 5       # Keep last 5 ReplicaSets for rollback
  minReadySeconds: 10           # Wait 10s AFTER readiness probe before considering a Pod "Available"
                                # Note: Traffic enters immediately, but rollout pauses.

  # ---------------------------------------------------------
  # 2. STRATEGY SELECTION
  # ---------------------------------------------------------
  
  # OPTION A: RollingUpdate (Default - Zero Downtime)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2               # Allow 2 extra pods during update
      maxUnavailable: 1         # Allow only 1 pod to be down

  # OPTION B: Recreate (Downtime - Kill all first)
  # To use Recreate, comment out Option A and uncomment below:
  # strategy:
  #   type: Recreate
  #   # Note: Do NOT add rollingUpdate params here.
  
  # ---------------------------------------------------------
  # 3. SELECTOR & TEMPLATE
  # ---------------------------------------------------------
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.14