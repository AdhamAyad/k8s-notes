# Kubernetes Scheduling: Manual Pod Placement

**Tags:** #kubernetes #scheduling #pod #cka #binding #manifests

**Status:** Core Concept

## 1. Overview

By default, the `kube-scheduler` monitors the cluster for newly created Pods with no assigned Node (Status: `Pending`) and automatically selects the best Node based on resource availability and constraints.

However, in certain scenarios (like a failed scheduler or strict placement requirements), you must manually assign Pods to Nodes. There are three primary ways to achieve this.

## 2. Method A: `nodeName` (The Direct/Strict Way)

This is the most direct method to assign a Pod to a specific Node.

- **Mechanism:** It completely bypasses the `kube-scheduler`.
    
- **How it works:** When the API Server sees the `nodeName` field populated in the manifest, it assumes the Pod is already scheduled. The `kubelet` on that specific Node takes over immediately.
    
- **Risks:** If the specified Node does not exist, is cordoned, or lacks resources, the Pod will fail to start.
    

### YAML Implementation

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: strict-pod
spec:
  # Bypasses the scheduler entirely. Forces placement.
  nodeName: worker-node-01
  containers:
  - name: nginx
    image: nginx
```

## 3. Method B: `nodeSelector` (The Standard Way)

This is the recommended, safest approach for standard workloads. It relies on Node Labels and works in conjunction with the `kube-scheduler`.

- **Mechanism:** You instruct the scheduler to only place the Pod on a Node that contains a specific label.
    
- **Safety:** If the target Node goes down, the scheduler will wait and look for another Node that matches the label.
    

### Step 1: Label the Node

Bash

```
kubectl label nodes worker-node-02 disktype=ssd
```

### Step 2: YAML Implementation

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
spec:
  # The Scheduler will only pick nodes with this exact label
  nodeSelector:
    disktype: ssd
  containers:
  - name: nginx
    image: nginx
```

## 4. Method C: The Binding Object (The CKA Edge Case)

This is an advanced scenario heavily tested in the CKA exam.

- **The Scenario:** You have a Pod that is already created and stuck in a `Pending` state. The `kube-scheduler` is dead or missing.
    
- **The Constraint:** You are not allowed to delete and recreate the Pod. You cannot use `kubectl edit` to add a `nodeName` because it is an immutable field for existing Pods.
    
- **The Solution:** You must manually act as the scheduler by creating a `Binding` object and sending it directly to the API Server via an HTTP POST request.
    

### Step 1: Open a Proxy to the API Server

Run this in the background to safely communicate with the API server.

Bash

```
kubectl proxy &
```

### Step 2: Construct the Binding JSON

Create a file named `binding.json`. Notice there is no `spec` section.

JSON

```
{
  "apiVersion": "v1",
  "kind": "Binding",
  "metadata": {
    "name": "my-pending-pod"
  },
  "target": {
    "apiVersion": "v1",
    "kind": "Node",
    "name": "worker-node-01"
  }
}
```

### Step 3: POST the Binding via curl

Send the JSON payload directly to the specific Pod's binding endpoint. Once accepted, the API server assigns the Node, and the `kubelet` starts the Pod.

Bash

```
curl --header "Content-Type: application/json" \
     --request POST \
     --data @binding.json \
     http://localhost:8001/api/v1/namespaces/default/pods/my-pending-pod/binding
```