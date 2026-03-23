# Security Contexts in Kubernetes

**Tags:** #kubernetes #security #cka #containers #linux-capabilities

**Status:** Core Security Operations

---

## 1. Architectural Overview

By default, container runtimes (like Docker or containerd) often run applications as the `root` user. In a production Kubernetes environment, this is a major security vulnerability.

**Security Contexts** allow you to restrict the privileges of a Pod or Container. They define privilege and access control settings, such as which specific user ID (UID) the container should run as, or what specific Linux Capabilities it is granted or denied.

_Docker CLI Equivalent:_

- User Definition: `docker run --user=1001 ubuntu sleep 3600`
    
- Capability Addition: `docker run --cap-add MAC_ADMIN ubuntu`
    

## 2. The Two Levels of Application

In Kubernetes, you can apply a `securityContext` at two distinct levels within your YAML manifest. Understanding the difference is critical for cluster administration and the CKA exam.

### A. Pod-Level Security Context

Defined directly under `spec`. Any setting applied here becomes the default for **all** containers running inside that specific Pod.

### B. Container-Level Security Context

Defined under `spec.containers[]`. Any setting applied here strictly targets that individual container. _Note: Linux Capabilities can ONLY be added or dropped at the container level, not the pod level._

---

## 3. The Precedence Rule (Crucial Architecture Rule)

> [!IMPORTANT] The Override Rule (Pod vs. Container Scope)
> 
> When you configure security settings in Kubernetes, conflicts can arise if you define parameters at both the Pod and Container levels simultaneously.
> 
> **The Rule of Precedence:** > If the same configuration parameter (e.g., `runAsUser`) is specified at both the Pod level and the Container level, the **Container-level setting takes strict precedence**. The container's specific setting will override the global Pod setting.

---

## 4. Implementation (YAML Examples)

Below is a comprehensive YAML manifest demonstrating both the implementation of a Security Context and the Precedence Rule in action.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: multi-tier-security-pod
spec:
  securityContext:
    runAsUser: 2000                  # POD-LEVEL: Sets the default user to 2000 for the whole pod
  containers:
    - name: secure-ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        runAsUser: 1000              # CONTAINER-LEVEL: OVERRIDES the Pod level. This container runs as user 1000.
        capabilities:                # Capabilities are strictly a Container-level property
          add: ["MAC_ADMIN"]
    
    - name: secondary-ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      # No container-level security context is defined here.
      # INHERITANCE: This container will run as user 2000, inheriting the Pod-level setting.
```

> [!NOTE] Verifying Security Contexts
> 
> To verify that your configurations have been applied successfully, you can execute a command directly inside the running pod.
> 
> Run: `kubectl exec -it <pod-name> -- id`
> 
> This will output the current User ID (UID) and Group ID (GID) the container is actively utilizing.