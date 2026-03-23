# Kubernetes Architecture: Static Pods

**Tags:** #kubernetes #architecture #static-pods #kubelet #cka
**Status:** Core Concept

## 1. Core Concept: What is a Static Pod?

In a standard Kubernetes environment, the `kubelet` receives instructions from the `kube-apiserver` to create pods. A **Static Pod** bypasses this entire process.

* **Independent Management:** Managed directly by the `kubelet` daemon on a specific node, completely independent of the control plane (`kube-apiserver` and `kube-scheduler`).


* **Directory Watching:** The `kubelet` watches a specific local directory on the host server and automatically creates, restarts, or deletes pods based purely on the YAML files present there.

* **Primary Use Case:** Bootstrapping the Kubernetes Control Plane. Components like `kube-apiserver`, `etcd`, `kube-scheduler`, and `kube-controller-manager` are deployed as static pods so the cluster can initialize itself.

## 2. Configuration: Finding the Manifest Directory

To create a static pod, you must place the pod's YAML definition file inside the designated directory on the node. 

**CKA Exam Strategy: Locating the Directory**
Do not guess the directory path. Always verify it using the node's terminal.

**Method A: Check the running process**

```bash
ps -aux | grep kubelet

````

Look for one of these two parameters in the output:

1. `--pod-manifest-path=/etc/kubernetes/manifests` (Direct path)
    
2. `--config=/var/lib/kubelet/config.yaml` (Points to a config file)
    

**Method B: Inspect the Config File**

If the process uses a config file, inspect it directly:

Bash

```
cat /var/lib/kubelet/config.yaml

```

Search for the `staticPodPath` key:

YAML

```
staticPodPath: /etc/kubernetes/manifests

```

## 3. Operations and Management

Because the `kube-apiserver` does not manage static pods, standard commands behave differently:

- **Creation:** Move or create a valid Pod YAML file inside the `staticPodPath` directory. The `kubelet` detects it instantly and starts the container.
    
- **Modification:** Edit the YAML file directly using `vi`. Upon saving, the `kubelet` instantly detects the change, terminates the existing container, and recreates it.
    
- **Deletion:** You **MUST** delete the YAML file from the directory using the `rm` command.
    

> [!WARNING] The "Immortal" Pod Trap
> 
> If you run `kubectl delete pod <static-pod-name>`, the pod will terminate briefly, but the `kubelet` will instantly recreate it because the physical file still exists on the server.

## 4. Clustered Environment & Mirror Pods

If the node is part of a functional Kubernetes cluster, the `kubelet` will automatically create a **Mirror Pod** on the `kube-apiserver`.

- **Visibility:** Allows you to see the static pod when running `kubectl get pods`.
    
- **Naming Convention:** The node's name is automatically appended to the pod's name (e.g., `static-web-node01`).
    
- **Read-Only Status:** This mirror pod is purely for visibility. You cannot edit or delete the actual static pod via API requests to the mirror pod.
    

## 5. Architectural Limitations

**Static pods ONLY support the `Pod` resource type.** If you place a YAML file for a `Deployment`, `ReplicaSet`, `Service`, or `DaemonSet` into the static pod directory, the `kubelet` will ignore it. The `kubelet` lacks the built-in controllers required to manage high-level abstractions.

## 6. Static Pods vs. DaemonSets

Although both ensure specific pods run on nodes, they serve completely different architectural purposes:

|**Feature**|**Static Pods**|**DaemonSets**|
|---|---|---|
|**Created By**|`kubelet` directly reading local files.|DaemonSet Controller via `kube-apiserver`.|
|**Control Plane Needed?**|No. Works even on an isolated standalone node.|Yes. Requires a fully functional API server.|
|**Main Use Case**|Bootstrapping core Control Plane components.|Cluster-wide agents (Logging, Networking plugins).|
|**Scheduler Interaction**|Ignored completely by `kube-scheduler`.|Bypasses scheduler via built-in Tolerations.|
