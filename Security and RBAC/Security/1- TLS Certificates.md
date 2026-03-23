# Comprehensive Guide: TLS & Certificates in Kubernetes

**Tags:** #kubernetes #tls #security #certificates #cka #configmap
**Status:** Core Security Architecture & Operations

---

## 1. TLS Basics: Symmetric vs. Asymmetric Encryption

To understand Kubernetes security, you must first understand the underlying encryption mechanisms.

* **Symmetric Encryption:** Uses a single key to encrypt and decrypt data. 
  * *Advantage:* Fast and efficient. 
  * *Disadvantage:* Highly insecure to transmit the key over the network.
* **Asymmetric Encryption:** Uses a pair of keys (Public Key and Private Key).
  * **Public Key:** Shared openly. Used only to encrypt data.
  * **Private Key:** Kept securely by the owner. Used to decrypt data encrypted by the Public Key.
* **TLS / HTTPS:** Combines both. It uses Asymmetric encryption initially to securely exchange a Symmetric key. Once exchanged, the faster Symmetric key is used for the rest of the communication session.

> [!INFO] File Naming Conventions
> * **Public Keys / Certificates:** Usually have `.crt` or `.pem` extensions (e.g., `server.crt`, `server.pem`).
> * **Private Keys:** Usually include the word "key" in the filename or extension (e.g., `server.key`, `server-key.pem`).

---

## 2. Certificate Authority (CA) & Component Grouping

All certificates in the cluster must be signed by a trusted Certificate Authority (CA). The CA has its own Private Key (`ca.key`) and Public Root Certificate (`ca.crt`). 

Certificates in Kubernetes are divided into two main categories:

### Server Certificates (To secure their own services)
These components serve requests and must prove their identity to clients:
1. **Kube API Server:** `apiserver.crt` / `apiserver.key`
2. **ETCD Server:** `etcd-server.crt` / `etcd-server.key`
3. **Kubelet (on Worker & Master Nodes):** `kubelet.crt` / `kubelet.key` (Used to expose the HTTPS endpoint for the API Server to connect to).

### Client Certificates (To authenticate against the API Server)
These components act as clients requesting services from the API Server:
1. **Administrator (kubectl):** `admin.crt` / `admin.key`
2. **Kube Scheduler:** `scheduler.crt` / `scheduler.key`
3. **Kube Controller Manager:** `controller-manager.crt` / `controller-manager.key`
4. **Kube Proxy:** `kube-proxy.crt` / `kube-proxy.key`
5. **Kube API Server (as a client):** Needs client certificates when talking to ETCD or Kubelets.
6. **Kubelet (as a client):** Needs client certificates to authenticate with the API Server.

---

## 3. Creating Certificates via OpenSSL

The creation process always follows a strict 3-step workflow.

### Step 1: Generating the Certificate Authority (CA)
The CA is the supreme authority. It signs its own certificate (Self-Signed).
```bash
# 1. Generate Private Key
openssl genrsa -out ca.key 2048

# 2. Create Certificate Signing Request (CSR)
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# 3. Sign the CSR with its own Key to create the Public Certificate
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
````

### Step 2: Generating Client/Server Certificates (e.g., Admin)

All other components follow the exact same process, but the final step requires the CA's key and certificate to sign them.

Bash

```
# 1. Generate Private Key
openssl genrsa -out admin.key 2048

# 2. Create Certificate Signing Request (CSR) specifying Group (O=system:masters)
openssl req -new -key admin.key -subj "/CN=kube-admin/O=system:masters" -out admin.csr

# 3. Sign the CSR using the CA Certificate and CA Key
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -out admin.crt
```

> [!WARNING] API Server SANs (Subject Alternative Names)
> 
> The API Server certificate is special. Because it can be reached via multiple names (kubernetes, kubernetes.default, its IP address, etc.), you must create an `openssl.cnf` file listing all these DNS names and IPs under the `[alt_names]` section before generating its certificate.

---

## 4. Viewing and Troubleshooting Certificates

If the cluster is deployed using `kubeadm`, certificates are automatically generated and stored in specific directories.

### Important Default Paths

- **Certificates Directory:** `/etc/kubernetes/pki/`
    
- **Static Pod Manifests (To check cert configurations):** `/etc/kubernetes/manifests/`
    

### Inspecting Certificate Details

To read the details inside a certificate (Validity, Issuer, Subject Alternative Names), use the following OpenSSL command:

Bash

```
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

### Troubleshooting Steps

If you suspect a certificate issue (e.g., "tls: bad certificate"):

1. Check the Pod specifications in `/etc/kubernetes/manifests/` to ensure the file paths to the certificates are correct.
    
2. Decode the certificate using the `openssl x509 -text` command to verify expiration dates and Issuer details.
    
3. Check the logs:
    
    - **Native Services:** `journalctl -u etcd.service -l`
        
    - **Containerized Components (kubeadm):** Use container runtime logs. Find the container ID via `crictl ps` (or `docker ps`) and check logs via `crictl logs <container-id>`.


