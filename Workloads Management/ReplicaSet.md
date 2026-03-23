# Kubernetes Controllers: ReplicaSet & ReplicationController

**Tags:** #kubernetes #replicaset #controller #scaling #availability #CKA

**Status:** Core Concept

---

## 1. Overview

A **ReplicaSet** is a Kubernetes workload controller used to guarantee the availability of a specified number of identical Pods. Its primary purpose is to maintain a stable, self-healing set of replica Pods running at any given time.

### Core Function

- **Stability:** Ensures the exact specified number of replicas are always running.
    
- **Self-Healing:** Automatically creates new Pods if existing ones crash, are deleted, or if a Node fails.
    
- **Scaling:** Can be manually or automatically scaled up or down to handle changes in load.
    

---

## 2. ReplicaSet vs. ReplicationController

While both serve the exact same purpose (maintaining pod counts), there is a strict generational difference between them based on how they filter and select Pods.

|**Feature**|**ReplicationController (Legacy)**|**ReplicaSet (Modern)**|
|---|---|---|
|**Status**|Deprecated / Legacy. Not recommended for modern clusters.|Active / The modern standard.|
|**Selector Type**|**Equality-Based.** Supports only strict, exact matches (e.g., `app = frontend`).|**Set-Based.** Supports complex conditions and sets.|
|**Manifest Syntax**|Labels are written directly under the `selector` field.|Requires `selector.matchLabels` and/or `selector.matchExpressions`.|

---

## 3. How It Works: The Selector Logic

A ReplicaSet operates on an infinite reconciliation loop, constantly comparing the _Current State_ of the cluster to the _Desired State_. It identifies its owned Pods using Selectors.

### A. matchLabels (The AND Operator)

When you use `matchLabels`, Kubernetes treats multiple labels as an **AND** condition.

- **Strict Minimum Requirement:** The Pod must contain _all_ the labels specified in the ReplicaSet's `matchLabels`.
    
- **Extra Labels:** If the Pod has the required labels _plus_ additional labels, it will still be adopted by the ReplicaSet. Extra labels do not break the match.
    
- **Missing Labels:** If the Pod is missing even one of the specified labels, the ReplicaSet will ignore it completely.
    

### B. matchExpressions (The OR / Complex Operator)

If you need flexibility (e.g., selecting pods that are either `frontend` OR `backend`), you cannot use `matchLabels`. You must use `matchExpressions`.

YAML

```
selector:
  matchExpressions:
    - {key: tier, operator: In, values: [frontend, backend]}
    - {key: environment, operator: NotIn, values: [testing]}
```

---

## 4. The Reconciliation Loop Scenarios

Understanding how the ReplicaSet reacts to manual cluster changes is critical for debugging.

### Scenario A: The "Bare Pod" Instant Deletion

**Condition:** A ReplicaSet is configured to maintain 2 Pods with the label `type: frontend`, and 2 Pods are currently running.

**Action:** You manually apply a manifest for a new bare Pod (`test-pod`) that includes the label `type: frontend`.

**Result:** The `test-pod` is terminated almost instantly.

**Why:** 1. The ReplicaSet immediately adopts the new pod because the labels match.

2. It evaluates the state: `Current Pods (3) > Desired Pods (2)`.

3. It must scale down. The algorithm prioritizes terminating the youngest or unready pods first.

4. Because `test-pod` is brand new and likely hasn't fully started its containers, it is purged from `etcd` immediately, often bypassing the visible `Terminating` state in the CLI.

### Scenario B: The Ignored Pod

**Condition:** A ReplicaSet expects Pods with `type: frontend` AND `env: prod`.

**Action:** You manually create a Pod with only `type: frontend`.

**Result:** The Pod runs peacefully as a standalone Pod. The ReplicaSet completely ignores it because it fails the strict **AND** condition of `matchLabels`.

---

## 5. ReplicaSet vs. Service

It is crucial to separate the concepts of Workload Controllers and Network Abstractions.

- **ReplicaSet (The Controller):** Focuses strictly on **Compute and State**. It does not care about network traffic. Its only job is to ensure the required number of Pods are physically running in the cluster.
    
- **Service (The Router):** Focuses strictly on **Networking and Connectivity**. It acts as a static load balancer (with a persistent IP and DNS name). It intercepts traffic and routes it to whichever Pods currently match its selectors, regardless of who created those Pods.
    

---

## 6. Implementation Pattern

The following YAML demonstrates the strict relationship between the Selector and the Template Labels.

YAML

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend-rs
  labels:
    app: guestbook
spec:
  # 1. REPLICAS: Desired state
  replicas: 3

  # 2. SELECTOR: How the RS finds its pods
  selector:
    matchLabels:
      tier: frontend

  # 3. POD TEMPLATE: The blueprint for new pods
  template:
    metadata:
      # CRITICAL: These labels MUST match the selector above.
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:latest
```

> [!WARNING] Production Best Practice
> 
> Never deploy a ReplicaSet directly in a production environment. Always deploy a **Deployment** object. The Deployment will automatically create and manage the underlying ReplicaSet, providing essential features like Rolling Updates and Rollbacks, which a raw ReplicaSet cannot do.

---

## 7. Scaling & Deletion Logic

### Scaling Down Algorithm

When a ReplicaSet scales down, it deletes Pods based on this priority:

1. **Status:** Pending or unschedulable pods are deleted first.
    
2. **Annotation:** Pods with a lower `controller.kubernetes.io/pod-deletion-cost` value.
    
3. **Availability:** Pods on nodes with higher replica density (to maintain high availability across nodes).
    
4. **Age:** Newer pods are deleted before older pods.
    

### Deletion Strategies (Garbage Collection)

When deleting a ReplicaSet, you must define what happens to its dependent Pods.

- **Background (Default):** `kubectl delete rs <name>`. The ReplicaSet object is deleted immediately. The Kubernetes Garbage Collector then asynchronously terminates the underlying Pods.
    
- **Foreground:** The ReplicaSet enters a "Terminating" state and blocks deletion until all underlying Pods are fully terminated.
    
- **Orphan:** `kubectl delete rs <name> --cascade=orphan`. Deletes the ReplicaSet but leaves the Pods running independently. Useful for migrating workloads to a new controller without downtime.
    

---

## 8. CLI Operations & Debugging

### Creation & Status

Bash

```
# Create the ReplicaSet
kubectl apply -f frontend.yaml

# List ReplicaSets
kubectl get rs

# Describe to view Events (useful for checking scaled up/down actions)
kubectl describe rs frontend-rs
```

### Advanced Debugging Flags

|**Flag**|**Purpose**|**Example**|
|---|---|---|
|**`-L`**|**Columns.** Adds specific labels as new columns in the output for quick comparison.|`kubectl get pods -L env,tier`|
|**`-l`**|**Filter.** Shows _only_ resources matching the label query.|`kubectl get pods -l tier=frontend`|
|**`--show-labels`**|**Verbose.** Dumps all labels in a single column. Good for debugging selector mismatches.|`kubectl get pods --show-labels`|

### Scaling

Bash

```
# Imperative scaling
kubectl scale rs frontend-rs --replicas=5
```