# Comprehensive Guide: Advanced Troubleshooting with `kubectl debug`

**Tags:** #kubernetes #debugging #troubleshooting #cka #security

**Status:** Core Operations & Troubleshooting Guide

---

## 1. Architectural Overview: The Power of `kubectl debug`

When standard commands like `kubectl exec` or `kubectl logs` fail to provide the necessary insights, `kubectl debug` serves as the ultimate diagnostic tool in Kubernetes. It allows you to troubleshoot workloads and nodes without modifying the original declarative configurations.

The command operates using three primary low-level mechanisms:

1. **Ephemeral Containers:** Injecting a temporary, diagnostic container into an already running Pod without restarting it.
    
2. **Pod Copying:** Cloning an existing Pod and overriding its execution command to prevent it from crashing while you inspect its environment.
    
3. **Node Debugging (Host Access):** Deploying a highly privileged Pod directly onto a specific worker node that shares the host's namespaces, granting administrative access to the underlying operating system without requiring SSH.
    

---

## 2. Scenario 1: Debugging Distroless Images (Ephemeral Containers)

> [!SCENARIO] The "Distroless" Problem
> 
> Modern security best practices dictate using **Distroless Images** for application containers. These images contain only your application and its runtime dependencies. They strictly omit OS utilities like `sh`, `bash`, `curl`, or `ls` to reduce the attack surface.
> 
> If you attempt to run `kubectl exec -it my-pod -- sh`, Kubernetes will throw an error: `executable file not found in $PATH`. You are locked out of your own container.

**The Solution:** Inject an ephemeral debugging container (like `busybox`) directly into the running Pod.

### The Command

Bash

```
kubectl debug -it my-running-pod --image=busybox --target=my-app-container
```

### Command Breakdown

- `kubectl debug -it my-running-pod`: Targets the running Pod and opens an interactive terminal.
    
- `--image=busybox`: Tells the API server to pull and inject a new temporary container using the `busybox` image (which contains tools like `ping`, `wget`, `ps`).
    
- `--target=my-app-container`: **The crucial flag.** This instructs Kubernetes to share the **Process Namespace** between the new `busybox` container and your original application container.
    

> [!TIP] How to Access the Target's Filesystem
> 
> When you execute this command, you are dropped into the shell of the `busybox` container, NOT the application container. They share the same Network and Processes, but **not** the same filesystem.
> 
> To inspect the application's files from inside your `busybox` shell, you must look through the shared process tree:
> 
> 1. Run `ps aux` to find the Process ID (PID) of your application (usually PID 1).
>     
> 2. Run `ls /proc/1/root/` to browse the target application's actual filesystem.
>     

---

## 3. Scenario 2: Surviving a CrashLoopBackOff (Pod Copying)

> [!SCENARIO] The "CrashLoop" Problem
> 
> Your Pod is continuously crashing every few seconds (status `CrashLoopBackOff`). Because the container terminates almost immediately, you never have enough time to successfully run `kubectl exec` to get inside and investigate the misconfiguration or missing dependencies.

**The Solution:** Clone the Pod and override its startup command so it stays alive.

### The Command

Bash

```
kubectl debug my-crashing-pod -it --copy-to=my-debugger-pod --container=my-app-container -- sh
```

### Command Breakdown

- `my-crashing-pod`: The name of the original, failing Pod.
    
- `--copy-to=my-debugger-pod`: Instructs Kubernetes to create an exact duplicate of the Pod (inheriting the same ConfigMaps, Secrets, Volumes, and ServiceAccounts) and name it `my-debugger-pod`.
    
- `--container=my-app-container`: Specifies which container inside the Pod you want to manipulate.
    
- `-- sh`: **The Override.** This replaces the original failing `ENTRYPOINT` or `CMD` of your application with a simple `sh` shell.
    

> [!INFO] The Sandbox Concept: Why copy the Pod?
> A common misconception is that you create a copy of the Pod to fix the code inside it and leave it running. This is incorrect.
> 
> **The actual purpose:** When a Pod is in a `CrashLoopBackOff` state, it terminates too quickly for you to enter it and inspect the environment. The `--copy-to` flag creates a stable, isolated "laboratory" clone by overriding the crashing startup command with a simple shell (`sh`). 
> 
> Inside this stable clone, you act as an investigator (e.g., testing database URLs, pinging services, or manually running the application script to catch the exact error). Once you identify the root cause, you exit the clone, delete it entirely, and apply the permanent fix to your actual `Deployment` or `ConfigMap` manifests.

**The Result:** The cloned Pod spins up and stays in a healthy `Running` state because the failing application code was bypassed. You are dropped into the shell with the exact same environment variables and volume mounts, allowing you to manually test connections and inspect configuration files safely.

> [!WARNING] Best Practice: Housekeeping
> 
> Kubernetes does not automatically track or delete the cloned Pods created via the `--copy-to` flag. Once you have finished debugging and identified the root cause, you must manually clean up the cluster to avoid wasting resources:
> 
> `kubectl delete pod my-debugger-pod`

---

## 4. Scenario 3: Node-Level Troubleshooting (Privileged Access)

> [!SCENARIO] The "Node Access" Problem
> 
> You suspect a physical host issue on a specific worker node (e.g., DNS resolution failure at the host level, corrupted kubelet configurations, or network interface issues). However, you do not have SSH access to the node, or port 22 is blocked by corporate firewalls.

**The Solution:** Deploy a highly privileged debug Pod that bypasses standard isolation and mounts the host's namespaces.

### The Command

Bash

```
kubectl debug node/worker-node-1 -it --image=ubuntu
```

### Command Breakdown

- `node/worker-node-1`: Explicitly targets a Kubernetes Node rather than a Pod.
    
- `--image=ubuntu`: Uses a fully-featured OS image for the debugging session.
    

**The Result:** Kubernetes dynamically spins up a privileged Pod scheduled specifically on `worker-node-1`. This Pod is configured to share the **Host PID, Host IPC, and Host Network** namespaces.

> [!INFO] Achieving Root File Access to the Node
> 
> When you drop into this shell, you are running as root inside the Ubuntu container, but you are still seeing the container's filesystem.
> 
> Kubernetes automatically mounts the actual Node's root filesystem into a special directory inside your debug container. To act as if you have SSH access directly into the host machine, run:
> 
> `chroot /host`
> 
> You now have full root access to the Node's underlying filesystem and can restart systemd services, check host logs, or modify host network configurations.