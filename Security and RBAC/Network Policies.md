

# Network Policies in Kubernetes

**Tags:** #kubernetes #security #network #cka #firewall

**Status:** Core Security Operations

---

## 1. Architectural Overview & Default Behavior

In a Kubernetes cluster, every Pod gets its own IP address. By default, Kubernetes operates on an **"All-Allow"** network model. This means any Pod can communicate with any other Pod across any namespace without restriction.

A **NetworkPolicy** acts as a Layer 3/4 (IP/Port) firewall for your Pods. It allows you to restrict Ingress (incoming) and Egress (outgoing) traffic.

> [!WARNING] The Prerequisite: CNI Support
> 
> Network policies are strictly implemented by the cluster's network plugin (CNI).
> 
> **Crucial Rule:** If you use a network plugin that does NOT support NetworkPolicies (like **Flannel**), you can still successfully create the YAML object via the API server, but it will be completely ignored and have **zero effect**.
> 
> _Supported CNIs:_ Calico, Weave Net, Kube-router, Cilium, Romana.

## 2. The Concept of "Isolation"

Understanding how a Pod transitions from "Open" to "Isolated" is key:

- **Non-Isolated (Default):** If no NetworkPolicy selects a Pod, it accepts traffic from anywhere and can send traffic anywhere.
    
- **Isolated:** The moment a NetworkPolicy selects a Pod using a `podSelector` and specifies a direction (`Ingress` or `Egress` in `policyTypes`), that Pod becomes **isolated** for that specific direction.
    
- **Additive Rules:** Once isolated, the Pod will _only_ allow traffic explicitly whitelisted by the policy. If multiple policies target the same Pod, their rules are combined additively (a union of allows). There are no "Deny" rules in NetworkPolicies; you secure pods by isolating them and selectively allowing traffic.
    

## 3. Targeting Entities (Selectors)

When defining `ingress.from` or `egress.to` rules, you specify who the isolated Pod can communicate with using three main identifiers:

1. **`podSelector`**: Selects specific Pods within the _same_ namespace.
    
2. **`namespaceSelector`**: Selects entire namespaces (allowing all Pods within those namespaces). _Note: You must label namespaces to target them, or use the immutable `kubernetes.io/metadata.name` label._
    
3. **`ipBlock`**: Selects external IP CIDR ranges (e.g., `172.17.0.0/16`). Do not use this for internal Pod IPs since they are ephemeral.
    

## 4. The YAML Syntax Trap: AND vs OR

> [!DANGER] Selector Combination (The Biggest Exam Trap)
> 
> How you format your YAML list dictates whether Kubernetes treats combined selectors as an **AND** condition or an **OR** condition.
> 
> **Scenario A: The "AND" Condition (Strict)**
> 
> Both selectors are under the same `-` array item. The source must be in the matching namespace AND have the matching label.
> 
> YAML
> 
> ```
> ingress:
> - from:
>   - namespaceSelector:
>       matchLabels:
>         project: myproject
>     podSelector:          # Notice no hyphen here
>       matchLabels:
>         role: frontend
> ```
> 
> **Scenario B: The "OR" Condition (Broad)**
> 
> Selectors are separate array items. The source can be ANY pod in the matching namespace OR any matching pod in the local namespace.
> 
> YAML
> 
> ```
> ingress:
> - from:
>   - namespaceSelector:
>       matchLabels:
>         project: myproject
>   - podSelector:          # Notice the hyphen here
>       matchLabels:
>         role: frontend
> ```

> [!TIP] Multiple Sources in Ingress/Egress Rules
> You can easily define multiple sources for your incoming traffic. How you structure the YAML depends on your port requirements.
> 
> **1. Same Ports for Multiple Sources:**
> Use a single `- from:` block and list multiple peers underneath it. This acts as a logical **OR**.
> ```yaml
> ingress:
> - from:
>   - podSelector: { matchLabels: { role: backend } }
>   - namespaceSelector: { matchLabels: { project: dev } }
>   ports:
>   - port: 80
> ```
> 
> **2. Different Ports for Different Sources:**
> Create completely separate rule blocks under `ingress:`. Each block gets its own `- from:` and its own `ports:`.
> ```yaml
> ingress:
> - from:
>   - podSelector: { matchLabels: { role: backend } }
>   ports:
>   - port: 80
> - from:
>   - ipBlock: { cidr: 10.0.0.0/24 }
>   ports:
>   - port: 443
> ```

> [!INFO] Securing Namespaces with Specific Ports (Logical AND)
> To restrict traffic not just to a specific namespace, but also to specific ports *within* that namespace, you must define the `ports` field as a sibling to the `to` field within the same array element.

> [!TIP] The Blueprint
> ```yaml
> egress:
>   - to: # Target selection
>     - namespaceSelector:
>         matchLabels:
>           kubernetes.io/metadata.name: space2
>     ports: # Port restriction for the ABOVE target (No dash!)
>     - port: 80
> ```
> *Explanation:* By omitting the dash before `ports`, you create a mandatory `AND` condition. Traffic is only permitted if both the destination namespace AND the destination port match exactly.

> [!EXAMPLE] The CKA Masterpiece (Namespace-Ports + DNS)
> In strict lockdown scenarios, you combine an `AND` rule for your application traffic with an `OR` rule for DNS:
> ```yaml
> egress:
>   - to:
>     - namespaceSelector: ...
>     ports: 
>     - port: 80 # App traffic locked to Port 80 in target namespace
>   - ports: 
>     - port: 53 # DNS traffic allowed globally
> ```
## 5. Port Targeting & Port Ranges

You can explicitly allow traffic on specific ports. As of Kubernetes v1.25, you can also target a contiguous block of ports using the `endPort` field (your CNI must support this feature).

YAML

```
ports:
- protocol: TCP
  port: 32000
  endPort: 32768      # Must be equal to or greater than 'port'
```

## 6. Best Practice: Default Deny-All Policies

The standard security posture in a production environment is "Zero Trust". You achieve this by creating a default deny-all policy for an entire namespace.

> [!IMPORTANT] Implementing Default Deny-All
> 
> By selecting all Pods (empty `podSelector`) and providing no allow rules, you instantly isolate every Pod in the namespace. You must then write specific policies to whitelist required traffic.
> 
> YAML
> 
> ```
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: default-deny-all
>   namespace: secure-namespace
> spec:
>   podSelector: {}       # Selects ALL pods in the namespace
>   policyTypes:
>   - Ingress
>   - Egress
>   # Missing ingress/egress blocks means NOTHING is allowed.
> ```

## 7. Current Limitations

NetworkPolicies are strictly Layer 3/4. Currently, you **cannot** use standard NetworkPolicies to:

- Filter traffic by HTTP/HTTPS path or domain name (Layer 7).
    
- Target Services directly by name (you must target the Pod labels backing the service).
    
- Explicitly write a "Deny" rule (it is strictly whitelist/allow-based).
    
- Force traffic through a central gateway.

---

## 8. Comprehensive Manifest Example (The Master Template)

Below is a fully-fledged NetworkPolicy that demonstrates how to combine multiple rules into a single, cohesive firewall configuration for a database Pod.

YAML

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: testing
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```

> [!INFO] Reading the Master Template
> 
> If you translate the above YAML into plain English:
> 
> "Isolate the **db** pods.
> 
> For incoming traffic, only allow connections on port **6379**, and only if the request comes from the **172.17.0.0/16** subnet, OR a namespace labeled **myproject**, OR a local pod labeled **frontend**.
> 
> For outgoing traffic, the **db** pod is only allowed to establish connections to the **10.0.0.0/24** subnet on port **5978**."

> [!IMPORTANT] Scope and Boundaries of Network Policies
> A `NetworkPolicy` is strictly a **Namespace-scoped** resource. It only isolates and protects Pods located in the exact same namespace where the policy itself is created. It has no authority over Pods residing in other namespaces.
> 
> **Rule 1: Standalone `podSelector` (Local Search Only)**
> If you specify a `podSelector` inside an `ingress.from` or `egress.to` block without a `namespaceSelector`, Kubernetes restricts its search to the **local namespace**. It will explicitly ignore any Pods in other namespaces, even if they possess the exact matching labels.
> 
> **Rule 2: Standalone `namespaceSelector` (Broad Access)**
> If you specify a `namespaceSelector` without a `podSelector`, you are granting blanket access. This allows traffic from **every single Pod** residing within the selected namespace(s), regardless of their individual pod labels.

> [!NOTE] The Target Scope of a NetworkPolicy
> A NetworkPolicy only applies to, and isolates, Pods located in the **exact same namespace** as the policy itself. 
> 
> **The Root podSelector:**
> When you define the target pods using `spec.podSelector` (at the top level of the `spec` section), Kubernetes exclusively searches for matching labels within that local namespace. 
> 
> *Architecture Rule:* You cannot create a NetworkPolicy in Namespace A to protect, isolate, or define rules for a Pod living in Namespace B. Each namespace must manage its own security policies.

> [!INFO] Network Policy "Empty" Selectors (The Wildcards)
> Understanding empty brackets `{}` or empty arrays `[]` is crucial for translating Network Policies. They represent a "Match All" condition within their specific context.

> [!TIP] The Cheat Sheet of Emptiness
> * **`podSelector: {}` (under `spec`):** Applies the policy to ALL pods in the current namespace.
> * **`podSelector: {}` (under `ingress/egress`):** Allows traffic from/to ALL pods strictly within the **same** namespace.
> * **`namespaceSelector: {}`:** Allows traffic from/to ALL namespaces in the entire cluster.
> * **Missing Allow Block (e.g., policy type is Egress, but no `egress:` block exists):** Implicitly **DENIES** all traffic for that direction.
> * **Empty Rule (`- to:` or `- from:` with no selectors):** Implicitly **ALLOWS** all traffic for that direction (effectively 0.0.0.0/0).
