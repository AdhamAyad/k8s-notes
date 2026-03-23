# Kubernetes Workloads: StatefulSets

**Tags:** #kubernetes #workloads #statefulset #storage #networking #CKA

**Status:** Core Reference

---

## 1. Core Concept & Definition

**StatefulSet** is a workload API object used to manage stateful applications. It manages the deployment and scaling of a set of Pods and provides guarantees about the **ordering** and **uniqueness** of these Pods.

### Deployment vs. StatefulSet (The Comparison)

|**Feature**|**Deployment (Stateless)**|**StatefulSet (Stateful)**|
|---|---|---|
|**Pod Identity**|Interchangeable (Cattle). Random names (`app-5f67d8-xyz`).|**Sticky Identity (Pets).** Stable, ordinal names (`app-0`, `app-1`).|
|**Network Identity**|No specific DNS per pod. LoadBalanced via Service.|**Stable DNS.** Each pod has a unique hostname (`app-0.service.ns`).|
|**Storage**|Shared storage (ReadWriteMany) or Ephemeral.|**Unique Storage.** Each pod gets its own PVC via `volumeClaimTemplates`.|
|**Scaling Order**|Parallel (All at once).|**Sequential (Ordered).** 0 -> 1 -> 2.|
|**Termination**|Random order.|**Reverse Ordered.** 2 -> 1 -> 0.|

---

## 2. Mandatory Components

To successfully deploy a StatefulSet, you need three components working together:

1. **Headless Service:** Controls the network domain and provides stable DNS entries.
    
2. **StatefulSet Object:** The controller definition.
    
3. **VolumeClaimTemplate:** Automates the creation of PVCs for each Pod.
    

> [!CAUTION] Selector Constraint
> 
> You must set the `.spec.selector` field to match the labels in `.spec.template.metadata.labels`. Failing to specify a matching Pod Selector will result in a validation error during creation.

### Complete Manifest Example

YAML

```
# 1. The Headless Service (Required for Network Identity)
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None   # <--- Critical: Makes it Headless
  selector:
    app: nginx
---
# 2. The StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"  # <--- Must match the Headless Service name
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.24
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  # 3. VolumeClaimTemplates (Automated Storage Provisioning)
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

---

## 3. Network Identity & DNS

### The Headless Service Requirement

StatefulSets require a Headless Service (`clusterIP: None`) to be responsible for the network identity of the Pods.

- **Service Name:** The `spec.serviceName` in the StatefulSet must match the name of the Headless Service.
    
- **DNS Format:** `$(podname).$(service name).$(namespace).svc.cluster.local`
    
- **Example:** `web-0.nginx.default.svc.cluster.local`
    

### DNS Resolution Behavior

- **Lookup Headless Service:** Returns a list of IPs for all ready Pods (e.g., `10.1.1.1, 10.1.1.2`).
    
- **Lookup Specific Pod:** Returns the specific IP of that Pod.
    

> [!NOTE] DNS Negative Caching
> 
> Newly created Pods might not be immediately resolvable due to DNS negative caching (Negative TTL).

---

## 4. Storage & Persistence (VolumeClaimTemplates)

### How it works

Unlike Deployments which reference an existing PVC, StatefulSets define a **template**.

- When Pod `web-0` is created, the controller creates PVC `www-web-0`.
    
- When Pod `web-1` is created, the controller creates PVC `www-web-1`.
    

### Persistence Guarantee

- **Pod Failure:** If `web-0` fails and is rescheduled on a different node, it will re-attach to the **same** PVC (`www-web-0`). This ensures data persistence.
    
- **Scaling Down:** If you scale down (delete `web-2`), its PVC (`www-web-2`) is **NOT** deleted by default. This is for data safety.
    

### PVC Retention Policy (New Feature)

You can configure if PVCs should be deleted when the StatefulSet is deleted or scaled down.

YAML

```
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain  # (Default) Keep PVCs when StatefulSet is deleted
    whenScaled: Delete   # Delete PVCs when scaling down replicas
```

---

## 5. Deployment & Scaling Guarantees

### Ordered Creation

Pods are created in strictly increasing order index: 0, then 1, then 2.

- Pod `N` is not created until Pod `N-1` is **Running and Ready**.
    

### Ordered Termination

Pods are terminated in strictly reverse ordinal order: 2, then 1, then 0.

- Pod `N-1` is not terminated until Pod `N` is completely shutdown.
    

---

## 6. Update Strategies

### A. RollingUpdate (Default)

Updates Pods in reverse ordinal order (largest index first).

- **Partitioning:** You can split the update using `.spec.updateStrategy.rollingUpdate.partition`.
    
    - If `partition` is set to `2`: Only pods with index >= 2 will be updated. Pods 0 and 1 remain at the old version. Useful for Canary releases.
        
- **Max Unavailable (Beta):** Controls how many pods can be unavailable during an update (e.g., `maxUnavailable: 1`).
    

### B. OnDelete (Manual)

The controller will **not** automatically update Pods when you change the manifest.

- **Trigger:** You must **manually delete** a Pod to trigger the controller to recreate it with the new version.
    

---

## 7. Pod Management Policies

You can relax the ordering guarantees for faster startup using `.spec.podManagementPolicy`.

|**Policy**|**Description**|
|---|---|
|**OrderedReady** (Default)|Strict sequential startup (0 -> 1 -> 2). Waits for readiness.|
|**Parallel**|Launches or terminates all Pods simultaneously. Does not wait for readiness. Useful for stateless workloads that need unique identities but not strict ordering.|

---

## 8. Advanced Configuration & Tricks

### Revision History (ControllerRevision)

StatefulSets use a specific API object called `ControllerRevision` to track configuration changes.

- **`.spec.revisionHistoryLimit`:** Optional field (defaults to 10). It controls how many old versions (revisions) are kept to allow for rollbacks.
    
- **Best Practice:** Keep this between 5-10. Do not set to 0, or you cannot rollback.
    

### Minimum Ready Seconds

`.spec.minReadySeconds`: Specifies the minimum number of seconds a newly created Pod should be running and ready without crashing before it is considered "Available". Useful to slow down rollouts and catch errors early.

### Start Ordinal

`.spec.ordinals.start`: Allows starting the index from a number other than 0.

- Example: If `start: 10`, pods will be `web-10`, `web-11`, `web-12`.
    

### Forced Rollback & Broken States

If a Rolling Update fails (e.g., bad image), the rollout stops.

- **Fix:** Revert the template to the valid configuration.
    
- **Issue:** The controller might still wait for the broken pod to become ready (which will never happen).
    
- **Solution:** You must **manually delete** the broken Pods after reverting the template to force the controller to recreate them with the correct config.
    

### Pod Index Label

The controller automatically adds a label `apps.kubernetes.io/pod-index` to each Pod containing its ordinal number. This is useful for targeting specific pods with Services or monitoring tools.

---

## 9. Imperative Commands (Cheat Sheet)

**Scale StatefulSet**

Bash

```
kubectl scale statefulset web --replicas=5
```

**Debug & List**

Bash

```
# List StatefulSets
kubectl get sts

# Watch the ordered creation
kubectl get pods -w -l app=nginx
```

**Volume Management**

Bash

```
# Get PVCs associated with the set
kubectl get pvc -l app=nginx
```

**Delete (Cascading vs Non-Cascading)**

Bash

```
# Delete StatefulSet and its Pods (Standard)
kubectl delete sts web

# Delete StatefulSet but Keep Pods running (Orphan)
kubectl delete sts web --cascade=orphan
```

**Rollout Management**

Bash

```
# Check Rollout Status
kubectl rollout status sts/web

# View History (Uses ControllerRevision)
kubectl rollout history sts/web

# Undo Rollout
kubectl rollout undo sts/web
```