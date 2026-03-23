# Kubernetes Scheduling: Taints and Tolerations

**Tags:** #kubernetes #scheduling #taints #tolerations #cka #nodes

**Status:** Core Concept

## 1. Core Concepts

- **Node Affinity** is an "attraction" property; it draws Pods to specific nodes.
    
- **Taints** are a "repulsion" property. They are applied to Nodes to repel Pods. A tainted node will not accept any Pod unless that Pod explicitly tolerates the taint.
    
- **Tolerations** are applied to Pods. They act as a "VIP pass," allowing the scheduler to place the Pod on a node with a matching taint.
    

> [!IMPORTANT]
> 
> Tolerations _allow_ scheduling onto tainted nodes, but they do _not guarantee_ it. The scheduler still evaluates other parameters (like resource availability) before making a final decision.

## 2. Managing Taints via CLI

**Adding a Taint:**

You use the `kubectl taint` command. A taint consists of a Key, a Value, and an Effect.

Bash

```
kubectl taint nodes node1 key1=value1:NoSchedule
```

**Removing a Taint:**

To remove the exact taint added above, append a minus sign (`-`) to the end of the command:

Bash

```
kubectl taint nodes node1 key1=value1:NoSchedule-
```

## 3. Taint Effects

The `effect` dictates what happens to Pods that do _not_ have a matching toleration. There are three allowed values:

1. **`NoSchedule` (Strict Repulsion)**
    
    No new Pods will be scheduled on the tainted node. However, Pods that are _already running_ on the node before the taint was applied are left alone and are not evicted.
    
2. **`PreferNoSchedule` (Soft Repulsion)**
    
    The control plane will _try_ to avoid placing untolerated Pods on the node, but it is not a hard guarantee. If the cluster is full, the scheduler may still place Pods here.
    
3. **`NoExecute` (Eviction / Immediate Repulsion)**
    
    This affects both new and _currently running_ Pods:
    
    - Pods without a matching toleration are evicted immediately.
        
    - Pods with a matching toleration that lack `tolerationSeconds` remain bound forever.
        
    - Pods with a matching toleration that specify `tolerationSeconds` remain bound only for that specific duration before being evicted.
        

> [!IMPORTANT] Architectural Concept: Why Taints Do Not Affect Core Cluster Components
> You might wonder why placing a strict `NoExecute` or `NoSchedule` taint on a node does not break or evict essential cluster components. This happens for two primary architectural reasons:
> 
> **1. Static Pods Bypass the Scheduler**
> Core Control Plane components (such as `kube-apiserver`, `etcd`, `kube-scheduler`, and `kube-controller-manager`) run as **Static Pods**. They are managed directly by the local `kubelet` which reads their configuration files from a specific directory on the server (usually `/etc/kubernetes/manifests`). Because they completely bypass the `kube-scheduler`, taints have absolutely zero effect on them.
> 
> **2. Built-in Wildcard Tolerations (The "Master Key")**
> Critical cluster add-ons that are deployed as `DaemonSets` (such as `kube-proxy` or CNI network plugins like Calico/Flannel) must run on every node to keep the cluster functional. To guarantee this, their manifests are written with built-in, highly permissive tolerations. 
> 
> For example, many network plugins use a universal "Wildcard" toleration. By omitting the `key`, `value`, and `effect`, and only specifying the `Exists` operator, the Pod acts as a master key, bypassing *any* taint on *any* node:
> ```yaml
> tolerations:
> - operator: "Exists"
> ```
> 
> Similarly, standard cluster components meant to run on the master node automatically include explicit built-in tolerations to bypass the default control-plane taint:
> ```yaml
> tolerations:
> - key: "node-role.kubernetes.io/control-plane"
>   operator: "Exists"
>   effect: "NoSchedule"
> ```
## 4. Toleration Matching Logic (Operators)

A toleration in a PodSpec matches a taint if the keys and effects are the same, and the operator criteria are met. The default operator is `Equal`.

### Method A: `Equal` (Exact Match)

The `value` in the Pod must exactly match the `value` on the Node.

- **Real-World Scenario:** A node is dedicated to the HR department database (`department=hr:NoSchedule`). Only Pods explicitly stating they belong to the HR department are allowed in.
    

YAML

```
# Pod Toleration using Equal
tolerations:
- key: "department"
  operator: "Equal"
  value: "hr"
  effect: "NoSchedule"
```

### Method B: `Exists` (The Wildcard / VIP Pass)

You do not specify a `value` at all. The presence of the `key` is enough to grant access.

- **Real-World Scenario:** A Monitoring Pod needs to run on all department servers (HR, IT, Finance). Instead of listing every department value, it uses `Exists` to say: "I tolerate any taint with the key 'department', regardless of its value."
    

YAML

```
# Pod Toleration using Exists
tolerations:
- key: "department"
  operator: "Exists"
  effect: "NoSchedule"
```

> [!NOTE] Special Cases
> 
> - If the `key` is completely empty, the operator must be `Exists`. This matches _all_ keys, _all_ values, and _all_ effects (A universal master key).
>     
> - If the `effect` is empty, it matches all effects for the specified key.
>     

## 5. Multiple Taints and Filtering

You can apply multiple taints to one node, and multiple tolerations to one Pod. Kubernetes processes this like a filter:

1. It looks at all taints on the node.
    
2. It ignores any taints for which the Pod has a matching toleration.
    
3. It applies the effects of the _remaining, un-ignored taints_.
    

- If at least one un-ignored taint is `NoSchedule`, the Pod is not scheduled.
    
- If there is no `NoSchedule` taint, but an un-ignored `PreferNoSchedule` exists, the scheduler tries not to schedule it.
    
- If at least one un-ignored taint is `NoExecute`, the Pod is evicted (if already running) or rejected (if new).
    

> [!WARNING] Scenario: Multiple Taints vs. Single Toleration
> **Can a Pod with only ONE toleration be scheduled on a Node with MULTIPLE taints?**
> 
> **Generally, NO.** Kubernetes acts as a strict filter:
> 1. It compares the Pod's single toleration against all taints on the Node.
> 2. It "crosses out" or ignores the single matched taint.
> 3. It strictly applies the effects of **all remaining unmatched taints**.
> 
> If even **one** unmatched taint is `NoSchedule` or `NoExecute`, the Pod is completely rejected or evicted. To enter, the Pod must carry a specific toleration for *every single strict taint* present on that Node.

## 6. The `nodeName` Edge Case

If you manually specify `.spec.nodeName` in a Pod manifest, **it bypasses the scheduler completely**.

- The Pod will be bound to that node even if the node has a `NoSchedule` taint.
    
- However, if the node has a `NoExecute` taint, the kubelet on the node will immediately eject the Pod unless the Pod has the appropriate `NoExecute` toleration.
    

> [!warning] nodeName vs. Taint Effects
> Specifying a **nodeName** allows a Pod to completely bypass the Kubernetes Scheduler. This means it ignores **NoSchedule** taints existing on that specific node.
>
> However, **NoExecute** taints are enforced *after* scheduling. Therefore, even if a Pod is forced onto a node via **nodeName**, it will be immediately evicted if a matching **NoExecute** taint is present.

## 7. Numeric Comparison Operators (Alpha Feature - v1.35)

In addition to `Equal` and `Exists`, Kubernetes v1.35 introduces `Gt` (Greater than) and `Lt` (Less than) for integer values. Both values must be valid signed 64-bit integers with no leading zeros.

- **Scenario:** Threshold-based scheduling based on Service Level Agreements (SLA).
    
- A node is tainted with an SLA score: `servicelevel.../agreed-service-level=950:NoSchedule`
    

A Pod can use `Gt` to tolerate nodes with an SLA greater than 900:

YAML

```
tolerations:
- key: "servicelevel.organization.example/agreed-service-level"
  operator: "Gt"
  value: "900"
  effect: "NoSchedule"
```

_(Because 950 > 900, the toleration matches)._

## 8. Common Use Cases

1. **Dedicated Nodes:** Taint a set of nodes (`dedicated=groupName:NoSchedule`). Give corresponding tolerations to specific users' Pods. Add labels and Node Affinity to ensure those Pods _only_ go to those dedicated nodes.
    
2. **Nodes with Special Hardware:** Taint nodes containing GPUs (`special=true:NoSchedule`). Only Pods requiring GPUs get the toleration, keeping standard web Pods from wasting expensive GPU resources.
    

## 9. Taint-Based Evictions

The Node Controller automatically taints a node when certain conditions occur, causing Pod eviction based on the `NoExecute` effect.

**Built-in automatic taints include:**

- `node.kubernetes.io/not-ready`: Node is not ready (Ready condition is False).
    
- `node.kubernetes.io/unreachable`: Node is unreachable from the controller (Ready condition is Unknown).
    
- `node.kubernetes.io/memory-pressure`
    
- `node.kubernetes.io/disk-pressure`
    
- `node.kubernetes.io/pid-pressure`
    
- `node.kubernetes.io/network-unavailable`
    
- `node.kubernetes.io/unschedulable`
    

### The `tolerationSeconds` Grace Period

Kubernetes automatically adds a default toleration for `not-ready` and `unreachable` with `tolerationSeconds=300` to all standard Pods. This means if a node loses network connection, Kubernetes waits 5 minutes before evicting the Pods, hoping the network recovers.

You can override this for critical Pods that hold local state by defining a longer grace period:

YAML

```
tolerations:
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 6000
```

### DaemonSet Exceptions

`DaemonSet` Pods are automatically created with `NoExecute` tolerations for `unreachable` and `not-ready` with **no** `tolerationSeconds`. This ensures DaemonSet Pods are never evicted due to node network/readiness failures. They also automatically tolerate memory, disk, and PID pressure to ensure backward compatibility.

> [!INFO] Architectural Note (v1.29+)
> 
> The taint-based eviction implementation was moved out of the node controller into a dedicated, independent component called the `taint-eviction-controller`.

## 10. Device Taints

When using Dynamic Resource Allocation (DRA), administrators can taint individual hardware devices instead of entire nodes. This allows targeting faulty specific hardware without rendering the entire worker node unschedulable. Tolerations for these devices are specified when requesting the hardware.