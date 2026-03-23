# Kubernetes Scheduling: PriorityClass & Preemption

**Tags:** #kubernetes #scheduling #priorityclass #preemption #cka

**Status:** Core Concept

## 1. Core Concept: What is a PriorityClass?

In Kubernetes, when multiple Pods are waiting to be scheduled, or when a cluster is completely out of resources, the `kube-scheduler` needs a way to determine which Pod is the most important.

A **PriorityClass** is a **Cluster-scoped object** (it does not belong to any specific Namespace) that maps a name to an integer value.

- **The Rule:** The higher the integer value, the higher the priority of the Pod.
    
- **The Queue:** Pods with higher priority are placed at the front of the scheduling queue ahead of lower-priority Pods.
    

## 2. Value Ranges and System Defaults

The `value` field is the actual priority that pods receive.

- **User Applications:** The value can be any valid integer up to 1 billion (1,000,000,000).
    
- **System-Critical Pods:** Values above 1 billion are reserved for core Kubernetes components to ensure they are never evicted by user applications.
    
    - `system-cluster-critical`: 2,000,000,000
        
    - `system-node-critical`: 2,000,100,000
        

> [!INFO] Default Pod Priority
> 
> If a Pod is created without explicitly defining a `priorityClassName`, its default priority value is exactly `0`.

## 3. Preemption Policy (The Eviction Engine)

The true power of a PriorityClass lies in its `preemptionPolicy`. This dictates how the scheduler behaves when the cluster is 100% full and a high-priority Pod needs to be scheduled.

There are two possible enum values for this policy:

### A. `PreemptLowerPriority` (The Default "VIP" Scenario)

If the cluster is full, the scheduler will actively **evict (kill)** running Pods that have a lower priority value.

- **Workflow:** Lower-priority Pods are terminated to free up CPU/Memory -> The high-priority Pod is scheduled in their place -> The terminated Pods are sent back to the `Pending` queue.
    
- **Use Case:** Mission-critical applications (e.g., Payment gateways, Production Databases) that must run immediately, regardless of what else is disrupted.
    

### B. `Never` (The "Polite" Scenario)

This specifies that the Pod will **never preempt (evict)** other running Pods, even if they have a lower priority.

- **Workflow:** The high-priority Pod jumps to the absolute front of the `Pending` queue. It will not kill existing applications, but the moment any resources naturally free up in the cluster, this Pod will be the first to claim them.
    
- **Use Case:** Heavy Batch Jobs or Data Processing tasks. You want them to start as soon as possible, but you do not want to crash live customer-facing web servers to do it.
    

## 4. The `globalDefault` Property

> [!IMPORTANT] Global Default Behavior
> 
> `globalDefault` specifies whether this PriorityClass should be considered as the default priority for pods that do not have any priority class.
> 
> **By default, if omitted, this field is `false`.**
> 
> Only one PriorityClass can be marked as `globalDefault`. However, if more than one PriorityClasses exists with their `globalDefault` field set to true, the smallest value of such global default PriorityClasses will be used as the default priority.

## 5. YAML Implementation & Imperative Generation

### A. Generating via Imperative Command (CKA Exam Strategy)

During the CKA exam, the fastest way to create the foundational YAML for a PriorityClass without memorizing the API structure is using the imperative `kubectl create` command with the `--dry-run` flag.

**Command:**

Bash

```
kubectl create priorityclass test --value=1000000 --dry-run=client -o yaml > priority.yaml
```

**Generated Output:**

YAML

```
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  creationTimestamp: null
  name: test
value: 1000000
```

_Note: You can then use `vi priority.yaml` to add optional fields like `preemptionPolicy` or `description` if required by the exam question._

### B. Comprehensive YAML Implementation

Below is a complete manifest containing all available options for a PriorityClass, along with the effects of each choice.

YAML

```
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: comprehensive-priority-example
  # Note: Do not define a namespace here. PriorityClass is a cluster-scoped object.

# 1. value (Required)
# - Effect: Determines the actual priority. The higher the number, the more important the pod.
# - Range: Any integer up to 1,000,000,000 for standard applications.
value: 1000000

# 2. globalDefault (Optional)
# - Effect: If set to true, this value becomes the default for all new pods that lack a priorityClassName.
# - Options: true | false (Defaults to false if omitted).
globalDefault: false

# 3. description (Optional)
# - Effect: Provides an arbitrary string to guide other administrators on when to use this class.
description: "Use this exclusively for mission-critical database pods to ensure immediate scheduling."

# 4. preemptionPolicy (Optional)
# - Effect: Dictates the scheduler's behavior when the cluster is entirely out of resources.
# - Options:
#     * PreemptLowerPriority (Default): Actively kills lower priority pods to make room immediately.
#     * Never: Refuses to kill running pods; instead, it places the pod at the very front of the Pending queue to wait for the next available slot.
preemptionPolicy: PreemptLowerPriority
```

## 6. Assigning the PriorityClass to a Pod

To grant a Pod these privileges, you simply reference the name of the PriorityClass in the Pod's `spec`.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: critical-database-pod
  labels:
    tier: backend
spec:
  containers:
    - name: mysql-container
      image: mysql:8.0
  # Link the PriorityClass here
  priorityClassName: comprehensive-priority-example
```

> [!WARNING] Exam Verification Tip
> 
> To view all existing PriorityClasses, their exact integer values, and policies in the cluster, use the command:
> 
> `kubectl get priorityclass`