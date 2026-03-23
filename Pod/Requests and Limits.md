# Kubernetes Resources: Requests, Limits, and Quotas

**Tags:** #kubernetes #resources #cpu #memory #limitrange #resourcequota #qos #cka
**Status:** Core Concept

## 1. Core Concepts: Requests vs. Limits

In Kubernetes, resource management is strictly defined at the **Container level**, not the Pod level. The Pod's total resource requirement is the sum of the resources of all its containers.

* **Requests:** The minimum amount of resources guaranteed to a container.
* *Scheduler Role:* The `kube-scheduler` only considers the `requests` value when placing a Pod. It sums the container requests and searches for a Node with enough unallocated capacity. If no Node has enough capacity (e.g., `FailedScheduling: Insufficient cpu`), the Pod remains in a `Pending` state.


* **Limits:** The absolute maximum amount of resources a container is allowed to consume.

### Resource Measurement Units

* **CPU:** Measured in cores or fractional cores. `1` equals 1 full vCPU. `0.1` CPU is equivalent to `100m` (millicores).
* **Memory:** Measured in bytes. Typically written using Mebibytes (`Mi`) or Gibibytes (`Gi`). E.g., `256Mi` or `1Gi`.

## 2. The Enforcement: Throttling vs. OOMKilled

Kubernetes handles violations of CPU and Memory limits differently because of the nature of these resources:

* **CPU (Compressible Resource):** If a container attempts to use more CPU than its defined `limit`, Kubernetes will **throttle** the container. The application will run slower, but the container will **not** be terminated.
* **Memory (Incompressible Resource):** If a container attempts to consume more memory than its defined `limit`, the Linux kernel's OOM (Out Of Memory) Killer is invoked. The container is **instantly terminated** and enters the `OOMKilled` state.

## 3. Configuration Scenarios and Default Behaviors

By default, Kubernetes does not enforce CPU or memory requests and limits. Understanding the resulting behavior of different configurations is heavily tested in the CKA exam.

1. **No Requests and No Limits (BestEffort):**
* The container has a "blank check." It can consume all available resources on the node, potentially causing the "Noisy Neighbor" problem and starving other pods.


2. **Limits Specified Without Requests:**
* Kubernetes automatically sets the `requests` value to be equal to the `limit` value.


3. **Requests Defined Without Limits (Burstable):**
* The container is guaranteed its requested amount. Because there is no limit, it can "burst" and utilize idle resources on the node.
* *Risk:* If the container bursts and fills the node's memory, and then other Pods need their guaranteed memory, the bursting container is at high risk of being `OOMKilled` to free up space.


4. **Both Requests and Limits Defined (Guaranteed / Burstable):**
* The container has a guaranteed base and a strict ceiling.



## 4. Pod Quality of Service (QoS) Classes

Based on how you configure requests and limits, Kubernetes assigns a QoS class to the Pod. This dictates the eviction priority when a Node runs out of resources:

1. **Guaranteed (VIP - Last to be evicted):** Every container in the Pod has both memory and CPU requests and limits, and `requests` strictly equals `limits`.
2. **Burstable (Middle Class):** At least one container has a memory or CPU request, but the requests do not equal the limits (e.g., Requests without Limits, or Limits higher than Requests).
3. **BestEffort (First to be evicted):** No containers in the Pod have any memory or CPU requests or limits.

### YAML Implementation Example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: heavy-app
    image: nginx
    resources:
      requests:
        memory: "1Gi"
        cpu: "1"       # Or 1000m
      limits:
        memory: "2Gi" # Container is OOMKilled if it exceeds this
        cpu: "2"       # Container is throttled if it exceeds this

```

## 5. LimitRange (Namespace-Level Container Defaults)

To prevent users from deploying "BestEffort" pods that consume all cluster resources, administrators use a `LimitRange`.
A `LimitRange` is a namespace-scoped object that automatically assigns default resource values to containers that do not specify them, and enforces minimum/maximum boundaries.

> [!WARNING] Crucial Exam Detail
> A LimitRange only applies to **new** Pods created *after* the LimitRange is applied. It does not modify or affect existing Pods currently running in the namespace.

### YAML Example: CPU and Memory LimitRange

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: compute-resource-constraint
  namespace: development
spec:
  limits:
    - type: Container
      default:           # The default LIMIT applied if user omits it
        cpu: 500m
        memory: 1Gi
      defaultRequest:    # The default REQUEST applied if user omits it
        cpu: 500m
        memory: 1Gi
      max:               # The absolute maximum a user is allowed to request/limit
        cpu: "1"
        memory: 2Gi
      min:               # The absolute minimum a user must request/limit
        cpu: 100m
        memory: 500Mi

```

## 6. ResourceQuota (Namespace-Level Aggregate Limits)

While a `LimitRange` restricts a *single Pod*, a `ResourceQuota` restricts the **aggregate total** of all resources consumed by all Pods combined within a Namespace.

* It can limit total compute resources (e.g., maximum 10 CPUs total for the entire namespace).
* It can limit object counts (e.g., maximum 5 ConfigMaps, 10 Pods, 2 LoadBalancers).
* **Cluster Independence:** ResourceQuotas are independent of the cluster's physical capacity. Adding nodes to the cluster does not automatically grant a namespace more quota.

### YAML Example: ResourceQuota

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-and-count-quota
  namespace: development
spec:
  hard:
    # Compute Limits
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "10"
    limits.memory: "10Gi"
    # Object Count Limits
    pods: "10"
    services.loadbalancers: "2"
    count/deployments.apps: "5"

```

### Quota Scopes

ResourceQuotas can be heavily filtered using `scopeSelector` to only track specific types of Pods. Common scopes include:

* `PriorityClass`: Only track resources consumed by Pods with a specific priority level.
* `BestEffort` / `NotBestEffort`: Track based on the QoS class.
* `Terminating` / `NotTerminating`: Track based on the `activeDeadlineSeconds` state.
* `CrossNamespacePodAffinity`: Used by administrators with a hard limit of `0` to prevent namespaces from utilizing cross-namespace anti-affinity rules, preventing localized Denial of Service (DoS) attacks on the scheduler.