# Kubernetes EndpointSlices

**Tags:** #kubernetes #networking #services #endpoints #CKA

**Status:** Stable (v1.21+)

---

## 1. Overview

The **EndpointSlice API** (`discovery.k8s.io/v1`) is the scalable replacement for the legacy `Endpoints` object. It addresses network congestion issues in large clusters where updates to a single Pod previously triggered a full list update to all Nodes.

### Key Characteristics

- **Sharding:** Splits large lists of endpoints into smaller "slices" (default 100 endpoints per slice).
    
- **Scalability:** Only the modified slice is updated and transmitted to `kube-proxy`.
    
- **Dual Stack:** Supports separate slices for `IPv4` and `IPv6`.
    

---

## 2. Port Matching Logic

This mechanism governs how traffic flows between a Service and its underlying EndpointSlice.

### A. Blind Matching (Anonymous Ports)

Scenario: Service and Slice have **only one port** and **no names**.

> [!TIP] Behavior
> 
> Traffic flows regardless of port number mismatches.
> 
> - **Service:** Port 80
>     
> - **Slice:** Port 5000
>     
> - **Result:** **Success**. Kubernetes assumes implicit mapping when only one port exists on both sides.
>     

### B. Strict Matching (Named Ports)

Scenario: A `name` is assigned to the port in the Service definition.

> [!WARNING] Requirement
> 
> The `name` in the EndpointSlice must match the Service port name exactly.
> 
> - **Service:** `name: web`, Port 80
>     
> - **Slice:** `name: web`, Port 8080 $\rightarrow$ **Success**
>     
> - **Slice:** `name: http`, Port 8080 $\rightarrow$ **Failure** (Name mismatch)
>     
> - **Slice:** (No name), Port 8080 $\rightarrow$ **Failure** (Missing name)
>     

### C. Multi-Port Services

> [!IMPORTANT] Note
> 
> If a Service exposes multiple ports, defining **names is mandatory** for all ports to avoid ambiguity in routing. (See Section 5.B for the Manifest structure).

---

## 3. The Role of targetPort

The behavior of `targetPort` varies based on the configuration context:

|**Context**|**Behavior**|
|---|---|
|**Automatic (Selector)**|**Critical.** It dictates the specific port number populated in the auto-generated EndpointSlice.|
|**Manual (No Selector)**|**Ignored.** Used for documentation only. Traffic destination is strictly determined by the `port` field within the manual `EndpointSlice` manifest.|
|**String Value**|**Dynamic.** If `targetPort` is a string (e.g., "web"), Kubernetes resolves it to the container port with that name in the Pod spec. Decouples Service config from container changes.|

---

## 4. Use Case: Integrating External Services

A primary use case for manual EndpointSlices is the **"Service without Selector"** pattern. This allows internal cluster applications to communicate with external backends (e.g., an external Database, legacy VM, or SaaS API) through a standard Kubernetes Service.

### The Architecture

`[Pod]` $\rightarrow$ `[Service IP]` $\rightarrow$ `[EndpointSlice]` $\rightarrow$ `[External Real IP]`

### Key Benefits

1. **Abstraction:** Applications communicate with a stable internal Service name (e.g., `my-db`), remaining agnostic to the backend's actual location or IP.
    
2. **Migration:** Enables seamless migration of external backends into the cluster. You can replace the manual EndpointSlice with a Selector-based Service later without changing a single line of application code.
    
3. **Unified Interface:** Provides a consistent discovery mechanism for both internal and external resources.
    

---

## 5. Manual Implementation Pattern

To implement external services or custom routing, you must link the Slice to the Service using the `kubernetes.io/service-name` label.

### A. Basic External Service (Single Port)

Use this when you have a simple backend (e.g., a database).

YAML

```
# 1. The Service
apiVersion: v1
kind: Service
metadata:
  name: external-backend-svc
spec:
  ports:
    - name: tcp-port
      port: 80
      targetPort: 8080       # Ignored (Documentation only)
---
# 2. The EndpointSlice
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: external-backend-slice
  labels:
    kubernetes.io/service-name: external-backend-svc # <--- Critical Link
addressType: IPv4
ports:
  - name: tcp-port           # MUST match Service port name
    port: 3306               # Actual External Port
    protocol: TCP
endpoints:
  - addresses:
      - "192.168.1.50"
    conditions:
      ready: true
```

### B. Multi-Port Strategy with Multiple IPs

Use this when the backend exposes multiple ports AND you have multiple backend instances (Load Balancing).

YAML

```
# 1. The Multi-Port Service
apiVersion: v1
kind: Service
metadata:
  name: app-with-metrics
spec:
  ports:
    - name: web-traffic      # (1) Name Required
      port: 80
      protocol: TCP
    - name: metrics-data     # (2) Name Required
      port: 9090
      protocol: TCP
---
# 2. The Multi-Port EndpointSlice
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: app-with-metrics-slice
  labels:
    kubernetes.io/service-name: app-with-metrics
addressType: IPv4
ports:
  # Mapping Port 1
  - name: web-traffic        # Matches Service Name (1)
    port: 8080               # Container Web Port
    protocol: TCP
  # Mapping Port 2
  - name: metrics-data       # Matches Service Name (2)
    port: 5555               # Container Metrics Port
    protocol: TCP
endpoints:
  # --- Backend 1 ---
  - addresses:
      - "10.0.0.5"           # First Backend IP
    conditions:
      ready: true
      
  # --- Backend 2 ---
  - addresses:
      - "10.0.0.6"           # Second Backend IP
    conditions:
      ready: true
    topology:
      zone: us-west-2a       # Optional topology info
```

---

## 6. Advanced Concepts

### Conditions (Endpoint States)

These boolean states tell the Load Balancer exactly how to handle traffic for each Pod.

- **`ready`** (The Green Light)
    
    - **Meaning:** The Pod is running and has passed its **Readiness Probe**.
        
    - **Action:** Send new traffic here immediately.
        
- **`terminating`** (The Graceful Shutdown)
    
    - **Meaning:** The Pod is being deleted but is still running during its _Grace Period_ (usually 30s) to finish existing tasks.
        
    - **Action:** Stop sending _new_ requests, but keep the connection open for _existing_ active requests until they finish (Connection Draining).
        
- **`serving`** (Is it Alive?)
    
    - **Meaning:** A broad state introduced in v1.26. It is true if the Pod is either `ready` OR `terminating`.
        
    - **Action:** Used by advanced Load Balancers/Ingress controllers that need to track the IP as long as the container is technically running, even if it's shutting down.
        

### Topology

Used for **Topology Aware Hints** to optimize network costs and latency by keeping traffic within the same zone.

- `nodeName`: The node hosting the endpoint.
    
- `zone`: Availability zone (e.g., `us-east-1a`).
    

### Shared Namespace Constraint

> [!NOTE]
> 
> Containers within the **same Pod** share the same Network Namespace (IP & Localhost). Consequently, two containers in the same Pod **cannot** bind to the exact same port number.