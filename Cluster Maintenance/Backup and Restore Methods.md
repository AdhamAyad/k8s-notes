# Comprehensive ETCD Backup & Restore Guide

**Tags:** #kubernetes #etcd #backup #restore #cka #operations
**Status:** Core Operations Guide

---

## 1. Core Prerequisites & Validation

Before performing any ETCD operation, you must verify the tool version. The cluster deployed via `kubeadm` runs ETCD as a static pod, and all commands must interact with API version 3.

> [!IMPORTANT] Validation Command
> Verify the API version before proceeding:
> `etcdctl version`
> Ensure the output displays `API version: 3.5` (or any 3.x version).

---

## 2. Method A: Live Snapshot Backup (The Standard Way)

This is the official, secure method to back up a running Kubernetes cluster. It interacts directly with the ETCD API to generate a consistent snapshot file.

* **Scenario:** Routine daily backups, CKA exam tasks, and situations where the cluster is healthy and operational.
* **Constraints & Conditions:**
    * The ETCD service must be running.
    * You must authenticate using the ETCD PKI certificates.
    * Produces a single `.db` file.

### Backup Command
```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=[https://127.0.0.1:2379](https://127.0.0.1:2379) \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/backup/etcd-snapshot.db
````

### Verification Command

> [!NOTE] Best Practice
> 
> Always verify the integrity of the snapshot immediately after creation.

Bash

```
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /opt/backup/etcd-snapshot.db
```

### Restore Procedure (Method A)

To restore a `.db` snapshot, you must extract it to a new directory and then point the static pod to that new location.

**Step 1: Stop the ETCD Pod**

Bash

```
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
```

**Step 2: Extract the Snapshot to a New Directory**

Bash

```
etcdutl snapshot restore /opt/backup/etcd-snapshot.db --data-dir /var/lib/etcd-restored
```

**Step 3: Clear the Corrupted Data Directory**

Bash

```
rm -rf /var/lib/etcd/*
```

**Step 4: Copy the Backup Data to the Original Path**

Bash

```
cp -a /var/lib/etcd-restored/* /var/lib/etcd/
```


**Step 5: Restart the ETCD Pod**

Bash

```
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```

---

## 3. Method B: Offline Raw Backup (The Disaster Recovery Way)

This method bypasses the API completely and copies the raw backend database and Write-Ahead Log (WAL) files directly from the Linux filesystem.

- **Scenario:** Total API failure, crashed ETCD pod, or rescuing data from a corrupted node's hard drive.
    
- **Constraints & Conditions:**
    
    - The ETCD pod should ideally be stopped during the copy to prevent data corruption.
        
    - Does not require certificates or API access.
        
    - Produces a full directory containing raw files, not a single snapshot file.
        

> [!WARNING] Crucial Requirement
> 
> Always stop the ETCD static pod before performing an offline backup or restore to avoid inconsistent writes.

### Backup Command

**Step 1: Stop the ETCD Pod**

Bash

```
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
```

**Step 2: Copy the Raw Data**

Bash

```
etcdutl backup --data-dir /var/lib/etcd --backup-dir /opt/backup/etcd-raw-backup
```

**Step 3: Restart the Pod (If returning to normal operations)**

Bash

```
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```

### Restore Procedure (Method B)

Since the backup is just raw files, the restore process is a standard Linux file copy operation back into the original directory.

**Step 1: Stop the ETCD Pod**

Bash

```
mv /etc/kubernetes/manifests/etcd.yaml /tmp/
```

**Step 2: Clear the Corrupted Data Directory**

Bash

```
rm -rf /var/lib/etcd/*
```

**Step 3: Copy the Backup Data to the Original Path**

Bash

```
cp -a /opt/backup/etcd-raw-backup/* /var/lib/etcd/
```

**Step 4: Restart the ETCD Pod**

Bash

```
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```
