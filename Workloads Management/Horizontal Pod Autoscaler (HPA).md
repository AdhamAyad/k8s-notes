# Kubernetes Architecture: Horizontal Pod Autoscaler (HPA)

**Tags:** #kubernetes #hpa #autoscaling #architecture #cka
**Status:** Advanced Implementation & Behavior Guide

---

## 1. Core Concept: The HPA Controller

The HorizontalPodAutoscaler (HPA) automatically updates a workload resource (like a Deployment or StatefulSet) to scale the number of Pods out or in based on observed metrics (e.g., CPU, Memory, or Custom Metrics). 

Kubernetes implements HPA as a control loop that runs intermittently (default is every 15 seconds). Once attached to a Deployment, the **HPA becomes the ultimate source of truth** for the replica count, taking control away from the static Deployment manifest.



---

## 2. The Replica Conflict & GitOps Best Practice

A critical architectural challenge occurs when the HPA dynamically changes the replica count, but your infrastructure-as-code (YAML) still contains a static `replicas` value.

### The Conflict Scenario (Thrashing/Flapping)
If HPA scales your application to 5 Pods, and you run `kubectl apply -f deployment.yaml` to update an image version while the file still contains `replicas: 1`:
1. **The Strike:** Kubernetes executes your explicit command and instantly terminates 4 Pods.
2. **The Counter-Strike:** Seconds later, the HPA detects high load on the single remaining Pod and franticly spins up 4 new Pods.
This causes temporary service degradation and resource thrashing.

### The 3-Way Merge Patch Solution
To fix this, you must **completely remove the `replicas` key** from your Deployment manifest. 
* **Initial Creation:** If omitted, Kubernetes assumes a default of `1` and creates the first Pod. HPA takes over from there.
* **Future Updates:** When you run `kubectl apply`, Kubernetes compares the new file to the live state. Because the `replicas` field is missing from your file, Kubernetes assumes you do not want to alter it. It leaves the current replica count (managed by HPA) untouched while applying your other changes (like the image update).

> [!TIP] Best Practice for Migrations
> If you already have a running Deployment with `replicas` set, use `kubectl apply edit-last-applied deployment/<name>` to remove the `spec.replicas` field from the system's memory before removing it from your source code. This prevents a one-time scale-down to 1 during the transition.

---

## 3. Metrics, Requests, and Limits

> [!IMPORTANT] The Golden Rule of HPA Scaling
> * **Percentage (`averageUtilization`):** STRICTLY DEPENDS on `requests`. The HPA calculates the target based on the Pod's `requests` value. If you forget to define `requests` in your Deployment, the HPA will fail and show an `Unknown` state.
> * **Absolute Value (`averageValue`):** COMPLETELY IGNORES `requests` and `limits`. The HPA only looks at the raw, actual consumption (e.g., if the Pod uses 50m, it triggers based on that raw number alone).

The HPA relies heavily on the `resources` block defined in your Pod specification, but it treats `requests` and `limits` very differently.

> [!WARNING] The Limits Misconception
> The HPA **completely ignores** resource `limits`. Limits are enforced exclusively by the `kubelet` on the Node to throttle containers. If you set a CPU limit lower than the HPA target, the pod will be throttled before it ever reaches the HPA target, meaning the HPA will never scale up.

### `averageValue` vs `averageUtilization`

* **`type: AverageValue` (Raw Metric):** If you set `averageValue: 50m`, the HPA ignores the Pod's CPU `requests`. It simply measures the raw CPU usage. If usage exceeds 50m, it scales up.
* **`type: Utilization` (Percentage):** If you set `averageUtilization: 50`, the HPA **requires** the Pod to have CPU `requests` defined. It calculates the target as a percentage of the request. (e.g., 50% of a 250m request = 125m target). If the request is missing, HPA fails with an `Unknown` state.

---

## 4. The Scaling Algorithm & Scenarios

The HPA calculates the required number of replicas using this formula:

$$\text{Desired Replicas} = \lceil \text{Current Replicas} \times \left( \frac{\text{Current Metric Value}}{\text{Desired Metric Value}} \right) \rceil$$



### Practical Scenarios (Assuming Target = 50m, MaxReplicas = 5)

* **Scenario A: Normal Traffic (No Action)**
  * Current Usage: 20m.
  * Calculation: $1 \times (20 / 50) = 0.4 \rightarrow 1$ Pod.
* **Scenario B: Sudden Spike (Scale Up)**
  * Current Usage: 200m.
  * Calculation: $1 \times (200 / 50) = 4.0 \rightarrow 4$ Pods.
  * Result: HPA creates 3 new Pods. The 200m load distributes across 4 Pods (50m each).
* **Scenario C: Massive Tsunami (Max Cap Reached)**
  * Current Usage: 600m (across 4 Pods = 150m each).
  * Calculation: $4 \times (150 / 50) = 12 \rightarrow 12$ Pods.
  * Result: The math demands 12 Pods, but `maxReplicas: 5` enforces a hard stop. HPA creates only 1 more Pod. The 5 Pods will share the 600m load (120m each) and remain overloaded until traffic subsides.

---

## 5. Advanced Configurations (Configurable Behavior)

Starting in `autoscaling/v2`, you can control how aggressively the HPA scales up or down using the `behavior` block.



> [!INFO] Stabilization Window (Anti-Flapping)
> To prevent "flapping" (rapidly creating and destroying pods due to fluctuating metrics), HPA uses a `stabilizationWindowSeconds`. 
> * Default scale-down window is **300 seconds (5 minutes)**. HPA will look back over the last 5 minutes and choose the highest recommended scale to ensure the traffic drop is permanent before killing Pods.

> [!NOTE] Implicit Maintenance Mode
> If you manually scale your Deployment to `0` replicas, the HPA will automatically pause its scaling activities (ScalingActive = false) until you manually scale the Deployment back above `0` or change the HPA's `minReplicas` to `0`.

---

## 6. Complete YAML Implementation

Here is the optimized setup combining the Deployment (without static replicas) and the v2 HPA manifest.

### 1. The Deployment (Optimized for HPA)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  # 'replicas' is intentionally omitted to allow HPA control
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        resources: 
          requests:
            cpu: "250m"
            memory: "250Mi"
          limits:
            cpu: "500m"
            memory: "500Mi"
````

### 2. The HorizontalPodAutoscaler (v2)

YAML

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        # Using Utilization depends on the 250m 'requests' in the Deployment
        type: Utilization
        averageUtilization: 50
  
  # Optional: Advanced behavior configuration
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

---
# 7. Exam Tip: Imperative Command (Fast YAML Generation)

In the CKA exam, do not write the HPA YAML from scratch. Use the imperative `autoscale` command to generate the base manifest quickly.

```bash
# Generate the HPA YAML file targeting 50% CPU utilization
kubectl autoscale deployment test-hpa --cpu-percent=50 --min=1 --max=5 --dry-run=client -o yaml > HPA.yaml
````

> [!WARNING] Absolute Values (averageValue)
> 
> The `autoscale` command natively flags `--cpu-percent` (which translates to `averageUtilization`). If your task specifically requires an absolute value like `50m` (`averageValue`), generate the file using the command above, then `vi HPA.yaml` and manually change `averageUtilization: 50` to `averageValue: 50m`.

---

## Strategic Use Cases: When to use HPA

HPA is designed for **Scaling OUT** (adding more identical Pods to distribute traffic). It assumes your application architecture can handle load balancing across multiple independent instances.

**Target Workloads: Stateless Applications**
* **Web Servers & APIs (Nginx, Node.js, Spring Boot):** Easily handle spikes in HTTP traffic by adding more web servers behind a load balancer.
* **Background Job Processors (Celery, Sidekiq):** When the message queue gets long, HPA spins up more workers to process messages in parallel.
* **Microservices:** Perfect for decoupling services where one specific component experiences high load and needs to scale independently.

> [!WARNING] The Ultimate Anti-Pattern (HPA + VPA)
> **NEVER** configure HPA and VPA to scale based on the exact same metric (e.g., both targeting CPU) for the same Deployment. 
> * HPA will see high CPU and create more Pods.
> * VPA will see high CPU and kill those exact same Pods to resize them.
> * **Result:** A catastrophic race condition causing continuous restarts and system instability.
> * *(Exception: You can mix them if they target entirely different metrics, like HPA for HTTP requests and VPA for Memory).*

> [!summary] Kubernetes HPA Metric Types Quick Reference
> * **Resource:** Tracks standard CPU or Memory utilization for the entire Pod.
> * **ContainerResource:** Tracks standard CPU or Memory utilization for a specific Container inside the Pod.
> * **Pods:** Tracks custom metrics emitted by the application itself, averaged across all Pods (e.g., HTTP requests per second per pod).
> * **Object:** Tracks metrics describing a different Kubernetes object in the cluster, not the pods (e.g., total traffic hitting a specific Ingress).
> * **External:** Tracks metrics from services completely outside the Kubernetes cluster (e.g., messages pending in an AWS SQS or Google Pub/Sub queue).
> 
> 

