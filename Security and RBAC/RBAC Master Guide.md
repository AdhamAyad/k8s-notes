# Kubernetes Authorization: RBAC Master Guide

**Tags:** #kubernetes #security #RBAC #CKA #devops

**Status:** Core Reference

---

## 1. Core Concepts & The Golden Rule

Role-Based Access Control (RBAC) regulates access to resources based on the roles of individual users. It uses the `rbac.authorization.k8s.io` API group.

### The Authorization Policy

- **Additive (Allow-List):** Permissions are purely additive. There are **NO "Deny" rules** in RBAC. If a permission is not explicitly granted, it is forbidden.
    
- **Empty Verbs:** If you create a rule with `verbs: []`, it effectively grants nothing (Deny All).
    
- **The Equation:** `Subject + Role (Verbs + Resources) + Binding = Access`
    

---

## 2. The Scope Matrix (Role vs ClusterRole)

This is the most critical concept for determining where permissions apply.

|**Role Type (Source)**|**Binding Type (Connector)**|**Resulting Scope**|**Use Case**|
|---|---|---|---|
|**Role**|**RoleBinding**|**Namespace**|Standard local access (e.g., Read Pods in `dev`).|
|**ClusterRole**|**RoleBinding**|**Namespace**|**Templating Strategy:** Reuse a global role definition (like `view`) but limit its effect to a single namespace.|
|**ClusterRole**|**ClusterRoleBinding**|**Cluster-Wide**|**Super User:** Access across ALL namespaces + Cluster-scoped resources (Nodes).|
|**Role**|**ClusterRoleBinding**|**Invalid**|You cannot bind a namespaced Role globally.|

---

## 3. RBAC API Objects

### A. Roles (Namespaced)

Defines rules within a specific namespace.

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods", "pods/log"] # Can include subresources
  verbs: ["get", "watch", "list"]
```

> [!NOTE] RBAC Verbs: `get` vs `list`
> Understanding the exact API behavior of these two verbs is critical for least-privilege security and the CKA exam.
> 
> * **`get`:** Grants permission to retrieve a **single, specific resource by its exact name**. 
>     * *Example:* `kubectl get pod nginx-pod` will work.
>     * *Limitation:* The user cannot query the cluster to find out what pods exist.
> * **`list`:** Grants permission to retrieve a **collection of resources**, allowing the user to view all resources of that type without knowing their specific names.
>     * *Example:* `kubectl get pods` will work, returning the full list.
> 
> *Best Practice:* In most troubleshooting roles, users require both `["get", "list"]` combined. A third verb, `watch`, is often added `["get", "list", "watch"]` to allow the user to stream live updates (using the `-w` flag) without repeatedly querying the API.

### B. ClusterRoles (Non-Namespaced)

Used for:

1. Cluster-scoped resources (Nodes, PVs).
    
2. Non-resource endpoints (`/healthz`).
    
3. Defining common permissions (Templates) to be referenced by RoleBindings.
    

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

> [!NOTE] Modifying Roles
> - **Roles are Mutable:** You can add or remove permissions (verbs/resources) from a Role/ClusterRole at any time. The changes apply **immediately** to all bound users.

### C. RoleBinding

> [!IMPORTANT] RoleBinding Multiplicity Rule: Subjects vs. RoleRef
> A single `RoleBinding` (or `ClusterRoleBinding`) has a strict one-to-many relationship regarding roles and subjects:
> 
> * **Multiple Subjects (Many):** The `subjects` field is a list. You can bind multiple Users, Groups, and ServiceAccounts simultaneously within a single manifest.
> * **Single Role (One):** The `roleRef` field is a single object, not a list. A binding can ONLY reference **exactly one** `Role` or `ClusterRole`. 
> 
> *Architecture Note:* If you need to assign multiple different roles to the same user or group, you must create a separate `RoleBinding` object for each specific role.

Grants permissions defined in a Role (or ClusterRole) to a user within a specific namespace.

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-secrets
  namespace: development # Permissions apply ONLY here
subjects:
- kind: User
  name: dave
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole # Referencing a global role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

> [!IMPORTANT] Immutability
> You cannot change the `roleRef` of an existing binding. You must delete and recreate the object to change the Role it points to.

### D. ClusterRoleBinding

Grants permissions cluster-wide (all namespaces).

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

> [!INFO] Dynamic RBAC Updates (Role Modification)
> Permissions in Kubernetes are evaluated dynamically at runtime. A `RoleBinding` strictly acts as a pointer (`roleRef`) to a `Role`, it does not copy the rules. Therefore, if you update or modify a `Role` (e.g., adding or removing verbs like `delete`), those changes will instantaneously apply to all users, groups, or service accounts bound to that Role without requiring any modifications to the existing `RoleBinding` objects.

> [!INFO] Binding a ClusterRole 
> You can bind a `ClusterRole` to a `ServiceAccount` using **either** a `RoleBinding` or a `ClusterRoleBinding`. The choice strictly dictates the scope of the execution:
> 
> 1. **Using `RoleBinding` (Namespace Scope):** >    The ServiceAccount receives the permissions defined in the ClusterRole, but can **only** exercise them within the specific namespace where the RoleBinding is created. 
>    *Use Case:* Granting a deployment tool global 'edit' capabilities, but locking it to only modify resources within the `staging` namespace.
> 
> 2. **Using `ClusterRoleBinding` (Cluster Scope):**
>    The ServiceAccount receives the permissions across the **entire cluster** (all namespaces) and can also access cluster-scoped resources (like Nodes). 
>    *Use Case:* Granting a monitoring agent (like Prometheus) or an Ingress Controller the ability to 'list' and 'watch' pods across every namespace.

---

## 4. Advanced Resource Definitions

### API Groups

- **Core Resources (Pods, Services, Nodes):** `apiGroups: [""]`
    
- **Apps (Deployments, DaemonSets):** `apiGroups: ["apps"]`
    
- **Batch (Jobs):** `apiGroups: ["batch"]`
    

### Subresources

To allow access to logs or exec, append the subresource to the resource name.

- `resources: ["pods/log", "pods/exec"]`
    

### ResourceNames (Limiting to Specific Instances)

You can restrict access to a specific object by name.

YAML

```
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-configmap"] # Only this specific map
  verbs: ["update", "get"]
```

> [!CAUTION] Constraint
> You cannot restrict `create` or `deletecollection` calls by `resourceName` because the object name might not be known at authorization time.

### Wildcards

You can use `*` for resources, verbs, or API groups.

- `resources: ["*"]`
    
- `verbs: ["*"]`
    
- **Warning:** This is dangerous and grants overly permissive access.
    

---

## 5. Aggregated ClusterRoles (Automation)

Kubernetes allows combining multiple ClusterRoles into one using `aggregationRule`. The Controller Manager watches for these rules.

**Example:** Adding rules to the default `monitoring` role automatically.

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.example.com/aggregate-to-monitoring: "true"
rules: [] # Automatically filled by the controller
```

Any new ClusterRole created with the label `rbac.example.com/aggregate-to-monitoring: "true"` will have its rules automatically added to the `monitoring` role.

---

## 6. Built-in Default Roles

Kubernetes ships with default ClusterRoles that satisfy most common needs. This supports the **DRY (Don't Repeat Yourself)** principle.

### Key Default Roles

1. **`view`**: Read-only access to most objects. **Excludes Secrets** and Roles/Bindings.
    
2. **`edit`**: Read/Write access to most objects. **Includes Secrets**. Cannot modify Roles/Bindings.
    
3. **`admin`**: Full control within a namespace. Can create Roles/Bindings within that namespace.
    
4. **`cluster-admin`**: Super-user. Full control over the entire cluster.
    

### Example: Using a Built-in Role (Templating)

**Scenario:** You want to give user `jane` read-only access (view) to the `development` namespace only. You do **not** need to create a Role. You simply bind the existing `view` ClusterRole.

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-view-access
  namespace: development   # 1. Scope limited to this namespace
subjects:
- kind: User
  name: jane               # 2. Subject is Jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole        # 3. Source is a Global Role
  name: view               # 4. Using the built-in 'view' template
  apiGroup: rbac.authorization.k8s.io
```

---

## 7. Service Accounts & Subjects

### Subject Formatting

- **User:** `name: "alice@example.com"`
    
- **Group:** `name: "frontend-admins"`
> [!IMPORTANT] Managing Groups in Kubernetes
> Kubernetes does not natively store Group or User objects. Group membership is extracted dynamically from the authentication method used. 
> 
> **How to implement Groups with X.509 Certificates:**
> 1. **Authentication:** When generating the CSR, define the group using the Organization (`O=`) field in the subject line (e.g., `openssl req -subj "/CN=user1/O=qa-team"`).
> 2. **Authorization:** Create a `RoleBinding` or `ClusterRoleBinding` and set the `subject.kind` to `Group`, and the `subject.name` to match the exact string used in the `O=` field (e.g., `qa-team`).
> 
> *Best Practice:* Always bind roles to Groups rather than individual Users. When a new person joins the team, you just issue them a certificate with the correct `O=` group, and they instantly inherit all the necessary permissions without you having to write any new RBAC YAML!
    
- **ServiceAccount:** `name: "default"`, `namespace: "kube-system"`
> [!IMPORTANT] Binding a ServiceAccount to a ClusterRole
> Even though a `ServiceAccount` is confined to a specific namespace, it **CAN** be granted cluster-wide permissions using a `ClusterRole`. The resulting scope depends entirely on the Binding used:
> 
> 1. **Via `RoleBinding` (Namespace Scope):** The ServiceAccount gets the ClusterRole's permissions, but they are restricted to the namespace where the RoleBinding exists.
> 2. **Via `ClusterRoleBinding` (Cluster-wide Scope):** The ServiceAccount gets global permissions across all namespaces. This is essential for cluster-level tools like Ingress Controllers, Promethues, or network plugins.
> 
> *YAML Syntax Trap:* When adding a `ServiceAccount` to the `subjects` of a `ClusterRoleBinding`, you **must explicitly declare its namespace**, because the ClusterRoleBinding itself does not belong to any namespace and wouldn't know where to find the ServiceAccount otherwise.
> ```yaml
> subjects:
> - kind: ServiceAccount
>   name: ingress-sa
>   namespace: ingress-nginx
> ```
    

### System Groups

- `system:serviceaccounts`: All service accounts in the cluster.
    
- `system:serviceaccounts:<namespace>`: All service accounts in a specific namespace.
    
- `system:authenticated`: All authenticated users.
    
- `system:unauthenticated`: Anonymous users.

> [!IMPORTANT] Using Built-in System Groups in RBAC
> Kubernetes provides dynamic, built-in system groups that automatically aggregate users or service accounts. The most critical trap when using these in a `RoleBinding` or `ClusterRoleBinding` is that **the `kind` MUST be set to `Group`**, even if the group name contains the word "serviceaccounts".
> 
> **Common System Groups & Syntax:**
> 
> * `system:serviceaccounts`: Targets every single service account across the entire cluster.
> * `system:serviceaccounts:<namespace>`: Targets all service accounts currently existing within a specific namespace (e.g., `system:serviceaccounts:kube-system`).
> * `system:authenticated`: Targets any user or service account that has successfully passed the authentication step.
> * `system:unauthenticated`: Targets unauthenticated (anonymous) requests.
> 
> **YAML Implementation Example:**
> ```yaml
> subjects:
> - kind: Group
>   name: system:serviceaccounts:frontend  # Grants access to all SAs in the 'frontend' namespace
>   apiGroup: rbac.authorization.k8s.io
> - kind: Group
>   name: system:authenticated              # Grants access to any authenticated entity
>   apiGroup: rbac.authorization.k8s.io
> ```

---

## 8. Privilege Escalation Prevention

The RBAC API prevents users from creating permissions they do not possess themselves.

1. **Role Creation:** You can only create a Role if you already have all the permissions contained in that Role.
    
2. **Role Binding:** You can only bind a Role to a user if you possess the permissions in that Role.
    
3. **Exceptions:**
    
    - You have explicit permission to perform the `escalate` verb on roles.
        
    - You have explicit permission to perform the `bind` verb on roles.
        

---

## 9. Imperative Commands (Cheat Sheet)

**Create Role**

Bash

```
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create role specific-reader --verb=get --resource=pods --resource-name=mypod
```

**Create ClusterRole**

Bash

```
kubectl create clusterrole node-admin --verb=get,list,watch --resource=nodes
```

**Create RoleBinding (The Template Strategy)**

Bash

```
# Binds the global 'view' role to user 'jane' only in namespace 'acme'
kubectl create rolebinding jane-view --clusterrole=view --user=jane --namespace=acme
```

**Create ClusterRoleBinding (Global Access)**

Bash

```
# Grants 'root' user full cluster-admin access
kubectl create clusterrolebinding root-admin --clusterrole=cluster-admin --user=root
```

**Verification (Can-I)**

Bash

```
# Check my permissions
kubectl auth can-i create deployments

# Check someone else's permissions (Impersonation)
kubectl auth can-i list secrets --as=dave --namespace=dev
```

**Reconcile (Manifest Management)**

Bash

```
# Updates RBAC objects from a file, adding missing permissions/subjects
kubectl auth reconcile -f my-rbac-rules.yaml
```