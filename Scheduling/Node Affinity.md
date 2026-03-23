# Kubernetes Scheduling: Node Affinity

**Tags:** #kubernetes #scheduling #affinity #node-affinity #pod-affinity #cka

**Status:** Core Concept

---
## 1. Overview and Core Concepts

Node affinity is conceptually similar to `nodeSelector`, allowing you to constrain which nodes your Pod can be scheduled on based on node labels. However, it provides a much more expressive syntax and advanced logical operations.

There are two primary types of node affinity:

1. **`requiredDuringSchedulingIgnoredDuringExecution`:** The scheduler cannot schedule the Pod unless the rule is met. This functions as a strict, hard requirement (similar to `nodeSelector`).
    
2. **`preferredDuringSchedulingIgnoredDuringExecution`:** The scheduler tries to find a node that meets the rule. If a matching node is not available, the scheduler will still schedule the Pod on another available node. This functions as a soft preference.
    

> [!INFO] What does "IgnoredDuringExecution" mean?
> 
> In both types, the suffix `IgnoredDuringExecution` means that if the node labels change _after_ Kubernetes schedules the Pod, the Pod continues to run. The scheduler does not evict it.

## 2. Defining Node Affinity Rules

You specify node affinities using the `.spec.affinity.nodeAffinity` field in your Pod spec.

### Example: Basic Node Affinity

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:3.8
```

**In this example:**

- The node **must** have a label with the key `topology.kubernetes.io/zone` and the value **must** be either `antarctica-east1` or `antarctica-west1`.
    
- The node **preferably** has a label with the key `another-node-label-key` and the value `another-node-label-value`.
    

### Operators

You use the `operator` field to specify a logical operator for Kubernetes to interpret the rules. Allowed operators are:

- `In`
    
- `NotIn`
    
- `Exists`
    
- `DoesNotExist`
    
- `Gt` (Greater than)
    
- `Lt` (Less than)
    

> [!NOTE] Anti-Affinity via Operators
> 
> `NotIn` and `DoesNotExist` allow you to define node _anti-affinity_ behavior (repelling Pods). Alternatively, you can use node taints to repel Pods from specific nodes.

### The Logical Rules (AND vs OR)

The way Kubernetes parses multiple rules is strictly defined:

1. **`nodeSelector` + `nodeAffinity`:** If you specify both, **BOTH** must be satisfied for the Pod to be scheduled.
    
2. **Multiple `nodeSelectorTerms` (OR Logic):** If you specify multiple terms under `nodeAffinity` types, the Pod can be scheduled if **ONE** of the specified terms can be satisfied.
    
3. **Multiple `matchExpressions` (AND Logic):** If you specify multiple expressions inside a single `matchExpressions` block associated with a `nodeSelectorTerm`, the Pod can be scheduled only if **ALL** the expressions are satisfied.
    

## 3. Node Affinity Weight

You can specify a `weight` between 1 and 100 for each instance of the `preferredDuringSchedulingIgnoredDuringExecution` affinity type.

**How Scoring Works:**

When the scheduler finds nodes that meet all the hard scheduling requirements, it iterates through every preferred rule that the node satisfies. It adds the `weight` of that expression to a sum. This final sum is added to the score of other priority functions for the node. The node with the highest total score wins.

### Example: Weighted Preferences

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: with-affinity-preferred-weight
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: label-1
            operator: In
            values:
            - key-1
      - weight: 50
        preference:
          matchExpressions:
          - key: label-2
            operator: In
            values:
            - key-2
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:3.8
```

- If Node A matches `label-1:key-1` (Score: 1) and Node B matches `label-2:key-2` (Score: 50), the scheduler will schedule the Pod onto Node B due to the higher weight.
    
- _Prerequisite:_ Both nodes must already have the `kubernetes.io/os=linux` label to even be considered.
    

## 4. Node Affinity per Scheduling Profile (Beta Feature)

When configuring multiple scheduling profiles, you can associate a profile with a node affinity. This is useful if a specific scheduler should only apply to a specific set of nodes.

You configure this by adding an `addedAffinity` to the `args` field of the `NodeAffinity` plugin in the scheduler configuration.

YAML

```
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
  - schedulerName: foo-scheduler
    pluginConfig:
      - name: NodeAffinity
        args:
          addedAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: scheduler-profile
                  operator: In
                  values:
                  - foo
```

- The `addedAffinity` is applied to all Pods that set `.spec.schedulerName: foo-scheduler` in addition to whatever is written in the Pod's `.spec.NodeAffinity`.
    
- **Best Practice:** Because `addedAffinity` is not visible to end-users, it can cause unexpected behavior. Always use node labels that have a clear correlation to the scheduler profile name.
    

> [!WARNING] DaemonSet Restriction
> 
> The DaemonSet controller does not support scheduling profiles. When creating Pods, it forces the use of the default Kubernetes scheduler while honoring any `nodeAffinity` rules defined in the DaemonSet manifest.

---

## 5. Inter-Pod Affinity and Anti-Affinity

Inter-pod affinity and anti-affinity allow you to constrain which nodes your Pods can be scheduled on based on the **labels of Pods already running on that node**, instead of the node labels.

### The Topology Domain Structure

The rules take the form: _"This Pod should (or should not) run in an **X** if that **X** is already running one or more Pods that meet rule **Y**."_

- **Y (The Rule):** Expressed as label selectors with an optional associated list of namespaces (since Pod labels are namespaced).
    
- **X (The Topology Domain):** Expressed using a `topologyKey`. This is the key for the node label that denotes the domain (e.g., node, rack, cloud provider zone, or region).
    

### Critical Warnings

> [!WARNING] Performance Impact
> 
> Inter-pod affinity and anti-affinity require substantial amounts of processing which can slow down scheduling in large clusters significantly. **It is not recommended to use them in clusters larger than several hundred nodes.**

> [!WARNING] Labeling Consistency
> 
> Pod anti-affinity requires nodes to be consistently labeled. Every node in the cluster must have an appropriate label matching the `topologyKey`. Missing labels lead to unintended behavior.

### Configuration Fields

Like node affinity, Inter-pod affinity uses two types:

- `requiredDuringSchedulingIgnoredDuringExecution` (Hard constraints)
    
- `preferredDuringSchedulingIgnoredDuringExecution` (Soft constraints)
    

These are placed under `.spec.affinity.podAffinity` or `.spec.affinity.podAntiAffinity`.

**Use Case Examples:**

- **Affinity:** Co-locate Pods of two different services in the same cloud provider zone because they communicate heavily.
    
- **Anti-Affinity:** Spread Pods from a single service across multiple cloud provider zones for high availability.
    

### 6. Inter-Pod Scheduling Behavior

When scheduling a new Pod, the scheduler evaluates the Pod's affinity/anti-affinity rules against the current cluster state through strict operational phases:

1. **Hard Constraints (Node Filtering):**
    
    The scheduler ensures the new Pod is assigned to nodes that satisfy the `requiredDuringSchedulingIgnoredDuringExecution` affinity and anti-affinity rules based on the existing Pods.
    
2. **Soft Constraints (Scoring):**
    
    The scheduler scores nodes based on how well they meet the `preferredDuringSchedulingIgnoredDuringExecution` affinity and anti-affinity rules to optimize Pod placement.
    
3. **Ignored Fields (Performance Optimization):**
    
    The scheduler explicitly **ignores** the `preferredDuringSchedulingIgnoredDuringExecution` affinity and anti-affinity rules of the _existing_ Pods already on the cluster when making the scheduling decision for the _new_ Pod.