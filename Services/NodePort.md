# Kubernetes Services: NodePort

**Tags:** #kubernetes #service #networking #nodeport #externalaccess #CKA

**Status:** Core Concept

---

## 1. Overview

A **NodePort** Service is a type of Service that exposes the application on a specific port on **every Node's IP** in the cluster. It acts as a bridge between the external world and the internal application.

### Core Behavior

- **External Access:** It opens a port (range 30000-32767) on all Nodes. Clients connect via `<NodeIP>:<NodePort>`.
    
- **Underlying ClusterIP:** When you create a NodePort, Kubernetes **automatically** creates a ClusterIP Service underneath to handle the internal routing.
    
- **Protocol Matching:** It allocates a port for TCP, UDP, or SCTP to match the service protocol.
    

> [!NOTE] The Proxy Logic
> 
> Every node in the cluster configures itself (via `kube-proxy`) to listen on that assigned port and forward traffic to one of the ready endpoints, even if the Pod is running on a _different_ node.

---

## 2. The "Ports" Trinity

Understanding the chain of forwarding is the most critical part of NodePort.

|**Field**|**Description**|**Flow**|**Range**|
|---|---|---|---|
|**`nodePort`**|The port opened on the **Physical Node**.|Client -> **Node:30007**|30000-32767|
|**`port`**|The port on the **Internal Service VIP**.|Node -> **Service:80**|Any|
|**`targetPort`**|The port the **Pod/Container** listens on.|Service -> **Pod:8080**|App-specific|

---

## 3. Port Allocation Strategy (Avoid Collisions)

Kubernetes splits the NodePort range (`30000-32767`) into two bands to minimize conflicts between manual assignment and auto-assignment.

### A. Dynamic Band (Auto-Assignment)

- **Range:** `30086 - 32767` (The Upper Band).
    
- **Behavior:** If you do **NOT** specify a `nodePort`, Kubernetes picks a random port from this range.
    

### B. Static Band (Manual Assignment)

- **Range:** `30000 - 30085` (The Lower Band).
    
- **Behavior:** If you want to force a specific port (e.g., `30007`), it is safer to pick from this range to ensure the cluster hasn't already given it to someone else.
    

> [!WARNING] Collision Risk
> 
> If you manually specify a port that is already in use, the API transaction will **FAIL**. You must manage these collisions yourself.

---

## 4. Advanced Network Configuration

### Custom Interface Binding (`--nodeport-addresses`)

By default, `kube-proxy` listens on **ALL** network interfaces of the Node. You can restrict this for security.

- **Config:** Set via the `--nodeport-addresses` flag in `kube-proxy`.
    
- **Example:** `127.0.0.0/8` (Loopback only) or `10.0.0.0/8` (Private Network only).
    
- **Result:** The Service is only accessible via the filtered Node IPs.
    

---

## 5. Troubleshooting Checklist

1. **Check Assigned Port:**
    
    - `kubectl get svc <svc-name>` -> Look at `PORT(S)` (e.g., `80:30007/TCP`).
        
2. **Firewall / Security Groups:**
    
    - **Crucial:** Ensure your Cloud Provider (AWS/Azure) or VM Firewall allows traffic on the specific `nodePort` (e.g., 30007).
        
3. **Local Test:**
    
    - `curl <NodeIP>:<NodePort>` from outside the cluster.
        
    - `curl localhost:<NodePort>` from inside the Node itself.
        

---

## 6. Master YAML Reference

A comprehensive example covering Manual Port Allocation, Protocols, and Traffic Policies.

YAML

```
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
  namespace: prod
  labels:
    app: frontend
spec:
  # --- 1. SERVICE TYPE ---
  type: NodePort
  
  # --- 2. SELECTOR ---
  selector:
    app: frontend
    role: ui

  # --- 3. PORTS CONFIGURATION ---
  ports:
    - name: http-web          # Mandatory if multi-port
      protocol: TCP           # TCP, UDP, or SCTP
      
      # A. Internal Service Port (VIP)
      port: 80
      
      # B. Container Port (Pod)
      # By default, equals 'port' if not specified.
      targetPort: 8080
      
      # C. External Node Port (The Gate)
      # Optional: If removed, K8s picks from Dynamic Band (30086-32767).
      # Manual: Pick from Static Band (30000-30085) to avoid conflicts.
      nodePort: 30007

  # --- 4. TRAFFIC POLICY (Performance) ---
  # "Cluster" (Default): Hop to other nodes if needed (Good for HA).
  # "Local": Only accept traffic if the Pod is on THIS node (Preserves Source IP).
  externalTrafficPolicy: Cluster
  
  # --- 5. IP FAMILY ---
  ipFamilies:
    - IPv4
  ipFamilyPolicy: SingleStack
```