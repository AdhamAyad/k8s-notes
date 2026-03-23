# Kubernetes Autoscaling: Vertical Pod Autoscaler (VPA)

**Tags:** #kubernetes #vpa #autoscaling #architecture #cka
**Status:** Advanced Implementation & Behavior Guide

---

## 1. Core Concept: What is VPA?

While the Horizontal Pod Autoscaler (HPA) scales by adding *more* Pods (scaling out), the **Vertical Pod Autoscaler (VPA)** scales by giving *existing* Pods more or less CPU and Memory (scaling up/down). This process is known as **rightsizing**.

VPA analyzes the actual, historical resource consumption of your containers and automatically adjusts their `requests` and `limits` to match reality, preventing both resource starvation (throttling/OOMKills) and resource waste (over-provisioning).



> [!WARNING] The Golden Rule: VPA vs HPA
> **DO NOT** use VPA and HPA together on the same target metrics (e.g., both managing CPU). 
> If CPU spikes, HPA will try to create new Pods, while VPA will simultaneously try to kill existing Pods to increase their size. This creates a destructive loop (thrashing) that will severely degrade your application.

---

## 2. The Three VPA Components (How it works under the hood)

VPA is not a single controller; it operates as a trio of cooperating components:

1. **The Recommender (The Analyst):**
   * Monitors real-time and historical metrics (from the Metrics Server).
   * Ignores what you wrote in your YAML and calculates what the Pod *actually* needs.
   * Outputs 3 values: `Target` (optimal), `Lower Bound` (minimum viable), and `Upper Bound` (maximum safe).
2. **The Updater (The Executioner):**
   * Constantly compares running Pods against the Recommender's `Target`.
   * If a Pod is too small (or too large), the Updater **evicts (kills)** the Pod so it can be recreated.
3. **The Admission Controller (The Mutating Webhook):**
   * Intercepts the creation request of the *new* Pod.
   * Strips out the old `requests` and `limits` defined in the Deployment manifest.
   * Injects the new, optimized numbers calculated by the Recommender before the Pod is scheduled on a Node.

---

## 3. How VPA Handles Requests vs. Limits

VPA operates independently of the numbers you initially configure, but it respects your **design ratios**.

* **Requests:** VPA completely overwrites the original requests with its calculated `Target` recommendation.
* **Limits (Proportional Scaling):** VPA maintains the original ratio between your requests and limits. 
  * *Example:* If your original YAML had a Request of 100m and a Limit of 200m (a 1:2 ratio), and VPA decides the new Request should be 500m, it will automatically set the new Limit to 1000m to preserve that 1:2 ratio.

---

## 4. VPA Update Modes (Lifecycle Control)

Because modifying CPU/Memory for a running Pod historically required a restart, VPA offers different modes to control how aggressive it should be.

| Mode | Behavior | Use Case |
| :--- | :--- | :--- |
| **Off** | Recommender calculates the optimal sizes and stores them in the VPA object's `.status`, but takes no action. | **Dry-Run/Audit:** Great for observing a workload for a week to manually tune your YAML files later. |
| **Initial** | VPA only injects the new sizes when a Pod is first created (e.g., during a Deployment rollout). It will *never* kill a running Pod to update it. | **Safe Mode:** Good for workloads that cannot tolerate sudden restarts but need to be rightsized on the next deployment. |
| **Recreate** | The standard active mode. The Updater actively evicts running Pods if they deviate from the recommendation, forcing the Admission Controller to recreate them with new sizes. | **Active Autoscaling:** For robust, highly available stateless apps (Must use `PodDisruptionBudgets`). |
| **InPlaceOrRecreate** | Attempts to resize the Pod's resources dynamically *without* restarting it (using the modern In-Place resize feature). If the Node lacks capacity for an in-place resize, it falls back to evicting the Pod (like Recreate mode). | **The Future Standard:** Best for minimizing downtime, provided your cluster supports in-place updates. |

*(Note: `Auto` mode is deprecated and acts as an alias for `Recreate`).*

---

## 5. Resource Policies (Boundary Control)

You can put guardrails on the VPA to ensure it doesn't scale a Pod too small (causing it to fail) or too large (consuming the entire Node).

### Key Policy Configurations:
* **`minAllowed` & `maxAllowed`:** Hard boundaries. VPA will never recommend values outside this range, regardless of actual usage.
* **`controlledResources`:** Define if VPA should manage `cpu`, `memory`, or both.
* **`controlledValues`:**  `RequestsAndLimits` (Default): Manages both proportionally.
  * `RequestsOnly`: VPA optimizes the requests but leaves your hard limits untouched.

> [!INFO] LimitRanges Interaction
> If your namespace has a `LimitRange` configured with maximum container sizes, the VPA will respect it. If VPA's recommendation exceeds the LimitRange `max`, VPA will cap its recommendation at that limit and proportionally scale down the request to match.

---

## 6. Complete YAML Implementation

Here is an example of a VPA configured for safe, bounded eviction-based scaling.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  # 1. Target the workload
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: web-app
    
  # 2. Define how aggressive VPA should be
  updatePolicy:
    updateMode: "Recreate" # Actively evicts pods to resize them
    
  # 3. Set Guardrails (Resource Policies)
  resourcePolicy:
    containerPolicies:
    - containerName: "nginx" # Target a specific container in the Pod
      controlledResources:
        - cpu
        - memory
      controlledValues: RequestsAndLimits
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2 # 2 Cores maximum
        memory: 2Gi
````

---

## Strategic Use Cases: When to use VPA

VPA is designed for **Scaling UP** (increasing CPU/RAM of existing Pods). It assumes your application needs more sheer computing power within a single instance and cannot be easily split.

**Target Workloads: Stateful Apps, Databases, and ML**
* **Databases (PostgreSQL, MySQL):** Relational databases are notoriously hard to scale horizontally without complex sharding. VPA allows the DB to simply consume more RAM for caching and CPU for heavy queries as data grows.
* **Machine Learning (ML) & Big Data:** ML training jobs often require loading massive datasets entirely into memory. Splitting this across smaller pods doesn't work; they need one massive, powerful Pod to avoid Out Of Memory (OOM) crashes.
* **Legacy Monolithic Applications:** Older apps that keep active user sessions or state directly in memory and are not designed for distributed computing.

> [!WARNING] The Ultimate Anti-Pattern (VPA + HPA)
> **NEVER** configure VPA and HPA to scale based on the exact same metric (e.g., both targeting CPU) for the same Deployment. VPA will attempt to evict Pods to resize them precisely when HPA is trying to scale them out, leading to severe cluster thrashing and downtime.