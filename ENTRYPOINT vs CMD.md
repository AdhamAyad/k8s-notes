# Docker & Kubernetes Architecture: ENTRYPOINT vs CMD

**Tags:** #docker #kubernetes #containers #architecture #cka #manifests
**Status:** Core Concept

---

## 1. Core Concept: The TL;DR

Both `ENTRYPOINT` and `CMD` are instructions in a Dockerfile that define what command executes when a container starts. However, they serve different architectural purposes:

* **`CMD` (The Default):** Defines default commands or parameters. It is highly flexible and **easily overridden** by the user when running the container.
* **`ENTRYPOINT` (The Executable):** Configures the container to run as a specific executable. It enforces a strict command that is **hard to override**, treating any user input as arguments to that command.



---

## 2. The Kubernetes Translation (The "Gotcha")

When moving from Docker to Kubernetes, the terminology changes completely. The creators of Kubernetes chose different names for these exact same concepts, which is a common trap in the CKA exam.

Here is the strict mapping you must memorize:
* Docker **`ENTRYPOINT`** = Kubernetes **`command`**
* Docker **`CMD`** = Kubernetes **`args`**

> [!WARNING] Naming Confusion
> In a Kubernetes manifest, `command` does NOT map to Docker's `CMD`. 
> If you provide `command` in a Kubernetes Pod manifest, it completely overrides the Docker image's `ENTRYPOINT`. 
> If you provide `args` in a Kubernetes Pod manifest, it completely overrides the Docker image's `CMD`.

---

## 3. Deep Dive: CMD (K8s: args)

Use `CMD` when you want to provide a default command, but you want to give the user the complete freedom to run something else.

**Dockerfile Example:**
```dockerfile
FROM ubuntu
CMD ["sleep", "10"]
````

**Execution Behavior:**

- `docker run my-image` -> Runs `sleep 10` (Uses the default).
    
- `docker run my-image sleep 5` -> Runs `sleep 5` (User input completely replaces the CMD).
    

**Kubernetes Equivalent (Overriding CMD):**

To override the Docker `CMD` in Kubernetes, you use the `args` field.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: override-cmd-pod
spec:
  containers:
  - name: ubuntu-container
    image: my-image
    # This overrides the Dockerfile's CMD
    args: ["sleep", "50"] 
```

---

## 4. Deep Dive: ENTRYPOINT (K8s: command)

Use `ENTRYPOINT` when your container is designed to do exactly one thing and you do not want users to accidentally override the main binary.

**Dockerfile Example:**

Dockerfile

```
FROM ubuntu
ENTRYPOINT ["sleep"]
```

**Execution Behavior:**

- `docker run my-image 10` -> Runs `sleep 10` (User input is appended as an argument).
    
- `docker run my-image bash` -> Runs `sleep bash` (Fails, because ENTRYPOINT was not overridden).
    

**Kubernetes Equivalent (Overriding ENTRYPOINT):**

To override the strict `ENTRYPOINT` in Kubernetes, you must use the `command` field.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: override-entrypoint-pod
spec:
  containers:
  - name: ubuntu-container
    image: my-image
    # This completely replaces the Dockerfile's ENTRYPOINT
    command: ["/bin/bash"]
    args: ["-c", "echo Hello World"]
```

---

## 5. The Best Practice: Combining Both

In production environments, the most powerful and common pattern is to combine both instructions in the Dockerfile. `ENTRYPOINT` defines the fixed executable, and `CMD` provides the default arguments.

**Dockerfile Example:**

Dockerfile

```
FROM ubuntu
# 1. The fixed binary that will always run
ENTRYPOINT ["sleep"]

# 2. The default argument if the user provides nothing
CMD ["10"]
```

**Kubernetes Implementation (Overriding Default Arguments):**

When an image is built this way, you typically only want to change the _arguments_ in Kubernetes, not the executable itself.

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: best-practice-pod
spec:
  containers:
  - name: sleeper
    image: my-sleeper-image
    # We leave the ENTRYPOINT ("sleep") alone.
    # We only override the CMD ("10") using args.
    args: ["3600"] 
```

_Result in K8s:_ The container runs `sleep 3600`.

---

## 6. Exec Form vs. Shell Form

How you write the instruction completely changes how Docker executes it inside the container. This applies to both `ENTRYPOINT` and `CMD`.

### A. Exec Form (JSON Array) - Recommended

Dockerfile

```
ENTRYPOINT ["sleep", "10"]
```

- Executes the command directly, bypassing the shell.
    
- The process becomes `PID 1` inside the container.
    
- It cleanly receives system signals (like `SIGTERM` when you run `docker stop` or when Kubernetes deletes the Pod).
    

### B. Shell Form (String) - Not Recommended

Dockerfile

```
ENTRYPOINT sleep 10
```

- Docker silently wraps your command in `/bin/sh -c`.
    
- The shell becomes `PID 1`, and your actual application becomes a child process.
    

> [!IMPORTANT] The PID 1 Problem
> 
> If you use the Shell Form, a termination signal (`SIGTERM`) will be sent to the shell, not your application. The shell often ignores this, causing your container to hang for a 10-to-30-second grace period until Docker/Kubernetes forcefully kills it (`SIGKILL`). Always use the Exec Form (JSON array `["command", "arg"]`) to ensure graceful shutdowns and zero-downtime deployments.
