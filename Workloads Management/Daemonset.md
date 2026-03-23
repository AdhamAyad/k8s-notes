# Kubernetes Controllers: DaemonSet

**Tags:** #kubernetes #daemonset #controller #scheduling #logging #monitoring #CKA

**Status:** Core Concept

---

## 1. Overview

A **DaemonSet** ensures that **all** (or some) Nodes run a specific copy of a Pod. It is designed for "Node-Local Facilities" (background processes that need to run on the infrastructure level).



### Core Behavior
* **Node Expansion:** As nodes are added to the cluster, the DaemonSet automatically adds Pods to them.
* **Node Removal:** As nodes are removed, those Pods are garbage collected.
* **Cleanup:** Deleting a DaemonSet will clean up (delete) all the Pods it created.

> [!NOTE] Difference from other Controllers
> Unlike a **[[Deployment]]** which focuses on maintaining a specific number of replicas across the cluster (using a **[[ReplicaSet]]**), a DaemonSet focuses on **topology**: ensuring one pod per node.

---

## 2. Typical Use Cases

DaemonSets are perfect for system daemons that need to run everywhere:

1.  **Logs Collection:** Running a logs daemon (e.g., `fluentd`, `logstash`) on every node to gather container logs.
2.  **Cluster Storage:** Running a storage daemon (e.g., `glusterd`, `ceph`) on every node.
3.  **Node Monitoring:** Running a monitoring daemon (e.g., `Prometheus Node Exporter`) to monitor node health (CPU, RAM, Disk).

---

## 3. Scheduling & Node Selection

How does Kubernetes decide where to put DaemonSet pods?

### A. Default Behavior
By default, a DaemonSet deploys a Pod on **every single node** in the cluster.

### B. Strict Selection (`nodeSelector`) - The "AND" Logic 
If you use `.spec.template.spec.nodeSelector`, the DaemonSet acts as a strict filter.
* **Logic:** It uses **AND** logic.
* **Scenario:** If you specify multiple labels (e.g., `disk: ssd` and `gpu: true`).
* **Result:** The Pod will **ONLY** run on nodes that have **BOTH** labels simultaneously. Nodes with only one of them will be ignored.

### C. Flexible Selection (`nodeAffinity`) - The "OR" Logic 
If you need to support multiple types of nodes (e.g., run on SSD nodes **OR** GPU nodes), you must use `.spec.affinity`.
* **Logic:** Defining multiple `nodeSelectorTerms` creates **OR** logic.
* **Result:** The Pod will run if **ANY** of the conditions are met.



### D. Dynamic Behavior (Real-time Watch) 
The DaemonSet controller constantly watches node labels:
* **Label Added:** If you add a matching label to an existing node, the DaemonSet immediately schedules a Pod there.
* **Label Removed:** If you remove the matching label, the DaemonSet immediately **terminates** the Pod on that node (Garbage Collection).

### E. Taints and Tolerations (Critical Feature) 
DaemonSet Pods are special. The controller automatically adds **Tolerations** to them so they can run on nodes where normal Pods cannot.

| Toleration Key | Effect | Why? |
| :--- | :--- | :--- |
| `node.kubernetes.io/not-ready` | `NoExecute` | Allows the daemon to run even if the node is not healthy yet. |
| `node.kubernetes.io/unreachable` | `NoExecute` | Prevents eviction if the node loses connection to the controller. |
| `node.kubernetes.io/disk-pressure` | `NoSchedule` | Runs even if the disk is full (critical for log/cleanup tools). |
| `node.kubernetes.io/unschedulable` | `NoSchedule` | Allows running on nodes marked as "Unschedulable". |

---

## 4. Update Strategies

Defined in `.spec.updateStrategy.type`.

| Strategy | Description | Key Behavior |
| :--- | :--- | :--- |
| **RollingUpdate** (Default) | Automatically updates Pods node by node. | **No Surge:** Since a node can only hold one Daemon pod, `maxSurge` does not exist. It only uses `maxUnavailable`. |
| **OnDelete** | Updates the template, but **does not** restart Pods automatically. | **Manual Trigger:** New Pods are only created when you manually **delete** the old Pod on a specific node. Useful for manual control. |

---

## 5. Stability & Rollout Safety 

How to prevent a bad update from killing the whole cluster (`minReadySeconds` nuance).

### A. The "Soak" Period (`minReadySeconds`)
This setting creates a buffer period (e.g., 10s) after the pod is `Ready` but before the rollout continues.

| Action | Behavior |
| :--- | :--- |
| **Traffic / Work** | Starts **IMMEDIATELY**. The pod receives live traffic as soon as it passes readiness probes. It *does not* wait for the 10s. |
| **Rollout / Update** | **WAITS** for the 10s. The controller monitors the pod during this time. |

### B. Failure Behavior (No Auto-Rollback) 
If a Pod crashes during the `minReadySeconds` window:
1.  The Rolling Update **PAUSES** (Stops) immediately.
2.  It does **NOT** automatically rollback.
3.  **Result:** One node is broken (the one being updated), but the rest of the cluster is safe on the old version.
4.  **Fix:** You must manually run `kubectl rollout undo`.

---

## 6. Advanced Behavior: Pod Adoption

What happens if a Pod already exists on a node with labels matching the DaemonSet selector?

1.  **Orphan Pod (Unmanaged):** The DaemonSet will **adopt** it (add an `OwnerReference`). If the pod spec doesn't match the template, it will eventually replace it.
2.  **Owned Pod (Managed by Deployment):** The DaemonSet will **NOT** touch it to avoid conflict. It will try to schedule its own pod (potentially causing a port conflict if they share host ports).

---

## 7. Communication Patterns

How do you talk to these Pods?

1.  **Push:** The Pods send data out (e.g., sending logs to a central ElasticSearch). They don't need incoming connections.
2.  **NodeIP & HostPort:** Clients use the Node IP and a known port. This bypasses Services/Ingress and maps the port directly to the Node's network interface.
3.  **Service (DNS):**
    * **Standard Service:** Load balances traffic to a random daemon pod.
    * **Headless Service:** Used to discover all daemon pod IPs via DNS endpoints.

---

## 8. DaemonSet vs. Deployment

This is the most common confusion. Here is when to use which:

| Feature | **[[Deployment]]** | **DaemonSet** |
| :--- | :--- | :--- |
| **Goal** | Run X copies of an app (Scalability). | Run 1 copy on **every** node (Infrastructure). |
| **Pod Distribution** | Pods can land anywhere (e.g., 5 pods on 1 node). | Strictly one Pod per Node. |
| **Use Case** | Frontend, Backend, Stateless Apps. | Networking plugins (CNI), Logging agents, Monitoring agents. |
| **Scaling** | You scale up/down manually or via HPA. | Scales automatically with the cluster size (Node count). |

---

## 9. Alternatives to DaemonSet

* **Init Scripts (systemd):** Running processes directly on the OS.
    * *Cons:* No Kubernetes monitoring, no resource limits, harder to manage configuration.
* **Static Pods:** Files placed in `/etc/kubernetes/manifests`.
    * *Cons:* Cannot be managed via `kubectl` or API. Useful only for bootstrapping the cluster itself.
* **Bare Pods:** Creating a Pod manually.
    * *Cons:* If it dies or the node fails, it's gone forever. DaemonSet ensures resurrection.

---

## 10. Master YAML Reference

A comprehensive example covering Rollback history, Stability, Update Strategies, Node Selection, Tolerations, and HostPorts.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  # --- 1. STABILITY & ROLLBACK ---
  revisionHistoryLimit: 5     # Keep last 5 ControllerRevisions for rollback
  minReadySeconds: 10         # Wait 10s after pod start before considering it "Safe".
                              # NOTE: Traffic starts immediately! This only pauses the rollout to next node.

  # --- 2. UPDATE STRATEGY ---
  # Option A: Rolling Update (Default)
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  
  # Option B: OnDelete (Manual Control)
  # updateStrategy:
  #   type: OnDelete
      
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      # --- 3. NODE SELECTION ---
      
      # Option A: NodeSelector (Simple AND logic)
      # nodeSelector:
      #   disk: ssd
      
      # Option B: Node Affinity (Complex OR logic)
      # affinity:
      #   nodeAffinity:
      #     requiredDuringSchedulingIgnoredDuringExecution:
      #       nodeSelectorTerms:
      #       - matchExpressions:     # Rule 1 (SSD)
      #         - key: disk
      #           operator: In
      #           values: ["ssd"]
      #       - matchExpressions:     # Rule 2 (GPU) - Acts as OR
      #         - key: gpu
      #           operator: In
      #           values: ["true"]

      # --- 4. TOLERATIONS ---
      # Allow running on Control Plane / Master nodes
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        
        # --- 5. NETWORKING (HostPort) ---
        ports:
        - containerPort: 80
          # hostPort: 8080      # Uncomment to open port 8080 directly on the Node IP
          protocol: TCP

        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      
      # --- 6. PRIORITY ---
      # priorityClassName: system-node-critical
      
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log