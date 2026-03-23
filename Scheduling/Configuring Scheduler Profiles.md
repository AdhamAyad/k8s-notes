# Kubernetes Scheduling: Framework & Profiles

**Tags:** #kubernetes #scheduling #framework #profiles #cka

**Status:** Core Concept

## 1. Core Concept: The Scheduling Context

The Kubernetes Scheduling Framework is a pluggable architecture that compiles directly into the core scheduler. It breaks down the process of placing a Pod onto a Node into well-defined phases.

Every attempt to schedule a Pod is split into two distinct cycles, which together form the **Scheduling Context**:

1. **Scheduling Cycle:** The process of selecting a feasible Node for the Pod. This cycle runs **serially** (one Pod at a time) to prevent race conditions during node selection.
    
2. **Binding Cycle:** The process of actually applying the scheduling decision to the cluster (updating the API). This cycle can run **concurrently** for multiple Pods.
    

> [!IMPORTANT] Cycle Abortion
> 
> If a Pod is determined to be unschedulable or if an internal error occurs at any point during the Scheduling or Binding cycles, the cycle is aborted. The Pod is returned to the queue to be retried later.

## 2. The Scheduling Framework Extension Points

The framework exposes specific "Extension Points" where plugins can hook into the process. The scheduler executes these points in a strict chronological order.

![Kubernetes Scheduling Framework Extensions](https://kubernetes.io/images/docs/scheduling-framework-extensions.png)

### A. Queueing Phase

Before a Pod is even evaluated for a Node, it must be queued.

- **PreEnqueue:** Called before adding Pods to the active queue. If a plugin returns anything other than `Success`, the Pod goes to the unschedulable list without triggering an `Unschedulable` condition.
    
- **EnqueueExtension & QueueingHint:** Evaluates cluster events to decide if a rejected Pod should be moved back to the active queue to retry scheduling.
    
- **QueueSort:** Determines the order of Pods in the queue (e.g., sorting by PriorityClass).
    

> [!WARNING] QueueSort Limitation
> 
> Unlike other extension points where multiple plugins can run sequentially, **only ONE** QueueSort plugin may be enabled at a time. The default is the Priority Sort plugin.

### B. Filtering Phase

This phase eliminates Nodes that absolutely cannot run the Pod.

- **PreFilter:** Pre-processes Pod information or checks cluster-level conditions. If it returns an error, the cycle is aborted.
    
- **Filter:** Evaluates each Node individually. If a Node fails this check (e.g., insufficient CPU, NodeName mismatch, or Taints), it is marked infeasible, and remaining filter plugins are skipped for that Node. _Nodes may be evaluated concurrently here._
    
- **PostFilter:** Triggered **only** if no feasible Nodes were found during the Filter phase. Its primary use case is executing **Preemption** (evicting lower-priority Pods to make room).
    

### C. Scoring Phase

This phase ranks the surviving feasible Nodes to find the absolute best fit.

- **PreScore:** Generates shared state/data for the Score plugins to use.
    
- **Score:** Assigns a raw integer score to each Node. For example, the `ImageLocality` plugin scores Nodes higher if they already have the container image downloaded.
    
- **NormalizeScore:** Adjusts and modifies the raw scores to fit within a standardized minimum and maximum range before the final ranking is calculated.
    

### D. Binding Preparation Phase

This phase prepares the chosen Node and Pod for the final physical binding.

- **Reserve:** Reserves resources on the target Node to prevent race conditions while the scheduler waits for the bind to succeed. It has two methods: `Reserve` and `Unreserve`.
    
- **Permit:** Can either `approve`, `deny`, or `wait`. If it returns `wait`, the Pod is placed in a waiting list with a timeout, halting its binding cycle until an external event approves it.
    

> [!IMPORTANT] The Unreserve Idempotency Rule
> 
> If the Reserve phase or any later phase fails, the `Unreserve` method is triggered to clean up allocated resources. The implementation of the `Unreserve` method **must be idempotent and may not fail**.

### E. Binding Phase

- **PreBind:** Executes prerequisite work, such as provisioning a network volume and mounting it to the Node before the Pod runs.
    
- **Bind:** The actual assignment of the Pod to the Node. The first Bind plugin that chooses to handle the Pod executes the bind, and the remaining Bind plugins are skipped.
    
- **PostBind:** An informational phase used exclusively to clean up associated resources after a successful bind.
    

## 3. Standard Scheduler Plugins

The default Kubernetes scheduler utilizes this exact framework using built-in plugins:

- **Priority Sort:** Hooks into `QueueSort`.
    
- **Node Resources Fit:** Hooks into `Filter` (checks CPU/Memory).
    
- **Node Name:** Hooks into `Filter` (checks if `nodeName` is specified).
    
- **Node Unschedulable:** Hooks into `Filter` (respects Cordon/Drain commands).
    
- **Default Binder:** Hooks into `Bind`.
    

## 4. Configuring Scheduler Profiles (Single Binary Strategy)

Historically, if you wanted different scheduling logic, you had to run entirely separate scheduler binaries (Multiple Schedulers). This caused significant operational overhead and potential race conditions.

Kubernetes introduced **Scheduler Profiles**, allowing you to run multiple independent scheduling configurations within a **single** `kube-scheduler` binary.

> [!NOTE] Architectural Benefit of Profiles
> 
> Running multiple profiles inside one binary completely eliminates race conditions because the single scheduler process maintains a unified view of the cluster state and resource allocations, while still offering different scheduling algorithms based on the profile name.

### YAML Implementation: Multiple Profiles

You configure profiles by defining them inside the `KubeSchedulerConfiguration` file. You can disable default plugins and enable custom ones for specific extension points.

YAML

```
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  # Profile 1: The Default Scheduler
  - schedulerName: default-scheduler

  # Profile 2: A custom profile that disables Taint checks
  - schedulerName: no-taints-scheduler
    plugins:
      filter:
        disabled:
          - name: TaintToleration
      score:
        disabled:
          - name: TaintToleration
        enabled:
          - name: MyCustomPluginA

  # Profile 3: A custom profile that completely disables scoring
  - schedulerName: fast-scheduler
    plugins:
      preScore:
        disabled:
          - name: '*' # The wildcard disables ALL built-in preScore plugins
      score:
        disabled:
          - name: '*' # The wildcard disables ALL built-in score plugins
```

### Assigning a Profile to a Pod

Just like traditional Multiple Schedulers, you assign a Pod to a specific profile using the `schedulerName` field in the Pod Spec:

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: fast-scheduled-pod
spec:
  # This points to the profile defined in the single scheduler binary
  schedulerName: fast-scheduler
  containers:
    - name: nginx
      image: nginx
```