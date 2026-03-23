# Kubernetes Security: Encrypting Confidential Data at Rest

**Tags:** #kubernetes #security #encryption #etcd #cka #manifests
**Status:** Advanced Implementation Guide

---

## 1. Core Concept: Encryption at Rest

By default, the Kubernetes API server stores resources (like Secrets and ConfigMaps) in the `etcd` database as plain text (Secrets are only Base64 encoded, which is not encryption). "Encryption at Rest" ensures that the API server encrypts this data *before* writing it to `etcd`, protecting it from infrastructure-level breaches (e.g., someone stealing the physical disks or gaining unauthorized access to the `etcd` database).



> [!IMPORTANT] Scope of Encryption
> This mechanism only encrypts data inside `etcd`. It does not encrypt data in transit, nor does it encrypt filesystems mounted into containers (Volumes). Once a Secret is retrieved by a user via `kubectl` or mounted into a Pod, it is decrypted and presented as standard Base64/Plaintext.

---

## 2. Encryption Providers

Kubernetes uses an `EncryptionConfiguration` file to define how data is encrypted. You must choose a "Provider" that dictates the encryption algorithm.

* **identity:** No encryption (Plain text). Used as a default or as a fallback during migrations.
* **aescbc:** AES-CBC with PKCS#7 padding. Fast, but considered weaker due to CBC vulnerabilities.
* **aesgcm:** AES-GCM. Fastest, but requires key rotation every 200,000 writes.
* **secretbox:** XSalsa20 and Poly1305. Strong and fast.
* **kms (v1/v2):** Key Management Service. The strongest option. Uses envelope encryption where a third-party service (like AWS KMS or Azure Key Vault) manages the master keys.

---

## 3. Step-by-Step Implementation Guide (Local Key Storage)

This guide uses a local static key (aescbc/secretbox) for demonstration.

### Step 1: Generate the Encryption Key
Generate a 32-byte random key and encode it in Base64.
```bash
head -c 32 /dev/urandom | base64
````

_(Save the output string, you will need it for the configuration file)._

### Step 2: Create the Encryption Configuration File

Create a file named `enc.yaml` on your control plane node (e.g., at `/etc/kubernetes/enc/enc.yaml`).

YAML

```
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <PASTE_YOUR_BASE64_KEY_HERE>
      - identity: {} # Fallback to allow reading existing unencrypted secrets
```

> [!INFO] Wildcard Support
> 
> If your cluster is running Kubernetes v1.27 or newer, you can use wildcards in the `resources` array, such as `'*.apps'` to encrypt all resources in the apps group, or `'*.*'` to encrypt absolutely everything.

### Step 3: Mount the Config to the API Server

You must modify the static Pod manifest for the `kube-apiserver` (usually located at `/etc/kubernetes/manifests/kube-apiserver.yaml`).

**1. Add the command line flag:**

YAML

```
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

**2. Add the Volume Mount (Inside the container):**

YAML

```
    volumeMounts:
    - name: enc
      mountPath: /etc/kubernetes/enc
      readOnly: true
```

**3. Add the Host Volume (On the actual node):**

YAML

```
  volumes:
  - name: enc
    hostPath:
      path: /etc/kubernetes/enc
      type: DirectoryOrCreate
```

Once you save the file, the `kube-apiserver` will automatically restart and apply the configuration.

---

## 4. The "Gotcha": Encrypting Existing Data

When the API server restarts, it will ONLY encrypt **newly created or updated** Secrets. Any Secrets that existed prior to this configuration will remain completely unencrypted in `etcd`.

> [!WARNING] Mandatory Migration Step
> 
> To secure your cluster, you must force the API server to rewrite all existing Secrets into `etcd` so they pass through the new encryption provider.

Run this command as a cluster administrator to rewrite all secrets:

Bash

```
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

### Locking Down the Cluster (Removing Plaintext Access)

Once you confirm all data is encrypted, edit your `enc.yaml` file and **delete** the `- identity: {}` line from the `providers` list. Restart the API server. This ensures the API server will strictly refuse to read any unencrypted secrets, preventing accidental plaintext storage.

---

## 5. Day 2 Operations

### How to Verify Encryption

Create a new secret, then use `etcdctl` to read it directly from the database:

Bash

```
ETCDCTL_API=3 etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret | hexdump -C
```

_Look for the prefix `k8s:enc:aescbc:v1:key1`. If you see this instead of plain text, the encryption is working._

### How to Decrypt All Data (Revert to Plaintext)

If you need to disable encryption at rest:

1. Edit `enc.yaml` and move `- identity: {}` to be the **very first** item in the `providers` list.
    
2. Restart the API server.
    
3. Run the rewrite command: `kubectl get secrets --all-namespaces -o json | kubectl replace -f -`
    
4. Remove the `--encryption-provider-config` flag from the `kube-apiserver.yaml`.
    

> [!NOTE] Automatic Reloading
> 
> Instead of restarting the API server every time you rotate a key, you can add the flag `--encryption-provider-config-automatic-reload=true` to the `kube-apiserver`. This forces the server to poll the `enc.yaml` file every minute and apply changes dynamically without downtime.
