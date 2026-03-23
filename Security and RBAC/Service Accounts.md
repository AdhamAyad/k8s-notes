# Comprehensive Guide: Kubernetes Service Accounts

**Tags:** #kubernetes #security #rbac #service-accounts #cka

**Status:** Core Security Operations Guide

---

## 1. Architectural Overview: Users vs. Service Accounts

In Kubernetes, identities are strictly divided into two categories based on who (or what) is interacting with the API Server.

- **User Accounts:** Designed for human administrators and developers. Kubernetes does not natively manage these (relies on external Identity Providers or x509 Certificates).
    
- **Service Accounts (SA):** Designed for **machines, applications, and processes** running inside Pods. Examples include monitoring tools (Prometheus), CI/CD pipelines (Jenkins), or a custom dashboard application querying the API to list pods. Kubernetes manages these natively within its own datastore.
    

---

## 2. Namespace Isolation & Default Behavior

> [!WARNING] The Hard Boundary Rule
> 
> Service Accounts are strictly **namespaced resources**. A Pod running in the `dev` namespace can ONLY mount and use a Service Account that exists within the `dev` namespace. Cross-namespace Service Account mounting is impossible.

Because of this strict requirement, the Kubernetes Control Plane automatically provisions a Service Account named `default` every time a new namespace is created.

If you create a Pod without explicitly specifying a Service Account, the Admission Controller automatically injects this `default` Service Account into your Pod.

---

## 3. The Token: Where is it and how does it work?

To communicate with the Kubernetes API, the application inside the Pod needs a Bearer Token. Kubernetes handles the delivery of this token dynamically.

> [!INFO] The Token Mount Path
> 
> Kubernetes automatically mounts the Service Account token into the running container as a file. Regardless of the environment, the token is always found at this exact path inside the container:
> 
> `/var/run/secrets/kubernetes.io/serviceaccount`

If you execute `ls` or `cat` on that directory inside a running Pod, you will typically find three files:

1. **`token`**: The actual JWT (JSON Web Token) used for Bearer authentication.
    
2. **`ca.crt`**: The certificate authority file to verify the API server's identity.
    
3. **`namespace`**: A text file containing the name of the namespace the Pod is running in.
    

---

## 4. Manipulating Pod Configurations

### A. Assigning a Custom Service Account

To assign a specific Service Account (e.g., `dashboard-sa`) instead of the default one, you must specify the `serviceAccountName` directly under the Pod's `spec`.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-dashboard
spec:
  serviceAccountName: dashboard-sa   # Must match the exact name of the SA
  containers:
  - name: dashboard
    image: my-dashboard-image
```

> [!DANGER] Immutability Trap
> 
> You **cannot** change the `serviceAccountName` of an already running Pod. To change the identity of an application, you must delete the Pod and recreate it (or update the Deployment, which forces a rolling restart).

### B. Disabling Token Auto-mounting

> [!WARNING] The Hidden Risks of the 'Default' Service Account
> A common misconception is that the `default` Service Account token is harmless because it lacks RBAC permissions. However, security best practices (and the CKS exam) strictly mandate disabling `automountServiceAccountToken` even for the default SA, due to the following risks:
> 
> 1. **Reconnaissance (Discovery APIs):** While unprivileged, the token provides valid **Authentication**. An attacker can use it to query unauthenticated discovery endpoints (e.g., `/api/v1`, `/version`) to map the cluster's API surface and identify the exact Kubernetes version to find known vulnerabilities.
> 2. **Configuration Drift (The "Lazy Admin" Risk):** If a cluster administrator mistakenly binds a `Role` (e.g., `view` or `edit`) to the `default` Service Account for convenience, **every** pod in that namespace automatically and silently inherits those permissions. Disabling the mount prevents pods from inheriting accidental privilege escalations.
> 3. **Defense in Depth:** Following the Zero Trust model, any credential that is not strictly required for the application's core function should be removed to minimize the attack surface.

If a Pod does not need to talk to the Kubernetes API (e.g., a simple static NGINX web server), it is a security best practice to prevent the token from being mounted inside the container to reduce the attack surface.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: static-web
spec:
  automountServiceAccountToken: false  # Disables the token volume mount
  containers:
  - name: web
    image: nginx
```

### C. Customizing the Token Mount Path

> [!TIP] Manual Token Projection (Custom Path)
> If your application explicitly expects the authentication token to be in a non-standard directory (e.g., `/opt/app/secrets`), you must override Kubernetes' default behavior. You do this by disabling the auto-mount feature and manually projecting the Service Account token into your desired path.
> 
> **How to do it (Modern v1.22+ approach):**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: custom-token-path-pod
> spec:
>   serviceAccountName: dashboard-sa
>   automountServiceAccountToken: false   # 1. Disable the default mount
>   containers:
>   - name: my-app
>     image: nginx
>     volumeMounts:
>     - name: custom-sa-token-vol
>       mountPath: /opt/app/secrets       # 2. Define your exact custom path here
>   volumes:
>   - name: custom-sa-token-vol
>     projected:                          # 3. Manually request the token via API
>       sources:
>       - serviceAccountToken:
>           path: token
>           expirationSeconds: 3600       # Token expires in 1 hour
> ```

---

## 5. The Architecture Shift: Kubernetes v1.22 & v1.24 (Crucial for CKA)

The way Kubernetes handles Service Account tokens changed drastically for security reasons. You must understand the difference between legacy and modern clusters.

### Legacy Behavior (Pre-v1.22)

- When a Service Account was created, Kubernetes automatically created a `Secret` object containing a non-expiring JWT token.
    
- This Secret was permanently linked to the Service Account.
    
- **Security Flaw:** The token never expired, had a broad attack surface, and could be stolen and used indefinitely.
    

### Modern Behavior (v1.22 and v1.24+)

- **v1.22:** The **TokenRequest API** was introduced. Tokens are now audience-bound, time-bound (they expire), and object-bound (tied to a specific Pod's lifespan). They are injected using `projected` volumes rather than standard Secrets.
    
- **v1.24+:** Kubernetes **stopped** automatically generating Secret-based tokens for Service Accounts. When you run `kubectl create serviceaccount my-sa`, no Secret is created.
    

> [!TIP] How to get a token in modern Kubernetes (v1.24+)
> 
> **Scenario A: For a Pod**
> 
> You do nothing. Simply assign the `serviceAccountName` in the Pod YAML. The kubelet uses the TokenRequest API to generate a temporary, secure token and mounts it dynamically when the container starts.
> 
> **Scenario B: For manual testing (e.g., curl from your laptop)**
> 
> You must generate a temporary token manually using the imperative command:
> 
> `kubectl create token <service-account-name>`
> 
> _(This outputs a JWT string that expires in 1 hour by default)._
> 
> **Scenario C: Creating a legacy, non-expiring token**
> 
> If you have a legacy CI/CD system that strictly requires a permanent token, you must manually create a Secret and annotate it to link it to the SA:
> 
> YAML
> 
> ```
> apiVersion: v1
> kind: Secret
> metadata:
>   name: legacy-permanent-token
>   annotations:
>     kubernetes.io/service-account.name: dashboard-sa
> type: kubernetes.io/service-account-token
> ```

> [!WARNING] Token Expiration and Kubelet Auto-Renewal (The Application Trap)
> When using the modern TokenRequest API (Projected Volumes), tokens have a strict expiration time. However, the Pod does **not** permanently lose access when this time is reached.
> 
> **1. The Kubelet's Role (Auto-Rotation):** The `kubelet` proactively tracks the token's lifetime. When the token reaches 80% of its total TTL, the kubelet automatically requests a new token from the API server and seamlessly overwrites the `token` file inside the running container's volume. The Pod is *never* restarted during this process.
> 
> **2. The Application's Role (The Catch):** While Kubernetes updates the *file* on the disk, the application running inside the container must be engineered to **re-read** the file dynamically. If the application reads the token into memory only once at startup and caches it forever, it will eventually attempt to use an expired token and receive a `401 Unauthorized` error. 
> *(Note: Official Kubernetes client libraries, like `client-go` or the Python SDK, handle this dynamic reloading automatically).*