# Kubernetes Architecture: Multi-Container Pods

**Tags:** #kubernetes #architecture #pods #init-containers #sidecar #cka
**Status:** Core Reference Guide

---

## 1. Overview of Multi-Container Pods

In Kubernetes, a Pod is the smallest deployable unit. While typically used to run a single primary application container, a Pod can encapsulate multiple containers. These containers share the same network namespace (IP and localhost) and can share storage volumes. 

Based on their lifecycle and execution order, Kubernetes categorizes containers within a Pod into three distinct types:
1. Regular App Containers (Co-located)
2. Init Containers
3. Native Sidecar Containers



---

## 2. Init Containers

Init containers are specialized containers that run sequentially and must complete their execution before the main application containers are allowed to start.

### Core Concepts & Behavior
* **Sequential Execution:** If multiple init containers are defined, they run one by one. Container B will not start until Container A completes successfully.
* **Run to Completion:** They are designed to execute a specific task and exit (Exit Code 0). 
* **Blocking Nature:** The Pod remains in a `Pending` or `Init` state, and the main app containers will not start until all init containers succeed.
* **Probes:** Regular init containers do not support `readinessProbe`, `livenessProbe`, or `startupProbe`.

> [!NOTE] Use Cases for Init Containers
> * **Dependency Checking:** Waiting for a Service or Database to be ready before starting the app (e.g., using `nslookup` or `ping`).
> * **Data Preparation:** Cloning a Git repository into a shared volume for the main app to use.
> * **Configuration Generation:** Dynamically generating config files using templates and Pod IP details.

> [!IMPORTANT] Advantages & Pros
> * **Security:** Keeps unnecessary tools (like `sed`, `awk`, `curl`, `git`) out of the main application image, reducing the attack surface.
> * **Separation of Concerns:** Application developers and infrastructure engineers can build their images independently.
> * **Privilege Separation:** Init containers can be given elevated permissions or access to specific Secrets that the main app container does not need.

### YAML Example: Init Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  # This container runs first, finishes its task, and terminates.
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', "until nslookup mydb; do echo waiting for mydb; sleep 2; done"]
  
  containers:
  # This container only starts after 'init-mydb' exits successfully.
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
````

---

## 3. Native Sidecar Containers

A sidecar container runs alongside the main application to enhance or extend its functionality without altering the primary code. With the introduction of the Native Sidecar feature (stable in v1.29+), Kubernetes now handles them through a specific lifecycle mechanism.

### Core Concepts & Behavior

- **Definition:** They are defined within the `initContainers` array but are given a `restartPolicy: Always`.
    
- **Continuous Execution:** Unlike regular init containers, sidecars do not exit. They start before the main app and continue running in the background.
    
- **Probes:** Because they run continuously, native sidecars support `readinessProbe`, `livenessProbe`, and `startupProbe`.
    
- **Termination Logic:** Upon Pod termination, Kubernetes waits for the main app containers to gracefully stop first, and only then sends the termination signal to the sidecar containers.
    

> [!NOTE] Use Cases for Sidecar Containers
> 
> - **Logging:** Forwarding or shipping local log files to a centralized logging system (e.g., Fluentd, Promtail).
>     
> - **Monitoring/Metrics:** Adapting legacy metrics into Prometheus-readable formats.
>     
> - **Network Proxies:** Handling mTLS, routing, or database connection pooling (e.g., Istio Envoy proxy, PgBouncer).
>     

> [!TIP] Best Practice: Solving the Kubernetes Job Problem
> 
> Previously, sidecars were placed in the `containers` array. This caused a major issue with Kubernetes `Jobs`: when the main task finished, the Job would hang forever because the sidecar kept running. By using Native Sidecars (`restartPolicy: Always` inside `initContainers`), Kubernetes knows to automatically terminate the sidecar once the main job completes, allowing the Job to succeed gracefully.

### YAML Example: Native Sidecar Container

YAML

```
apiVersion: batch/v1
kind: Job
metadata:
  name: job-with-sidecar
spec:
  template:
    spec:
      restartPolicy: Never
      volumes:
        - name: shared-data
          emptyDir: {}
          
      initContainers:
        # Native Sidecar: Starts first, stays running to ship logs
        - name: logshipper
          image: alpine:latest
          restartPolicy: Always 
          command: ['sh', '-c', 'tail -F /opt/logs.txt']
          volumeMounts:
            - name: shared-data
              mountPath: /opt
              
      containers:
        # Main App: Writes logs, finishes, and exits. 
        # Once it exits, K8s will automatically kill the logshipper.
        - name: main-task
          image: alpine:latest
          command: ['sh', '-c', 'echo "Task complete" > /opt/logs.txt']
          volumeMounts:
            - name: shared-data
              mountPath: /opt
```

---

## 4. Resource Allocation & Limits

Understanding how Kubernetes calculates resource requests (CPU/Memory) for Multi-Container Pods is crucial for accurate scheduling.

> [!IMPORTANT] The Resource Calculation Rule
> 
> Scheduling is based on the **effective request/limit**. Kubernetes calculates the Pod's total resource needs by taking the HIGHER value between:
> 
> 1. The highest requested resource of any single `initContainer`.
>     
> 2. The sum of all requested resources from the regular `containers` (app + native sidecars).
>     
> 
> _Why?_ Because init containers run sequentially before the apps start, they can reserve large amounts of temporary resources for initialization without permanently starving the node for the lifetime of the Pod.

---

> [!INFO] Pod `restartPolicy` Lifecycle Control
> The `restartPolicy` dictates the kubelet's behavior when a container inside the Pod terminates. It is defined at the Pod `spec` level, applying to all containers within that Pod.

> [!important] The Three Policies
> * **`Always` (Default):** Restarts the container regardless of why it exited. Ideal for long-running services (Deployments, web servers).
> * **`OnFailure`:** Restarts the container ONLY if it exits with a non-zero status (error). Ideal for tasks that might fail temporarily but should eventually succeed.
> * **`Never`:** NEVER restarts the container. If it succeeds, it goes to `Completed`. If it fails, it goes to `Error`. Ideal for one-off troubleshooting pods or strict single-run scripts.

> [!WARNING] Manifest Placement
> `restartPolicy` is a sibling to `containers` and `volumes`. It MUST be placed directly under `spec:`, not nested inside a specific container's definition.