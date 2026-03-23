# Image Security & Private Registries

**Tags:** #kubernetes #security #containers #docker-registry #cka

**Status:** Core Deployment Operations

---
## 1. Container Image Naming Conventions

Before securing the images, you must understand how container runtimes (like Docker or containerd) resolve image names. If you do not specify a full path, Kubernetes assumes the default public registry.

- **Implicit Default (Public):** If your Pod YAML says `image: nginx`, the system translates this to `docker.io/library/nginx:latest`.
    
- **Custom Account (Public):** If you use `image: your-account/nginx`, it translates to `docker.io/your-account/nginx:latest`.
    
- **Private Registry (Explicit):** When using a private registry (like AWS ECR, GCP GCR, or an internal corporate artifact), you **must** provide the fully qualified path.
    
    - _Example:_ `image: private-registry.io/apps/internal-app:v1.2`
        

## 2. The Private Registry Authentication Flow

When a Kubernetes worker node attempts to deploy a Pod with an image hosted on a private registry, the internal container runtime (Kubelet/containerd) will fail with an `ErrImagePull` or `ImagePullBackOff` error because it lacks the necessary login credentials.

To fix this, you must explicitly provide Kubernetes with the registry credentials. This is a strict two-step process:

### Step 1: Create a Docker-Registry Secret

You must store your private registry credentials inside a specific type of Kubernetes Secret called `docker-registry`.

The most efficient way to generate this is via the imperative command:

Bash

```
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=registry-user \
  --docker-password=registry-password \
  --docker-email=registry-user@org.com
```

_(In this example, the secret is named `regcred`)._

> [!WARNING] Secret Scope
> 
> Remember that Kubernetes Secrets are **namespace-bound**. The `docker-registry` secret must exist in the exact same namespace where you intend to deploy your Pods. If you deploy a Pod in the `dev` namespace, the secret must also be created in the `dev` namespace.

### Step 2: Bind the Secret to the Pod Specification

Once the secret exists, you must instruct the Pod to use it. This is done by adding the `imagePullSecrets` array directly under the Pod's `spec` section.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: internal-nginx-pod
spec:
  containers:
    - name: nginx
      image: private-registry.io/apps/internal-app:v1.2
  imagePullSecrets:
    - name: regcred    # This must match the name of the secret created in Step 1
```

> [!NOTE] Architecture Context: How it works under the hood
> 
> When the Pod is scheduled, the Kubelet on the assigned worker node reads the `imagePullSecrets` directive. It extracts the authentication token from the specified Kubernetes Secret and passes it to the container runtime (e.g., containerd or Docker). The runtime then performs a secure login to the private registry, downloads the image, and starts the container.

## 3. Advanced Trick: Attaching to a Service Account

> [!TIP] Automating Image Pull Secrets (Best Practice)
> 
> If you have a namespace with 50 different microservices, writing `imagePullSecrets` inside every single Pod YAML violates the DRY (Don't Repeat Yourself) principle.
> 
> **The Solution:** You can attach the `imagePullSecrets` directly to a `ServiceAccount`.
> 
> ```yaml
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: default
>   namespace: dev
> imagePullSecrets:
> - name: regcred
> ```
> 
> If you add the secret to the `default` Service Account of a namespace, every single Pod created in that namespace will automatically inherit the ability to pull images from that private registry without needing to modify the Pod YAML!

