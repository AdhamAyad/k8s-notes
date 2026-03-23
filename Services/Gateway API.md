# Kubernetes Gateway API: The Ultimate Guide

The Gateway API is the official successor to the standard Kubernetes Ingress. It provides a modern, extensible, and role-oriented approach to managing ingress and traffic routing in Kubernetes, eliminating the need for complex, vendor-specific annotations.

> [!INFO] Why Gateway API replaces Ingress
> 
> 1. **Role-Oriented Design:** It separates infrastructure configuration (Ports, TLS) from application routing logic (Paths, Headers).
>     
> 2. **Highly Expressive:** Advanced features like traffic splitting, header manipulation, and URL rewriting are built natively into the API (no annotations needed).
>     
> 3. **Multi-Protocol Support:** Natively supports HTTP, HTTPS, TCP, UDP, and gRPC.
>     
> 4. **Cross-Namespace Routing:** Securely allows a central load balancer to route traffic to services in completely different namespaces.
>     

## 1. Core Architecture (The Mall Analogy)

The API breaks down the monolithic Ingress into distinct objects, manageable by different roles:

- **GatewayClass (The Management Company):** Defines which controller (e.g., NGINX, Istio) will implement and manage the Gateways.
    
- **Gateway (The Mall Gates & Security):** Managed by Infrastructure Admins. It defines the physical entry points (Ports, Protocols like HTTP/HTTPS) and handles TLS decryption. It does NOT know about application URLs.
    
- **HTTPRoute (The Receptionist/Directory):** Managed by Application Developers. It attaches to a Gateway and dictates where traffic should go based on paths, headers, or weights.
    

---

## 2. The Unified Manifest (All-in-One Example)

This manifest demonstrates how a `GatewayClass`, a `Gateway`, and an `HTTPRoute` work together in a single namespace to perform basic routing, URL rewriting, header injection, and traffic splitting (Canary).

YAML

```
# ==========================================
# 1. GatewayClass (The Management Blueprint)
# ==========================================
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: mall-nginx-class
spec:
  controllerName: nginx.org/gateway-controller

---
# ==========================================
# 2. Gateway (The Infrastructure Entrypoint)
# ==========================================
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-mall-gateway
  namespace: default
spec:
  gatewayClassName: mall-nginx-class # Linking to the class above
  listeners:
  - name: http-listener
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same # Only accepts routes from the same namespace

---
# ==========================================
# 3. HTTPRoute (The Routing Logic & Filters)
# ==========================================
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mall-traffic-router
  namespace: default
spec:
  parentRefs:
  - name: main-mall-gateway # Attaching this route to the Gateway above
  
  rules:
  # ---------------------------------------------------------
  # Feature 1: Basic Routing
  # Traffic to /clothes goes to the clothes-service
  # ---------------------------------------------------------
  - matches:
    - path:
        type: PathPrefix
        value: /clothes
    backendRefs:
    - name: clothes-service
      port: 80

  # ---------------------------------------------------------
  # Feature 2: URL Rewrite + Header Modification (Filters)
  # Traffic to /old-cinema is rewritten to /new-cinema
  # A custom header (x-visitor-type: VIP) is injected
  # ---------------------------------------------------------
  - matches:
    - path:
        type: PathPrefix
        value: /old-cinema
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          replacePrefixMatch: /new-cinema
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
          x-visitor-type: "VIP"
    backendRefs:
    - name: cinema-service
      port: 80

  # ---------------------------------------------------------
  # Feature 3: Traffic Splitting (Canary Deployment)
  # Traffic to /food is split: 80% to old service, 20% to new
  # ---------------------------------------------------------
  - matches:
    - path:
        type: PathPrefix
        value: /food
    backendRefs:
    - name: old-restaurant-service
      port: 80
      weight: 80   # 80% of traffic
    - name: new-restaurant-service
      port: 80
      weight: 20   # 20% of traffic
```

---

## 3. Advanced Traffic Control & Security

The Gateway API handles advanced networking requirements natively through `Filters` and specialized configurations.

### A. HTTP to HTTPS Redirects & TLS Termination

> [!IMPORTANT] Securing Traffic
> 
> TLS termination happens at the `Gateway` level (managed by the admin). Redirection happens at the `HTTPRoute` level via the `RequestRedirect` filter.

- **TLS Termination in Gateway:** Add a listener for HTTPS on port 443 and reference a Kubernetes Secret containing the certificate.
    
- **Forcing HTTPS:** Create an `HTTPRoute` attached to the HTTP listener (port 80) with a `RequestRedirect` filter to force the scheme to `https`.
    

### B. Request Mirroring (Traffic Shadowing)

> [!TIP] Testing in Production Safely
> 
> You can send a copy of live production traffic to a secondary service for testing/analysis without impacting the actual user experience.

- Use the `RequestMirror` filter inside an `HTTPRoute`. It duplicates the incoming request to a `mirror-service` while forwarding the primary request to the actual backend.
    

> [!IMPORTANT] Request Mirroring vs. Traffic Splitting (Crucial Distinction)
> It is a common misconception that mirroring is just traffic splitting without the `weight` field. They operate on completely different mechanisms:
> 
> * **Traffic Splitting (Canary):** Divides the traffic. A single request goes to EITHER Service A OR Service B. The user receives a response from whichever service received the request. This is configured entirely inside `backendRefs` using the `weight` field.
> * **Request Mirroring (Shadowing):** Duplicates the traffic. Every single request goes to Service A (which processes it and responds to the user). Simultaneously, a silent carbon copy is sent to Service B. The user never sees a response from Service B. This is configured using a specific `RequestMirror` Filter.
> 
> 
> 
> **The Mirroring Manifest Structure:**
> Notice how the mirror is defined as a `filter` *before* the actual `backendRefs`.
> 
> ```yaml
> apiVersion: gateway.networking.k8s.io/v1
> kind: HTTPRoute
> metadata:
>   name: checkout-mirror-route
>   namespace: production
> spec:
>   parentRefs:
>   - name: main-gateway
>   rules:
>   - matches:
>     - path:
>         type: PathPrefix
>         value: /checkout
>     
>     # 1. The Filter: Creates a silent carbon copy of the request
>     filters:
>     - type: RequestMirror
>       requestMirror:
>         backendRef:
>           name: mirror-audit-service # The testing/shadow service
>           port: 9000
> 
>     # 2. The Primary Backend: Handles the actual user traffic and responds
>     backendRefs:
>     - name: primary-checkout-service # The live production service
>       port: 80
> ```

### C. Cross-Namespace Routing & ReferenceGrant

> [!WARNING] Security Boundary Concept
> 
> By default, an `HTTPRoute` cannot route traffic to a `Service` in a different namespace. This prevents malicious developers from hijacking traffic intended for secure namespaces.

To allow cross-namespace routing, the target namespace MUST explicitly issue a `ReferenceGrant`.

YAML

```
# Example: Issued by the 'backend-ns' team to allow 'frontend-ns' to reach their services
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-frontend
  namespace: backend-ns
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: frontend-ns
  to:
  - group: ""
    kind: Service
```

Once this is applied, the `HTTPRoute` in `frontend-ns` can safely set a `backendRef` pointing to a service in `backend-ns`.

### D. Multi-Protocol Support (TCP, UDP, gRPC)

Unlike standard Ingress which is strictly L7 (HTTP/HTTPS), the Gateway API supports L4 routing natively.

- **TCPRoute:** Used for connection-oriented protocols (e.g., exposing a MySQL database on port 3306).
    
- **UDPRoute:** Used for connectionless protocols (e.g., exposing a DNS server on port 53).
    
- **GRPCRoute:** Supports matching specific gRPC services and methods directly, ensuring HTTP/2 flows properly without complex upgrades.