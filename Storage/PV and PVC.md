# Comprehensive Guide: Persistent Volume Claims (PVC)

**Tags:** #kubernetes #storage #pvc #pv #binding #cka

**Status:** Core Storage Operations Guide

---

## 1. Architectural Overview: Separation of Concerns

Persistent Volumes (PV) and Persistent Volume Claims (PVC) are two distinct objects in the Kubernetes storage architecture, designed to separate infrastructure provisioning from application deployment.

- **Administrator Role:** Responsible for provisioning the physical storage and creating the **PV** objects to represent that storage in the cluster.
    
- **Developer/User Role:** Responsible for creating **PVC** objects to request a specific amount of storage with specific access requirements for their applications.
    

---

## 2. The Binding Process and Criteria

When a PVC is created, the Kubernetes Control Plane automatically attempts to bind it to an available PV. This binding process evaluates several strict criteria:

1. **Capacity:** The PV must have a capacity equal to or greater than the requested storage in the PVC.
    
2. **Access Modes:** The PV must support the exact access modes requested by the PVC (e.g., `ReadWriteOnce`, `ReadWriteMany`).
    
3. **Volume Modes:** Block vs. Filesystem configurations must match.
    
4. **Storage Class:** The `storageClassName` must align between the two resources.
    

> [!TIP] Best Practice: Granular Binding with Selectors
> 
> If multiple PVs in the cluster can satisfy a single PVC's claim, Kubernetes might pick any of them randomly. To force a PVC to bind to one specific PV, administrators should apply labels to the PV and use a `selector` block within the PVC manifest to guarantee an exact match.

> [!SCENARIO] The Pending State Scenario
> 
> If a developer creates a PVC requesting 10Gi of storage, but the cluster only has 5Gi PVs available, the PVC will remain stuck in a `Pending` state. It will continuously wait in this state indefinitely until the cluster administrator provisions a new PV that satisfies the 10Gi requirement. Once a suitable PV appears, Kubernetes will automatically bind them.

> [!WARNING] The Capacity Waste Rule
> 
> The relationship between a PV and a PVC is strictly one-to-one. If a smaller PVC (e.g., 500Mi) is matched with a larger PV (e.g., 1Gi) that meets all other criteria, the unrequested capacity (the remaining 500Mi) remains completely locked and unused by any other PVC in the cluster.

---

## 3. Manifest Implementations

Below are complete YAML definitions demonstrating how a PVC and a PV are structured and how they align with each other.

### The PV Definition (Created by Administrator)

This manifest defines a 1Gi volume backed by an AWS Elastic Block Store (EBS) disk.

YAML

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-vol1
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

### The PVC Definition (Created by Developer)

This manifest requests 500Mi of storage with the same access mode.

YAML

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

### Execution and Verification

To create the PVC and check its status, use the following commands:

Bash

```
kubectl create -f pvc-definition.yaml
kubectl get persistentvolumeclaim
```

**Expected Output:**

Before finding a PV, the status is `Pending`. Once Kubernetes inspects the available PVs and finds `pv-vol1`, it will bind them, and the status will transition to `Bound`.

Plaintext

```
NAME      STATUS   VOLUME    CAPACITY   ACCESS MODES
myclaim   Bound    pv-vol1   1Gi        RWO
```

---

## 4. Deletion and Reclaim Policies

When an application is decommissioned, the developer will delete the PVC:

Bash

```
kubectl delete persistentvolumeclaim myclaim
```

What happens to the underlying PV and the physical data after the PVC is deleted depends entirely on the `persistentVolumeReclaimPolicy` defined inside the PV.

|**Reclaim Policy**|**Architectural Behavior**|
|---|---|
|**Retain**|The PV remains in the cluster but transitions to a `Released` state. The physical data is perfectly safe. An administrator must manually clean up the data and recreate the PV to reuse the physical storage.|
|**Delete**|The PV is automatically deleted from the cluster along with the PVC. Furthermore, Kubernetes commands the underlying storage provider to destroy the physical storage device. **All data is permanently lost.**|
|**Recycle**|The PV data is automatically scrubbed (e.g., `rm -rf /thevolume/*`) before being made `Available` again for new claims.|

> [!WARNING] Deprecation Notice
> 
> The `Recycle` reclaim policy is officially deprecated in recent Kubernetes versions and might be completely disabled in your cluster environment. Modern dynamic provisioning relies on the `Delete` or `Retain` policies instead.

To explicitly set a reclaim policy, you add the following line to your PV specification:

YAML

```
persistentVolumeReclaimPolicy: Retain
```