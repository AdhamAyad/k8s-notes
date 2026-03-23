# Comprehensive Guide to Kubernetes Ingress and Network Routing

**Tags:** #kubernetes #ingress #networking #cka #metallb #routing
**Status:** Master Reference Note

---

## 1. Core Terminology and Concepts

Before diving into Ingress, it is crucial to understand the foundational network terms in a Kubernetes cluster:
* **Cluster Network:** The internal logical network that facilitates communication between Pods and Services.
* **Service:** A stable abstraction that identifies a set of Pods using label selectors and provides a stable internal IP (ClusterIP).
* **Edge Router:** The firewall or gateway that separates the cluster network from the public internet.

> [!INFO] What is an Ingress?
> An **Ingress** is an API object that exposes HTTP and HTTPS routes from outside the cluster to Services within the cluster. It maps external traffic to internal backends based on rules you define (URIs, hostnames, paths). 
> **Important:** Creating an Ingress resource alone does nothing. You MUST have an **Ingress Controller** running in your cluster to fulfill the rules.



---

## 2. The Bare-Metal LoadBalancer Scenario (MetalLB)

In cloud environments (AWS, GCP), requesting a Service of `type: LoadBalancer` automatically provisions an external load balancer. In bare-metal (local) environments, this stays in a `<pending>` state forever. 

**The Solution: MetalLB**
MetalLB acts as the local cloud provider. It listens for `LoadBalancer` services and assigns them a physical IP address from a pre-configured pool.

> [!NOTE] MetalLB Workflow
> 1. **Deploy MetalLB:** Runs controller and speaker pods.
> 2. **IPAddressPool:** You define a custom resource specifying a range of available IPs on your local router's subnet (e.g., `192.168.1.240-192.168.1.250`).
> 3. **L2Advertisement:** Instructs MetalLB to announce these IPs to the local network using ARP.
> 4. **Ingress Controller Deployment:** The NGINX Ingress Controller creates a `type: LoadBalancer` service. MetalLB assigns it one IP (e.g., `192.168.1.240`).



---

## 3. The Ingress Controller Architecture

The Ingress Controller is the actual "Gatekeeper" (usually a Pod running NGINX, Traefik, or HAProxy) that processes the traffic.

> [!IMPORTANT] Controller vs. Resource Scope
> * **The Controller:** Typically deployed in its own namespace (e.g., `ingress-nginx`). It possesses a `ClusterRole` that grants it cluster-wide read access. This allows it to monitor every namespace for new Ingress Resources.
> * **The Resources:** Ingress routing rules (`Ingress` objects) are **Namespaced**. They MUST reside in the same namespace as the backend Services they are routing traffic to.
> 
> A single Ingress Controller IP can serve hundreds of Ingress Resources spread across multiple namespaces. It dynamically updates its internal `nginx.conf` whenever a new Ingress Resource is created.

---

## 4. Routing Use Cases and Scenarios

### Scenario A: Simple Fanout (Path-Based Routing)
Routing traffic from a single domain to multiple Services based on the URL path.
* `example.com/foo` -> Service A
* `example.com/bar` -> Service B



### Scenario B: Name-Based Virtual Hosting (Host-Based Routing)
Routing traffic to entirely different applications based on the Domain Name (Host header) provided by the user, utilizing the exact same physical IP address.
* `app1.example.com` -> Service 1 (in Namespace A)
* `app2.example.com` -> Service 2 (in Namespace B)



### Scenario C: The "Rewrite-Target" Dilemma
Backend applications are usually designed to serve content from their root path (`/`). If an Ingress forwards traffic based on a path like `/watch`, the backend app receives `/watch` and throws a `404 Not Found` error because it does not have a `/watch` directory.

> [!WARNING] The Rewrite-Target Annotation Solution
> To prevent 404 errors, we use the `nginx.ingress.kubernetes.io/rewrite-target: /` annotation. 
> This acts as a search-and-replace mechanism. If a user requests `example.com/watch`, the Ingress Controller intercepts it, strips the `/watch` segment, and forwards a clean request to `/` on the backend Pod.

---

## 5. Path Types and Wildcards

When defining paths in an Ingress rule, you must explicitly declare a `pathType`:
1.  **Prefix:** Matches based on a URL path prefix split by `/`. (e.g., `/foo` matches `/foo/bar` but not `/foobar`).
2.  **Exact:** Matches the URL path exactly and with case sensitivity.
3.  **ImplementationSpecific:** Matching logic is deferred to the specific Ingress Controller being used.

**Hostname Wildcards:**
You can use wildcards in the `host` field. A rule for `*.foo.com` will match `app.foo.com` and `api.foo.com`, but it will NOT match `foo.com` or `my.app.foo.com` (it only covers a single DNS label).

---

## 6. The Ultimate Ingress Manifest (All Options)

Below is a comprehensive manifest showcasing all discussed options: Annotations, TLS, Virtual Hosting, Fanout, Regex rewriting, and Resource Backends.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ultimate-ingress-master
  namespace: production
  annotations:
    # Rewrite target using regex capture groups to strip the first path segment
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    # Define a custom load balancing algorithm
    nginx.ingress.kubernetes.io/load-balance: "round_robin"
    # Force HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  # Links this resource to a specific Ingress Controller configuration
  ingressClassName: nginx-external
  
  # Default backend if no rules match
  defaultBackend:
    service:
      name: default-error-handler
      port:
        number: 8080

  # TLS Configuration for HTTPS termination
  tls:
  - hosts:
    - secret-app.example.com
    - "*.wildcard.example.com"
    secretName: wildcard-tls-cert-secret

  rules:
  # 1. Virtual Hosting + Regex Path + Service Backend
  - host: secret-app.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80

  # 2. Simple Fanout 
  - host: store.example.com
    http:
      paths:
      - path: /catalog
        pathType: Exact
        backend:
          service:
            name: catalog-service
            port:
              number: 80
      - path: /cart
        pathType: Prefix
        backend:
          service:
            name: cart-service
            port:
              number: 80

  # 3. Wildcard Host Routing
  - host: "*.wildcard.example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wildcard-handler-service
            port:
              number: 80

  # 4. Resource Backend (Forwarding to an Object Storage Bucket directly)
  - host: static.example.com
    http:
      paths:
      - path: /assets
        pathType: ImplementationSpecific
        backend:
          resource:
            apiGroup: k8s.example.com
            kind: StorageBucket
            name: static-assets-bucket
````

---

## 7. Imperative Commands and Troubleshooting

> [!SUCCESS] The Speed Command for CKA
> 
> Do not waste time writing YAML from scratch in exams. Use the imperative command to generate an Ingress resource instantly:
> 
> Bash
> 
> ```
> kubectl create ingress ingress-test --rule="[wear.my-store.com/wear*=wear-service:80](https://wear.my-store.com/wear*=wear-service:80)"
> ```
> 
> This creates an Ingress named `ingress-test`, routing traffic from `wear.my-store.com/wear` to `wear-service` on port 80.

> [!WARNING] Troubleshooting: Resource vs. Controller
> 
> If routing fails, you must understand what you are inspecting:
> 
> - **To check the rules/mapping:** Run `kubectl describe ingress <ingress-name> -n <namespace>`. This shows the requested hosts, paths, and target services.
>     
> - **To check the actual engine:** Run `kubectl describe pod <controller-pod-name> -n ingress-nginx` or check its logs. This will reveal if the NGINX configuration failed to reload or if the controller lacks permissions.

---

## 8. IngressClass & Multiple Controllers Architecture

In real-world production environments, a cluster rarely relies on a single "gatekeeper" (Ingress Controller). We use the `IngressClass` to direct our traffic rules to the correct controller.

> [!INFO] Why use multiple Ingress Controllers?
> 1. **Network Isolation:** Dedicating one controller (Public) for internet visitors, and another (Private/Internal) strictly for internal employee applications.
> 2. **Performance Isolation:** Preventing high-traffic applications (like an e-commerce site) from consuming the controller's resources and bringing down sensitive applications (like a payment gateway).
> 3. **Technological Diversity:** Using Nginx for standard applications, while utilizing Traefik or HAProxy for applications requiring specific protocols like gRPC.

> [!IMPORTANT] What is an IngressClass?
> It is a distinct Kubernetes Resource, typically created automatically when you install an Ingress Controller. Its primary job is to act as an "identifier" or "profile" for that specific controller within the cluster.
> 
> To link your Ingress manifest to the desired controller, you take the name of this Class and specify it in the `ingressClassName` field inside your Ingress Resource.

> [!TIP] How to Install Multiple NGINX Controllers without Conflicts
> If you apply the default NGINX manifest twice, they will collide because both will attempt to create and use the `nginx` IngressClass and the same election IDs.
> 
> **The Production Solution (Using Helm):**
> You must override the default values during installation to ensure each controller operates in isolation. You need to change two critical parameters:
> 1. `controller.ingressClassResource.name` (The name you will use in your Ingress manifests).
> 2. `controller.ingressClassResource.controllerValue` (A unique identifier for the controller binary to watch).
> 
> **Helm Command Example:**
> ```bash
> helm install internal-nginx ingress-nginx/ingress-nginx \
>   --namespace internal-ingress --create-namespace \
>   --set controller.ingressClassResource.name=nginx-internal \
>   --set controller.ingressClassResource.controllerValue="k8s.io/ingress-nginx-internal" \
>   --set controller.ingressClass=nginx-internal
> ```
> By doing this, the new controller will ONLY process Ingress resources that explicitly state `ingressClassName: nginx-internal`.

> [!NOTE] Where does the `controllerValue` live?
> The `controllerValue` is literally written inside the `IngressClass` resource manifest under the `spec.controller` field.
> 
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: IngressClass
> metadata:
>   name: nginx-internal # The "name" you reference in your Ingress Resource
> spec:
>   controller: k8s.io/ingress-nginx-internal # The "controllerValue" (The internal ID)
> ```
> 
> **The Workflow:**
> 1. Your Ingress resource calls for `ingressClassName: nginx-internal`.
> 2. Kubernetes inspects the `nginx-internal` IngressClass object.
> 3. Kubernetes reads `spec.controller: k8s.io/ingress-nginx-internal`.
> 4. Only the NGINX Pods specifically deployed with that exact internal ID argument will process your Ingress rules. Other NGINX Pods with the default ID will safely ignore it.

```yaml
# Example: Routing traffic exclusively to an internal controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: internal-hr-app
  namespace: hr-space
spec:
  ingressClassName: nginx-internal # This line prevents the public controller from processing this rule
  rules:
  - host: hr.company.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hr-service
            port:
              number: 80
````

> [!WARNING] What happens if you omit the `ingressClassName`?
> 
> - **If there is a Default Controller:** Kubernetes will automatically route your Ingress resource to it. (The default status is defined via a specific Annotation inside the IngressClass resource itself).
>     
> - **If there is NO Default Controller:** Your Ingress resource will be completely ignored by all controllers, and its Address will remain in a `<pending>` state forever.
>     
> - **If there are MULTIPLE Default Controllers:** The Admission Controller will reject the creation of your Ingress resource and throw an error because it cannot determine which controller is the intended target.
>     
