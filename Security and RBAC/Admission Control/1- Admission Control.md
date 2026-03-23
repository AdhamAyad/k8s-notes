# Kubernetes Architecture: Admission Controllers

**Tags:** #kubernetes #architecture #admission-controllers #security #cka

**Status:** Core Concept

## 1. Core Concept: What is an Admission Controller?

When you send a request to the Kubernetes API server (e.g., using `kubectl`), the request must pass through multiple security checkpoints before it is persisted into the `etcd` database.

1. **Authentication:** Verifies _who_ you are (via certificates, tokens).
    
2. **Authorization:** Verifies _what_ you can do (via RBAC, like "can this user create pods?").
    
3. **Admission Control:** Intercepts the request to inspect, modify, or reject the _contents_ of the object being created or modified.
    

Admission controllers apply **only** to requests that create, delete, modify, or connect (proxy). They **do not** (and cannot) block read requests (`get`, `watch`, `list`) because reads completely bypass the admission control layer.

## 2. Why RBAC is Not Enough

RBAC operates strictly at the API endpoint level. It can say: _"Developer A is allowed to create Pods."_ However, RBAC **cannot** look inside the Pod's YAML configuration. You need Admission Controllers to enforce deeper, object-level policies, such as:

- Preventing the Pod from running as the `root` user.
    
- Preventing the use of the `latest` image tag.
    
- Ensuring the Pod only pulls images from a private, approved corporate registry.
    

## 3. The Two Phases of Admission Control

The admission control process operates in two distinct, sequential phases:

### Phase 1: Mutating Admission

Mutating controllers are executed first. They have the power to actually modify (mutate) the incoming request payload before it is saved.

- _Example:_ If a user creates a PersistentVolumeClaim (PVC) without specifying a StorageClass, the `DefaultStorageClass` mutating controller intercepts the request and injects the default storage class into the YAML.
    

### Phase 2: Validating Admission

Validating controllers are executed second. They cannot change the object; they can only inspect it and return a strict `Yes` (Approve) or `No` (Deny).

- _Example:_ The `NamespaceExists` validating controller checks if the target namespace exists. If it does not, it rejects the request outright.
    

> [!IMPORTANT] The Rejection Rule
> 
> If _any_ admission controller in _either_ the Mutating or Validating phase rejects the request, the entire request is rejected immediately, and an error is returned to the user. The object is never saved to `etcd`.

## 4. Enabling and Disabling Admission Plugins

Admission controllers are compiled directly into the `kube-apiserver` binary. They are controlled entirely by the cluster administrator using startup flags.

To see which plugins are enabled by default in your cluster, run:

Bash

```
kube-apiserver -h | grep enable-admission-plugins
```

### Modifying the API Server Manifest (kubeadm)

In a standard `kubeadm` setup, the API server runs as a Static Pod. To add or remove admission controllers, you must edit its manifest file.

Bash

```
vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

**YAML Configuration:**

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    # 1. Enable specific plugins (comma-separated)
    - --enable-admission-plugins=NodeRestriction,AlwaysPullImages
    # 2. Disable specific plugins (even if they are enabled by default)
    - --disable-admission-plugins=DefaultStorageClass
```

> [!WARNING] Core Cluster Functionality
> 
> Several crucial Kubernetes features absolutely require their corresponding admission controllers to be enabled (e.g., `LimitRanger`, `ResourceQuota`). If you improperly configure or disable the default set of admission controllers, your cluster will be incomplete and standard features will silently fail to work.

## 5. Notable Built-in Admission Controllers

Here are the most critical built-in admission controllers you need to know:

- **NamespaceLifecycle:** Prevents the creation of new objects in a Namespace that is currently undergoing termination. It also strictly prevents the deletion of core system namespaces (`default`, `kube-system`, `kube-public`).
    
- **NodeRestriction:** A critical security plugin. It limits the `kubelet` on a worker node so it can only modify its _own_ Node object, and only modify Pod objects that are physically bound to that specific node.
    
- **AlwaysPullImages:** Modifies every new Pod to force the `imagePullPolicy` to `Always`. Essential for multi-tenant clusters to ensure users must always authenticate against the container registry, preventing them from secretly reusing cached images pulled by other users.
    
- **LimitRanger & ResourceQuota:** Observes incoming requests and rejects them if they violate the constraints defined in `LimitRange` or `ResourceQuota` objects in that namespace.
    
- **DefaultStorageClass / DefaultIngressClass:** Automatically mutates PVCs or Ingresses that lack a specified class, injecting the cluster's default class into the specification.
    

> [!INFO] Deprecated Namespace Controllers
> 
> Previously, Kubernetes used `NamespaceExists` and `NamespaceAutoProvision` to manage namespace logic. These are now largely deprecated and replaced by the more robust `NamespaceLifecycle` controller.

## 6. Extending Admission Control (Webhooks)

If the built-in plugins do not cover your specific business logic, Kubernetes provides three special extension points to integrate custom admission logic:

1. **MutatingAdmissionWebhook:** Calls external HTTP webhooks that can mutate the object.
    
2. **ValidatingAdmissionWebhook:** Calls external HTTP webhooks (in parallel) to validate the object.
    
3. **ValidatingAdmissionPolicy:** (Newer feature) Allows administrators to embed declarative validation code directly into the API using CEL (Common Expression Language), removing the need to host and maintain external webhook servers.