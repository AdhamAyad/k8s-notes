# Kubernetes Services: ClusterIP

**Tags:** #kubernetes #service #networking #clusterip #servicediscovery #CKA

**Status:** Core Concept

---

## 1. Overview

A **ClusterIP** Service is the **default** Kubernetes Service type. It provides a stable, internal Virtual IP (VIP) address that exposes a set of Pods to other objects _inside_ the cluster.

### Core Behavior

- **Internal Only:** The IP address is only accessible from within the cluster nodes/pods. It is **not** accessible from the internet.
    
- **Stable Identity:** Even if Pods die and are recreated (changing their IPs), the Service IP remains constant.
    
- **Load Balancing:** Traffic sent to the Service IP is distributed (Round-Robin by default) to the healthy backing Pods.
    

> [!NOTE] Relationship to other Services
> 
> ClusterIP is the foundation. Both **[[NodePort Service]]** and **[[LoadBalancer Service]]** actually create a ClusterIP service in the background to handle the internal routing before exposing it externally.

---

## 2. The "Ports" Triangle

Understanding how traffic flows through the ports is critical.

|**Field**|**Description**|**Diagram Flow**|
|---|---|---|
|**`port`**|The port the **Service** listens on. This is what clients call.|Client -> **Service:80**|
|**`targetPort`**|The port the **Pod/Container** is actually listening on.|Service -> **Pod:8080**|
|**`nodePort`**|**N/A**. This field is ignored/not used in ClusterIP mode.|N/A|

---

## 3. Service Discovery Mechanisms

How do applications find this Service?

### A. DNS (Recommended)

Kubernetes runs an internal DNS server (CoreDNS).

- **Format:** `<service-name>.<namespace>.svc.cluster.local`
    
- **Example:** `my-db.prod.svc.cluster.local`
    
- **Short name:** If the client pod is in the same namespace, it can just call `my-db`.
    

### B. Environment Variables (Legacy)

When a Pod starts, K8s injects environment variables for all _currently running_ services.

- **Format:** `MY_SERVICE_SERVICE_HOST` and `MY_SERVICE_SERVICE_PORT`.
    
- _Downside:_ If the Service is created _after_ the client Pod, the variables won't exist.
    

---

## 4. Advanced Configurations

### A. Headless Services (`ClusterIP: None`)

Used when you don't want load balancing or a single VIP. You want to talk to specific Pods directly (often used with **[[StatefulSet]]**).

- **Behavior:** DNS lookup returns **A Records** for _all_ the individual Pod IPs instead of one Service VIP.
    
- **Use Case:** Database replication, Cluster peering (Kafka, Zookeeper).
    

### B. Session Affinity (Sticky Sessions)

By default, traffic is random. If you need a client to always hit the _same_ Pod (e.g., for caching):

- **Setting:** `.spec.sessionAffinity: ClientIP`
    
- **Mechanism:** Routes traffic based on the source IP of the client.
    

### C. Multi-Port Services

A single service can expose multiple ports (e.g., Main App + Metrics).

- **Requirement:** You **MUST** give each port a `name` in the YAML if you have more than one.
    

### D. Internal Traffic Policy

- **`Cluster` (Default):** Route to any pod in the cluster.
    
- **`Local`:** Only route to pods running on the _same node_ as the client. If no pods are on that node, drop traffic. (Optimizes latency).
    

---

## 5. Troubleshooting Checklist

1. **Check Endpoints:** Always start here. If this is empty, the Service selects nothing.
    
    - `kubectl get endpoints <svc-name>`
        
2. **Check Selectors:** Do the labels in the Service match the labels on the Pod exactly?
    
    - See **[[Labels and Annotations]]** for matching logic.
        
3. **Check TargetPort:** Is the application inside the container actually listening on the `targetPort` defined?
    
4. **DNS Check:**
    
    - `kubectl run test --rm -it --image=busybox -- nslookup my-service`
        

---

## 6. Master YAML Reference

A comprehensive example covering Dual Stack (IPv4/IPv6), Static IPs, Multi-ports, Session Affinity, and Traffic Policies.

YAML

```
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: prod
  # --- 1. METADATA & ANNOTATIONS ---
  labels:
    app: backend
    tier: core
  annotations:
    service.beta.kubernetes.io/description: "Primary backend API"
spec:
  # --- 2. SERVICE TYPE ---
  type: ClusterIP   # Default. Internal Only.
  
  # --- 3. SELECTOR ---
  # Defines which Pods receive the traffic.
  # If removed, this becomes an "Abstract Service" (needs manual Endpoints).
  selector:
    app: backend
    role: api

  # --- 4. PORTS CONFIGURATION ---
  # You can define multiple ports. Names are required if >1 port.
  ports:
    - name: http-web          # Mandatory for multi-port
      protocol: TCP           # Options: TCP (default), UDP, SCTP
      appProtocol: http       # Hint for Service Mesh (Istio/Linkerd)
      port: 80                # The Service Port (Virtual)
      targetPort: 8080        # The Container Port (Actual). Can be Int or String (Name).
      
    - name: metrics-port
      protocol: TCP
      port: 9090
      targetPort: metrics     # Using a named port defined in the Pod spec

  # --- 5. IP CONFIGURATION (Advanced) ---
  # clusterIP: 10.96.0.50     # Optional: Manually request a specific VIP.
                              # If "None", it becomes a HEADLESS Service (DNS Round Robin).
  
  # Dual Stack Support (IPv4 and IPv6)
  ipFamilies:
    - IPv4
    - IPv6
  ipFamilyPolicy: PreferDualStack

  # --- 6. TRAFFIC BEHAVIOR ---
  # Session Affinity (Sticky Session)
  sessionAffinity: ClientIP            # Default is "None"
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800            # Stickiness duration (3 hours)

  # Traffic Policy
  internalTrafficPolicy: Cluster       # Options:
                                       # "Cluster" (Default): Route to any pod.
                                       # "Local": Only route to pods on the SAME Node (reduces latency).

  # --- 7. PUBLISH NOT READY ---
  # allow traffic even if readiness probe fails (Rare, useful for StateulSets/Discovery)
  publishNotReadyAddresses: false 
```