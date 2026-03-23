# Kubernetes Objects: Secrets

**Tags:** #kubernetes #secrets #security #devops #CKA #configuration #volumes

**Status:** Core Concept

---

## 1. Overview & Core Concept

A **Secret** is an object designed to store sensitive data (Passwords, OAuth Tokens, SSH Keys). It creates a layer of abstraction that decouples "Confidential Data" from "Application Code".

### Why not use ConfigMaps?

While they look similar, Secrets have specific security features:

1. **Obfuscation:** Data is Base64 encoded (by default).
    
2. **Volatile Storage:** When mounted as a volume, Secrets are stored in **RAM (`tmpfs`)**, never written to the Node's disk (to prevent data theft from physical drives).
    
3. **Access Control:** Can be restricted via RBAC independently of **[[ConfigMaps]]**.
    

> [!CAUTION] The "Encoding" Trap
> 
> Kubernetes Secrets are **NOT Encrypted** by default; they are only **Base64 Encoded**.
> 
> - **Encoding:** Converting data format (Reversible by anyone).
>     
> - **Encryption:** Scrambling data using a key.
>     
> 
> _Anyone with API access to the namespace (or `etcd`) can decode your secrets._
> 
> **Best Practice:** Enable **Encryption at Rest** in your cluster configuration.

---

## 2. Types of Secrets

Kubernetes uses the `type` field to validate the structure of the secret.

|**Type**|**Name**|**Usage Scenario**|
|---|---|---|
|**Opaque**|`Opaque` (Default)|Arbitrary user-defined data (DB Passwords, API Keys).|
|**TLS**|`kubernetes.io/tls`|HTTPS Certificates. Requires keys: `tls.key`, `tls.crt`.|
|**Docker**|`kubernetes.io/dockerconfigjson`|Credentials to pull images from Private Registries.|
|**Service Account**|`kubernetes.io/service-account-token`|**Legacy.** Token for Pod to talk to API Server.|
|**Basic Auth**|`kubernetes.io/basic-auth`|Username/Password pair.|

---

## 3. Creating Secrets (Manifests & Commands)

### A. The Declarative Way (YAML)

You can use `data` (Base64) or `stringData` (Plain Text - easier for devs).

YAML

```
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: backend
type: Opaque
# Option 1: stringData (Write plain text, K8s encodes it for you)
stringData:
  db_host: "postgres-prod"
  db_pass: "P@ssw0rd123"

# Option 2: data (You must provide Base64 strings manually)
# echo -n "root" | base64 -> cm9vdA==
data:
  root_user: cm9vdA==
```

> [!TIP] `stringData` vs `data`
> 
> Always use `stringData` when writing manifests manually to avoid encoding errors. Kubernetes converts it to `data` (Base64) immediately upon creation.

### B. The Imperative Way (Kubectl Subcommands)

Fastest way to create secrets without writing YAML.

**1. Generic (Opaque) Secret:**

Bash

```
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=123456
```

**2. TLS Secret (For HTTPS):**

Bash

```
kubectl create secret tls my-website-tls \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

**3. Docker Registry Secret:**

Bash

```
kubectl create secret docker-registry my-reg-cred \
  --docker-server=my.registry.com \
  --docker-username=user \
  --docker-password=password \
  --docker-email=user@example.com
```

---

## 4. How to Consume Secrets (Usage Scenarios)

There are 3 main ways to use a secret inside a Pod.

### Method A: Environment Variables

**Best For:** Simple values (DB URL, Passwords, API Tokens).

- **Update Behavior:**  **No Live Updates.** Requires Pod Restart.
    

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: backend-app
spec:
  containers:
  - name: app
    image: my-app
    env:
      # Inject a specific key
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: db_pass
```

### Method B: Volume Mounts

**Best For:** Complex files (Certificates, JSON configs) or Rotation.

- **Security:** Stored in **RAM (`tmpfs`)**.
    
- **Update Behavior:**  **Live Updates.** (Updates automatically after ~1 minute).
    

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-https
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: tls-certs
      mountPath: "/etc/nginx/ssl"
      readOnly: true  # Recommended for security
  volumes:
  - name: tls-certs
    secret:
      secretName: my-website-tls
```

### Method C: ImagePullSecrets (Private Registry)

Used by the **Kubelet** (not the Pod's application) to authenticate with a registry like Docker Hub or GCR.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:
  - name: my-reg-cred
  containers:
  - name: main
    image: my-private-registry.com/app:v1
    # envFrom: 
      # secretRef:
      #   name: my-secret
      # prefix: PROD_ # Add this befor every variable
        
```

---

## 5.  Critical: Volume Mounts & The "Overwrite" Behavior

This is the most common pitfall when mounting secrets.

### Scenario A: The "Directory Overwrite" (Default)

If you mount a Secret to a directory that already exists in the image (e.g., `/etc/nginx/conf.d`), **Kubernetes will hide/remove all existing files** in that directory and show _only_ the Secret files.

YAML

```
volumeMounts:
- name: my-config
  mountPath: /etc/nginx/conf.d  #  DANGER: Deletes existing configs!
```

### Scenario B: The `subPath` Solution (Single File Injection)

To inject a _single file_ without destroying the directory, use `subPath`. This places the file alongside existing files.

- **Trade-off:** Using `subPath` **disables Live Updates**. If you change the Secret, the file inside the Pod will NOT change until restart.
    

YAML

```
volumeMounts:
- name: my-config
  mountPath: /etc/nginx/conf.d/secret.conf  # 1. Path to the FILE
  subPath: secret.conf                       # 2. Key name in Secret
```

---

## 6. Advanced Scenarios & Tricks

### Scenario 1: The "Dotfile" Trick (Hidden Files)

If you want to mount a hidden file (e.g., `.dockercfg`), name the key starting with a dot.

YAML

```
data:
  .secret-file: dmFsdWUtMg0KDQo=  # Base64 content
```

_When mounted:_ It appears as a hidden file. Use `ls -la` to see it.

### Scenario 2: Immutable Secrets (Performance)

If you know your secret won't change, mark it as `immutable`.

- **Benefit:** Kubelet stops "watching" it -> Less load on API Server.
    
- **Risk:** You must delete and recreate it to change data.
    

YAML

```
apiVersion: v1
kind: Secret
metadata:
  name: static-token
immutable: true
data: ...
```

---

## 7. Security & Best Practices Checklist 

1. **Least Privilege:** Use separate Namespaces. Don't put all secrets in `default`.
    
2. **Encryption at Rest:** Ensure `etcd` is encrypted so admins with disk access can't read secrets.
    
3. **GitOps Safety:** **NEVER** commit raw Secret YAMLs to Git.
    
    - _Solution:_ Use tools like **Sealed Secrets**, **External Secrets Operator**, or **HashiCorp Vault**.
        
4. **Container Access:** Only mount the secret to the specific container that needs it (not all containers in the Pod).
    
5. **Environment Variables Risk:** Avoid using Env Vars for highly sensitive data if possible, because they can appear in logs or `kubectl describe pod`. **Volume Mounts are slightly safer.**