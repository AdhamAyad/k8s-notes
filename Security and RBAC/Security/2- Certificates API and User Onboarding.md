# Comprehensive Guide: Kubernetes Certificates API & User Onboarding

**Tags:** #kubernetes #security #tls #certificates-api #rbac #cka 
**Status:** Core Security Operations Guide

---

## 1. Architectural Overview: Authentication vs. Authorization

Before executing any commands, it is crucial to understand the separation of concerns in Kubernetes security. This directly answers your question: *"Where does the user get permissions from?"*

> [!IMPORTANT] The Golden Rule of Kubernetes Security
> **1. Authentication (Who are you?):** This is handled entirely by the **TLS Certificate**. When Jane presents a certificate signed by the cluster's Certificate Authority (CA), the API Server says: *"I trust this signature, you are officially Jane."*
> **2. Authorization (What can you do?):** The certificate gives **ZERO** permissions. Even with a valid certificate, Jane cannot list a single pod. Permissions are granted in a completely separate step using **RBAC (Role-Based Access Control)**.



---

## 2. Phase 1: The User's Request (On Jane's Local Machine)

Jane needs access to the cluster. She must generate her cryptographic keys locally.

**Step 1: Generate the Private Key**
```bash
openssl genrsa -out jane.key 2048
````

- **Background:** This creates the mathematical secret key. It stays on Jane's machine and is never shared over the network.
    

**Step 2: Generate the Certificate Signing Request (CSR)**

Bash

```
openssl req -new -key jane.key -subj "/CN=jane/O=developers" -out jane.csr
```

- **Background:** This command mathematically derives a **Public Key** from `jane.key`. It packages this Public Key, along with her identity (`CN=jane` as the username, `O=developers` as the group), into a "request envelope" called `jane.csr`.
    
- **Next Action:** Jane sends `jane.csr` (the envelope) to the Cluster Administrator.
    

---

## 3. Phase 2: The Administrator's Job (On the Control Plane)

You, as the Administrator, receive `jane.csr`. Your job is to introduce this request to the Kubernetes API and approve it.

**Step 1: Encode the CSR to Base64**

Kubernetes manifests require binary data to be encoded as a single string.

Bash

```
cat jane.csr | base64 | tr -d "\n"
```

_Copy the output string. You will need it in the next step._

**Step 2: Create the CertificateSigningRequest Object**

Create a manifest file named `jane-csr.yaml`.

YAML

```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-request
spec:
  request: <PASTE_THE_BASE64_STRING_HERE>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # Certificate valid for 1 day
  usages:
  - client auth
```

Submit the request to the cluster:

Bash

```
kubectl apply -f jane-csr.yaml
```

**Step 3: Review and Approve the Request**

Check the status of pending requests:

Bash

```
kubectl get csr
```

_(You will see `jane-request` in a `Pending` state)._

Approve the request:

Bash

```
kubectl certificate approve jane-request
```

> [!NOTE] What happens in the background during approval?
> 
> When you hit approve, the **Controller Manager** wakes up. It opens the CSR, extracts Jane's Public Key, reads the cluster's highly secure `ca.key` and `ca.crt` (from `/etc/kubernetes/pki/`), and physically signs Jane's Public Key. The resulting signed certificate is then saved back into the `CertificateSigningRequest` database object in a base64 encoded format.

**Step 4: Extract the Signed Certificate**

Now you must extract the approved certificate from the API and decode it to send back to Jane.

Bash

```
kubectl get csr jane-request -o jsonpath='{.status.certificate}' | base64 --decode > jane.crt
```

- **Next Action:** You send the newly generated `jane.crt` file back to Jane securely.
    

---

## 4. Phase 3: User Configuration (On Jane's Local Machine)

Jane now has her original `jane.key` and the newly signed `jane.crt`. She needs to configure her `kubectl` tool to use them.

**Step 1: Embed Credentials into Kubeconfig**

Bash

```
kubectl config set-credentials jane \
  --client-key=jane.key \
  --client-certificate=jane.crt
```

**Step 2: Create a Context for Jane**

A context links a User to a Cluster.

Bash

```
kubectl config set-context jane-context \
  --cluster=kubernetes \
  --user=jane
```

**Step 3: Switch to the New Context**

Bash

```
kubectl config use-context jane-context
```

> [!WARNING] The Access Denial
> 
> If Jane types `kubectl get pods` right now, she will get an error:
> 
> `Error from server (Forbidden): pods is forbidden: User "jane" cannot list resource "pods" in API group ""`
> 
> **Why?** Because she is _Authenticated_ (the API knows she is Jane), but she is not _Authorized_ (she has no permissions).

---

## 5. Phase 4: Granting Permissions (The RBAC Link)

To give Jane permissions, the Administrator must switch back to the admin context and create RBAC resources. This is a two-step process:

**1. Create a Role (What can be done?)**

This defines the exact actions allowed.

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

**2. Create a RoleBinding (Who can do it?)**

This binds the `pod-reader` Role to the user `jane`. Note how the name matches the `/CN=jane` from her original CSR.

YAML

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Apply these files as an Administrator.

**The Final Result:**

Now, when Jane (using her `jane-context`) runs:

Bash

```
kubectl get pods
```

The API Server validates her `jane.crt` (Authentication: PASS), checks the `RoleBinding` for the user "jane" (Authorization: PASS), and successfully returns the list of pods.

---

## 6. Phase 5: User Lifecycle Management (Renewal & Revocation)

Managing a user's access over time is a critical operational task. You must know how to properly revoke access or renew it when a certificate expires.

### 1. Revoking Access (Offboarding a User)

> [!TIP] Security Best Practice: Short-Lived Certificates
> Because Kubernetes lacks a straightforward mechanism to revoke client certificates (like a Certificate Revocation List - CRL), a compromised or lost `kubeconfig` file poses a significant security risk. The industry best practice is to always define a short expiration period for every certificate you issue. You enforce this by explicitly setting the `expirationSeconds` field in the `CertificateSigningRequest` manifest (e.g., `expirationSeconds: 86400` for 24 hours). This ensures that if a user leaves the organization or loses their machine, their access automatically expires quickly, minimizing the attack window without requiring manual intervention.

> [!WARNING] The CSR Deletion Trap
> Deleting the `CertificateSigningRequest` (CSR) object from the cluster using `kubectl delete csr <name>` **DOES NOT** revoke the user's access. As long as the user holds a valid, unexpired certificate signed by the CA, the API server will authenticate them. Kubernetes does not have a simple built-in Certificate Revocation List (CRL) mechanism for client certificates.

**The Correct Way to Revoke Access:**
Since you cannot easily invalidate the certificate itself, you must remove the user's permissions. You do this by "destroying the bridge" — deleting the `RoleBinding` (or `ClusterRoleBinding`) associated with that user.

```bash
kubectl delete rolebinding read-pods-binding -n default
````

_Result:_ The user (Jane) will still authenticate successfully to the API server, but every command she runs will return a `403 Forbidden` error because she no longer has authorization to perform any actions.

### 2. Renewing an Expired Certificate

When Jane's certificate reaches its expiration date, the API server will immediately reject her requests (Authentication Fails). In TLS, certificates cannot be extended; a completely new one must be generated.

**The Renewal Process:**

1. Jane generates a new CSR locally. She can use her existing private key or generate a new one for better security. She must ensure she uses the exact same Common Name (`/CN=jane`).
    
2. The Administrator creates a new `CertificateSigningRequest` manifest, applies it, and approves it.
> [!INFO] CSR Naming & Cleanup During Renewal
> When renewing a certificate, you do not strictly need to delete the old `CertificateSigningRequest` object **if** you give the new request a different name (e.g., `jane-request-v2`). The old object will remain as an audit log. However, if you want to reuse the exact same name for the new CSR manifest, you must delete the existing object first using `kubectl delete csr <name>`, otherwise the API server will reject the new manifest with an `AlreadyExists` error.
    
3. The Administrator extracts the new `.crt` file and sends it to Jane.
    
4. Jane updates her `kubeconfig` file with the new certificate data.
    

> [!TIP] The RBAC Magic for Renewals
> 
> You **do not** need to recreate or modify the `Role` or `RoleBinding` during a certificate renewal. Because Kubernetes maps permissions to the username (the `CN` extracted from the certificate), as soon as Jane authenticates with her new certificate containing `/CN=jane`, the cluster automatically applies her existing permissions.