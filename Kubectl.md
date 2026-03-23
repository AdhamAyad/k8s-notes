# Kubectl Ultimate Cheat Sheet

**Tags:** #kubernetes #kubectl #cli #cheatsheet #cka #troubleshooting
**Status:** Reference Guide

---

## Table of Contents
*Click to jump to section*

- [[#1. Cluster Info & Context Management (Kubeconfig)|1. Cluster Info & Context Management]]
- [[#2. Object Introspection (Get & Describe)|2. Object Introspection (Get & Describe)]]
- [[#3. Workload Management (Create, Edit, Rollout)|3. Workload Management]]
- [[#4. Troubleshooting & Debugging (Logs, Exec)|4. Troubleshooting & Debugging]]
- [[#5. Labels, Annotations & Taints|5. Labels, Annotations & Taints]]
- [[#6. Namespace Management|6. Namespace Management]]
- [[#7. Advanced Formatting (JSONPath & Sort)|7. Advanced Formatting (JSONPath & Sort)]]
- [[#8. Declarative Management (YAML)|8. Declarative Management (YAML)]]
- [[#9. Service Management (Create & Expose)|9. Service Management (Create & Expose)]]

---

## 1. Cluster Info & Context Management (Kubeconfig)
*Essential commands for managing your connection to the cluster.*

### Basic Inspection

```bash
kubectl cluster-info
````

- **Purpose:** Checks if the Control Plane (Master) is running and accessible.
    

Bash

```
kubectl api-resources
```

- **Purpose:** Lists all resource types available in the cluster (e.g., pods, services, deploy) and their **shortnames** (e.g., `po`, `svc`).
    

### Context Switching

_Manage multiple clusters defined in your `~/.kube/config`._

Bash

```
kubectl config view
```

- **Purpose:** Displays the content of your kubeconfig file (clusters, users, and contexts).
    

Bash

```
kubectl config get-contexts
```

- **Purpose:** Lists all configured contexts. The one with `*` is the active one.
    

Bash

```
kubectl config current-context
```

- **Purpose:** Returns the name of the currently active context only.
    

Bash

```
kubectl config use-context <context-name>
```

- **Purpose:** Switches your active session to the specified context.
    

### Manual Context Creation (The Hard Way)

_How to build a kubeconfig from scratch. See [[Control Plane Installation Guide]] for certificate locations._

**Step 1: Define Cluster (Location & CA)**

Bash

```
kubectl config set-cluster production-cluster \
  --server=[https://192.168.1.50:6443](https://192.168.1.50:6443) \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true
```

**Step 2: Define User (Credentials)**

Bash

```
kubectl config set-credentials prod-admin \
  --client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \
  --client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key \
  --embed-certs=true
```

**Step 3: Link Cluster & User (Create Context)**

Bash

```
kubectl config set-context prod-context \
  --cluster=production-cluster \
  --user=prod-admin \
  --namespace=default
```

**Step 4: Create a Shortcut Context**

Bash

```
kubectl config set-context dev-ns-ctx --namespace=project-a --cluster=kubernetes --user=kubernetes-admin
```

---

## 2. Object Introspection (Get & Describe)

_The most used commands for checking status._

### Listing Resources

Bash

```
kubectl get nodes
```

- **Purpose:** Lists infrastructure nodes and their status (Ready/NotReady).
    

Bash

```
kubectl get pods
```

- **Purpose:** Lists Pods in the _current_ namespace.
    

Bash

```
kubectl get deploy,rs,svc
```

- **Purpose:** Lists multiple resource types at once.
    

### Advanced Formatting Flags

Bash

```
kubectl get pods -o wide
```

- `-o wide`: Adds columns for **Pod IP** and **Node Name**.
    

Bash

```
kubectl get pods --show-labels
```

- `--show-labels`: Adds a column showing all labels.
    

Bash

```
kubectl get pods -o yaml > pod-backup.yaml
```

- `-o yaml`: Exports the resource state as YAML code.
    

### Watching Live Changes

Bash

```
kubectl get pods -n kube-system -w
```

- `-w` (watch): Keeps the terminal open and updates the list in real-time.
    

### Deep Inspection

Bash

```
kubectl describe pod <pod-name>
```

- **Purpose:** Shows detailed configuration, **Events** (errors, image pull status), and conditions.
    

---

## 3. Workload Management (Create, Edit, Rollout)

_Managing applications. Related notes: [[Deployment]], [[ReplicaSet]]._

### Imperative Creation (Quick Testing)

Bash

```
kubectl run nginx-pod --image=nginx --restart=Never
```

- **Purpose:** Creates a single Pod immediately.
    

Bash

```
kubectl create deployment my-dep --image=nginx --replicas=3
```

- **Purpose:** Creates a Deployment object which manages 3 Pods.
    

Bash

```
kubectl create deployment my-dep --image=nginx --dry-run=client -o yaml
```

- **`--dry-run=client` (Local Generation):**
    
    - **Logic:** The `kubectl` binary on your laptop generates the structure locally.
        
    - **Result:** **Clean YAML**. Contains only what you explicitly typed.
        
    - **Best For:** Creating templates/manifests to save and edit.
        
- **`--dry-run=server` (Remote Simulation):**
    
    - **Logic:** The request is sent to the API Server. It runs validations and policies but stops before saving.
        
    - **Result:** **Verbose YAML**. Contains server-generated fields (`uid`, `status`).
        
    - **Best For:** Debugging policies or seeing exactly how the cluster will modify your object.
        

### Ephemeral Testing (Run Once & Delete)

_Create a temporary pod to run a single command and delete itself._

Bash

```
# Syntax: kubectl run <name> --image=<img > -it --rm --restart=Never -- <command>
kubectl run temp-test --image=busybox -it --rm --restart=Never -- echo "Hello Kubernetes"
```

- **`--rm`**: Deletes the pod immediately after the command finishes.
    
- **`-it`**: Interactive mode (to see output).
    

### Editing & Scaling

Bash

```
kubectl edit deployment my-dep
```

- **Purpose:** Opens the live configuration in the default editor.
    

Bash

```
kubectl scale deployment my-dep --replicas=5
```

- **Purpose:** Updates the replica count.
    

### Rollouts (Updates)

Bash

```
kubectl set image deployment/my-dep nginx=nginx:1.16.1
```

- **Purpose:** Triggers a Rolling Update.
    

Bash

```
kubectl rollout status deployment/my-dep
```

- **Purpose:** Watches the progress of the update.
    

Bash

```
kubectl rollout history deployment/my-dep
```

- **Purpose:** Lists previous revisions.
    

Bash

```
kubectl rollout undo deployment/my-dep
```

- **Purpose:** Reverts the deployment to the previous stable revision.
    

---

## 4. Troubleshooting & Debugging (Logs, Exec)

### Logs

Bash

```
kubectl logs <pod-name>
```

- **Purpose:** Fetches logs (Standard Output).
    

Bash

```
kubectl logs <pod-name> -f
```

- `-f`: Streams logs live.
    

Bash

```
kubectl logs <pod-name> -c <container-name>
```

- `-c`: Mandatory if the Pod has multiple containers.
    

Bash

```
kubectl logs -l app=frontend --tail=20
```

- `-l`: Selects ALL pods with label `app=frontend` and combines logs.
    

### Exec (Shell Access)

Bash

```
kubectl exec -it <pod-name> -- /bin/bash
```

- **Purpose:** Opens an interactive shell inside the pod.
    

Bash

```
kubectl exec <pod-name> -- ls /var/log
```

- **Purpose:** Runs a single command without entering.
    

### Port Forwarding (Secure Tunnel)

Bash

```
kubectl port-forward pod/mysql-pod 3306:3306
```

- **Purpose:** Creates a tunnel from `localhost:3306` to `Pod:3306`. Connection terminates when you close the terminal.
    

### Verifying Port Forward Status

Bash

```
# Method 1: Check Process
ps aux | grep "port-forward"

# Method 2: Check Network Port
netstat -tpln | grep 3306
```

---

## 5. Labels, Annotations & Taints

### Labels (Selecting & Organizing)

Bash

```
kubectl label node worker-node-01 disktype=ssd
```

- **Purpose:** Adds a label.
    

Bash

```
# Add multiple labels (Comma separated, no spaces)
kubectl run my-pod --image=nginx --labels="app=web,env=prod,tier=backend"
```

Bash

```
kubectl label pod nginx-pod app=v2 --overwrite
```

- `--overwrite`: Forces update if label exists.
    

Bash

```
kubectl label pod nginx-pod app-
```

- **Purpose:** Removes the label `app`.
    

### Taints (Node Protection)

Bash

```
kubectl taint nodes node1 key=value:NoSchedule
```

- **Effect:** Prevents new pods from scheduling on this node.
    

Bash

```
kubectl taint nodes node1 key:NoSchedule-
```

- **Purpose:** Removes the taint.
    

---

## 6. Namespace Management

Bash

```
kubectl create ns project-a
```

Bash

```
kubectl get ns
```

Bash

```
kubectl delete ns project-a
```

- **Warning:** Deletes the Namespace AND all resources inside it.
    

---

## 7. Advanced Formatting (JSONPath & Sort)

Bash

```
kubectl get pods --sort-by=.metadata.creationTimestamp
```

Bash

```
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP
```

Bash

```
kubectl get pod nginx-pod -o jsonpath='{.spec.containers[0].image}'
```

---

## 8. Declarative Management (YAML)

Bash

```
kubectl apply -f ./manifest.yaml
```

Bash

```
kubectl diff -f ./manifest.yaml
```

Bash

```
kubectl delete -f ./manifest.yaml
```

---

## 9. Service Management (Create & Expose)

_Commands to expose workloads to the network._

### Method A: The "Expose" Shortcut

_Best when you already have a Deployment and want to create a Service for it._

Bash

```
# Syntax: kubectl expose <type> <name> --type=<svc-type> --port=<svc-port> --target-port=<pod-port>
kubectl expose deployment my-dep --type=NodePort --port=80 --target-port=8080 --name=my-svc
```

- **Note:** For NodePort, this assigns a **Random** port. You cannot specify the nodePort here.
    

### Method B: The "Create" Command (Manual)

_Best for creating services from scratch with specific requirements._

**1. Create ClusterIP (Default)**

Bash

```
kubectl create service clusterip my-svc --tcp=80:80
```

**2. Create NodePort (With Static Port)**

Bash

```
# Syntax: --tcp=<svc-port>:<target-port> --node-port=<external-port>
kubectl create service nodeport my-svc --tcp=80:8080 --node-port=30007
```

**3. Create Multi-Port Service**

Bash

```
kubectl create service nodeport my-svc --tcp=80:80 --tcp=9090:9090
```

### Method C: Generate Service YAML

_Best practice for production templates._

Bash

```
kubectl create service nodeport my-svc --tcp=80:80 --node-port=30007 --dry-run=client -o yaml
```