# Kubernetes Objects: ConfigMaps

**Tags:** #kubernetes #configmap #configuration #volumes #env-vars #CKA #devops

**Status:** Core Concept

---

## 1. Overview & Core Concept

A **ConfigMap** is an API object used to store non-confidential data in key-value pairs. It allows you to **decouple** environment-specific configuration from your container images.

### Key Behaviors

- **Decoupling:** "Build once, Deploy anywhere." Your container image remains the same (immutable); the config changes based on the environment (Dev, Test, Prod).
    
- **Portability:** You don't need to rebuild the image to change a database URL or a log level.
    
- **Size Limit:** Maximum size is **1 MiB**. For larger data, use a standard Volume or external storage.
    

> [!WARNING] Security Alert
> 
> ConfigMaps provide **NO encryption**. They are stored in plain text.
> 
> - For sensitive data (Passwords, Keys, Tokens), use **[[Secrets]]** instead.
>     

---

## 2. Motivation (The Scenario)

Imagine you are developing a web app:

- **Local (Dev):** `DATABASE_HOST = localhost`
    
- **Cloud (Prod):** `DATABASE_HOST = prod-db.aws.com`
    

**Without ConfigMap:** You hardcode the IP in the code or build 2 different Docker images.

**With ConfigMap:**

1. Code reads `DATABASE_HOST` env var.
    
2. Deploy to Dev -> Inject ConfigMap with `localhost`.
    
3. Deploy to Prod -> Inject ConfigMap with `prod-db.aws.com`.
    
4. **Result:** The Docker Image is identical in both places.
    

---

## 3. How to Consume ConfigMaps

There are 4 ways to use a ConfigMap inside a Pod.

### A. Environment Variables (The Simple Way)

Best for simple values like URLs, Flags, or IDs.

|**Method**|**Syntax**|**Behavior**|
|---|---|---|
|**Specific Key**|`valueFrom.configMapKeyRef`|Picks **one** key and assigns it to a variable name you choose.|
|**All Keys**|`envFrom.configMapRef`|Dumps **ALL** keys from the ConfigMap as variables. Variable names = Key names.|

> [!NOTE] Naming Restrictions
> 
> If a key in the ConfigMap is invalid for an Environment Variable (e.g., contains a dot `app.config`), Kubernetes will silently **skip** it when using `envFrom`.

### B. Volumes (The File Way)

Best for full configuration files (e.g., `nginx.conf`, `redis.properties`).

- **Mechanism:** Kubernetes creates a "Virtual Directory" inside the container.
    
- **Mapping:** Each **Key** becomes a **Filename**. The **Value** becomes the **File Content**.
    
- **Overwrite:** Mounting a ConfigMap volume typically **covers/hides** everything else in that directory (unless using `subPath`).
    

---

## 4. Advanced Volume Mounting Strategies

### Strategy A: Directory Mount (The Default)

Mounts the _entire_ ConfigMap as a directory.

- **Behavior:** Overwrites the target directory.
    
- **Update:** Supports Live Updates.
    

YAML

```
volumeMounts:
  - name: config-vol
    mountPath: /etc/config  # Result: /etc/config/game.properties
```

### Strategy B: `subPath` (Single File Injection)

Injects a single file into an _existing_ directory.

- **Behavior:** Does NOT overwrite the directory. It places the file alongside existing files.
    
- **Update:** **NO Live Updates** (File is static).
    

YAML

```
volumeMounts:
  - name: config-vol
    mountPath: /etc/nginx/nginx.conf  # Path to the FILE
    subPath: nginx.conf               # The Key name in ConfigMap
```

### Strategy C: `items` (Projection & Renaming)

Filters specific keys and renames them before mounting.

- **Behavior:** Only mounts what you specify.
    
- **Update:** Supports Live Updates (if mounted as a directory).
    

YAML

```
volumes:
  - name: config-vol
    configMap:
      name: my-config
      items:
        - key: "db_url"        # Source Key
          path: "settings.xml" # Destination Filename (Renaming)
```

---

## 5. Lifecycle & Live Updates (Vital Distinction)

How does the Pod react when you change the ConfigMap data?

| **Consumption Method** | **Live Update?** | **Behavior**                                                                                       |
| ---------------------- | ---------------- | -------------------------------------------------------------------------------------------------- |
| **Environment Vars**   | **NO**           | Variables are injected at startup. You must **Restart** the Pod to see changes.                    |
| **Directory Mount**    | **YES**          | Kubelet syncs changes automatically using Symlinks. (Latency: ~1 minute). App must support reload. |
| **SubPath Mount**      | **NO**           | Files mounted via `subPath` break the symlink chain. They are snapshots. **Restart required**.     |
|                        |                  |                                                                                                    |


> [!IMPORTANT] Runtime Persistence (Experimental Fact)
> 
> Deleting a ConfigMap will **NOT** immediately affect a running Pod.
> 
> - **Behavior:** The Pod continues to function because the data remains injected in memory (RAM) or locked in the mount point.
>     
> - **The Risk:** The failure is deferred until the Pod **restarts** or is **re-created**, at which point it will fail with `CreateContainerConfigError` due to the missing source.
>     

---

## 6. Immutable ConfigMaps

_Feature State: Stable_

You can mark a ConfigMap as read-only.

- **Field:** `immutable: true`.
    
- **Why?**
    
    1. **Safety:** Prevents accidental updates that could break apps.
        
    2. **Performance:** Reduces load on the API Server (Kubelet stops watching for changes).
        
- **Action:** To change data, you must delete and recreate the ConfigMap.
    

---

## 7. Master YAML Reference

A comprehensive example covering Data types, Env injection, Directory Mounting, SubPath injection, and Item Renaming.

### The ConfigMap

YAML

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
  namespace: default
immutable: false     # Set to true to prevent changes
data:
  # --- Simple Keys ---
  player_lives: "3"
  ui_config: "user-interface.properties"

  # --- File-like Keys (Multi-line) ---
  game.properties: |
    enemy.types=aliens,monsters
    level=hard
  
  extra.config: |
    color=red
```

### The Pod (Consuming Everything)

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-master-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      
      # --- 1. ENVIRONMENT VARIABLES ---
      env:
        # A. Rename a Key
        - name: LIVES_COUNT
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: player_lives

      # B. Inject ALL Keys
      envFrom:
        - configMapRef:
            name: game-demo

      # --- 2. VOLUME MOUNTS ---
      volumeMounts:
        # Scenario A: Directory Mount (Live Updates)
        # Result: /etc/config/game.properties
        # Warning: Overwrites /etc/config content
        - name: full-vol
          mountPath: "/etc/config"
        
        # Scenario B: SubPath (Single File Injection) (No Live Updates)
        # Result: Injects 'game.properties' into /app/ without deleting existing files
        - name: full-vol
          mountPath: "/app/settings.properties"
          subPath: "game.properties"

        # Scenario C: Filtered Items (Live Updates)
        # Result: /etc/filtered/custom-name.xml
        - name: filtered-vol
          mountPath: "/etc/filtered"

  volumes:
    # 1. The Full Flash Drive
    - name: full-vol
      configMap:
        name: game-demo

    # 2. The Filtered Flash Drive (With Renaming)
    - name: filtered-vol
      configMap:
        name: game-demo
        items:
          - key: "extra.config"      # Take this key
            path: "custom-name.xml"  # Rename it to this file
```