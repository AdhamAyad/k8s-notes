# Kubernetes Architecture: Validating & Mutating Admission Webhooks

**Tags:** #kubernetes #architecture #admission-controllers #webhooks #security #python #cka

**Status:** Advanced Implementation Guide

## 1. Core Concept: Mutating vs. Validating

Admission Webhooks allow you to intercept requests to the Kubernetes API server using your own custom logic (an external HTTP server) before objects are saved to `etcd`. They operate in two strictly ordered phases:

1. **Phase 1 - Mutating Admission (The Modifier):** Runs first. It can modify the incoming YAML (e.g., injecting default labels, adding sidecar containers).
2. **Phase 2 - Validating Admission (The Judge):** Runs second. It looks at the *final* mutated YAML and decides to either Approve (`allowed: true`) or Deny (`allowed: false`) the request.

> [!IMPORTANT] The Golden Rule of Admission
> 
> If *any* admission controller (built-in or webhook) in either phase returns `allowed: false`, the entire request is instantly dropped, an error is returned to the user, and the object is never created.

## 2. The Communication Protocol (AdmissionReview)

The API server communicates with your webhook via a JSON object called `AdmissionReview`. 
- **The Request:** The API Server POSTs the user's YAML inside `request.object`.
- **The Response:** Your server must reply with the exact same `uid` and an `allowed` boolean. If mutating, it must also include a Base64 encoded `JSONPatch`.

> [!WARNING] Mandatory TLS
> 
> The Kubernetes API server strictly requires all webhooks to be served over **HTTPS**. Your webhook server must have TLS certificates, and you must provide the CA bundle to the cluster.

---

## 3. The Code: Smart Webhook Server (Python Flask)

Here is a full production-ready Python server handling both endpoints:
- `/validate`: Rejects the Pod if the user tries to name it after themselves.
- `/mutate`: Safely injects an `environment: dev` label. It handles edge cases (e.g., if the user didn't write any labels, or if they already specified an environment).

```python
from flask import Flask, request, jsonify
import base64
import json

app = Flask(__name__)

# ==========================================
# 1. VALIDATING LOGIC (The Judge)
# ==========================================
@app.route('/validate', methods=['POST'])
def validate():
    req = request.json
    uid = req["request"]["uid"]
    object_name = req["request"]["object"]["metadata"].get("name", "")
    user_name = req["request"]["userInfo"]["name"]
    
    status = True
    message = "Pod is valid."

    # Rule: Pod name cannot match the user's name
    if object_name == user_name:
        status = False
        message = "Security Violation: You cannot create objects with your own name."

    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": uid,
            "allowed": status,
            "status": {"message": message}
        }
    })

# ==========================================
# 2. MUTATING LOGIC (The Modifier)
# ==========================================
@app.route('/mutate', methods=['POST'])
def mutate():
    req = request.json
    uid = req["request"]["uid"]
    
    # Extract existing labels safely
    metadata = req["request"]["object"].get("metadata", {})
    labels = metadata.get("labels", None)
    
    patch = [] # JSONPatch instructions list

    # Smart Patching Logic
    if labels is None:
        # Case A: User wrote no labels at all -> Create the entire block
        patch.append({"op": "add", "path": "/metadata/labels", "value": {"environment": "dev"}})
    elif "environment" not in labels:
        # Case B: Labels exist, but no environment -> Add only environment
        patch.append({"op": "add", "path": "/metadata/labels/environment", "value": "dev"})
    else:
        # Case C: User already defined an environment -> Do nothing (Pass)
        pass 

    # Encode the patch to Base64 (Mandatory for Kubernetes)
    patch_string = json.dumps(patch)
    encoded_patch = base64.b64encode(patch_string.encode()).decode()

    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": uid,
            "allowed": True, # Mutators must allow the request to pass the patch
            "patchType": "JSONPatch",
            "patch": encoded_patch
        }
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=443, ssl_context=('/certs/tls.crt', '/certs/tls.key'))
````

---

## 4. Hosting the Webhook in Kubernetes (Deployment & Service)

You must build a Docker image for the Python code (e.g., `my-webhook-app:v1`) and deploy it inside the cluster.

YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-webhook-deploy
  namespace: webhook-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: custom-webhook
  template:
    metadata:
      labels:
        app: custom-webhook
    spec:
      containers:
      - name: server
        image: my-webhook-app:v1
        ports:
        - containerPort: 443
        volumeMounts:
        - name: tls-certs
          mountPath: "/certs"
          readOnly: true
      volumes:
      - name: tls-certs
        secret:
          secretName: webhook-tls-secret # Contains tls.crt and tls.key
---
apiVersion: v1
kind: Service
metadata:
  name: custom-webhook-svc
  namespace: webhook-system
spec:
  ports:
  - port: 443
    targetPort: 443
  selector:
    app: custom-webhook
```

---

## 5. The "Glue": Webhook Configurations

Finally, you tell the Kubernetes API Server to send traffic to your Service using Configuration objects.

### A. The Mutating Configuration

This intercepts the request at Phase 1 and sends it to `/mutate`.

YAML

```
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: inject-dev-label-config
webhooks:
  - name: mutate.example.com
    clientConfig:
      service:
        name: custom-webhook-svc
        namespace: webhook-system
        path: "/mutate"
      caBundle: "LS0tLS1CRUdJTi..." # Base64 encoded CA Certificate
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```

### B. The Validating Configuration

This intercepts the request at Phase 2 and sends it to `/validate`.

YAML

```
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: restrict-pod-name-config
webhooks:
  - name: validate.example.com
    clientConfig:
      service:
        name: custom-webhook-svc
        namespace: webhook-system
        path: "/validate"
      caBundle: "LS0tLS1CRUdJTi..." 
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
```
