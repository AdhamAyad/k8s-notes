# Kubernetes Controllers: Jobs

**Tags:** #kubernetes #jobs #batch #controller #workloads #CKA #patterns

**Status:** Core Concept

---

## 1. Overview & Core Concept

A **Job** creates one or more Pods and ensures a specified number of them successfully terminate. Unlike Deployments (which manage infinite/ongoing applications like Web Servers), Jobs are designed for **Finite Tasks** (Batch Processes) that must run to completion.

### Key Behaviors

- **Run-to-Completion:** The goal is for the process to exit with **Status Code 0**.
    
- **Retry Mechanism:** If a Pod fails (node crash, eviction, non-zero exit code), the Job controller creates a _new_ Pod or restarts the container based on policy.
    
- **Success Criteria:** The Job is considered complete when the number of successful Pods reaches `.spec.completions`.
    

---

## 2. Restart Policy: The Vital Distinction

Defined in `.spec.template.spec.restartPolicy`. **Jobs cannot use `Always`**.

|**Policy**|**Behavior (What happens when it fails?)**|**Data Persistence**|**Best For**|
|---|---|---|---|
|**`Never`**|**Re-create:** The Controller creates a completely **NEW Pod** (New Name, New IP, potentially New Node).|**Lost:** Data in `emptyDir` or container memory is lost.|Clean Slate tasks (e.g., trying to process a file from scratch).|
|**`OnFailure`**|**Restart:** The Kubelet restarts the **Container** inside the _SAME_ Pod (Same Node, Same IP).|**Kept:** Data in `emptyDir` is preserved.|Heavy tasks (e.g., Resume large download, ML checkpointing).|

---

## 3. Job Patterns (Parallelism vs. Completions)

This is the most critical part to understand the different behaviors.

- **Completions (`.spec.completions`):** Target number of successful terminations required to finish the Job.
    
- **Parallelism (`.spec.parallelism`):** Number of Pods to run simultaneously (Concurrency Level).
    

### Pattern A: Non-Parallel Job

- **Config:** `completions: 1`, `parallelism: 1` (Defaults).
    
- **Behavior:** Starts 1 pod. Job is done when pod succeeds.
    

### Pattern B: Parallel Job with Fixed Completion Count

- **Config:** `completions: 5`, `parallelism: 2`.
    
- **Behavior:** I need 5 successful tasks. Run 2 at a time. As one finishes, start another until 5 successes are reached.
    
- **Use Case:** Processing a batch of 5 independent files.
    

### Pattern C: Parallel Job with Work Queue (Data Analytics)

- **Config:** **`completions: unset` (null)**, `parallelism: N`.
    
- **Behavior (The "Coordinator" Logic):**
    
    - Pods start working (e.g., pulling from a Shared Queue like RabbitMQ).
        
    - **Termination:** The Job considers itself done when **at least one Pod succeeds** AND **all Pods are terminated**.
        
    - **Logic:** Pods are programmed to exit with `0` when the Queue is empty. When the first Pod finishes successfully, it signals that the queue is drained, triggering the termination of all other Pods.
        

### Pattern D: Parallel Job with Single Completion (Redundant Execution)

- **Config:** `completions: 1`, `parallelism: N`.
    
- **Behavior:** Start N pods at the same time. The **First one to succeed** makes the Job complete, and Kubernetes immediately **terminates the other running pods**.
    
- **Scenario (Redundant Backup):**
    
    - You need to backup a critical database.
        
    - You spin up 3 Pods simultaneously (e.g., uploading to AWS, Azure, and GCP).
        
    - As soon as the first one finishes the upload, the Job is marked successful, and the remaining Pods are terminated to save resources (since only one successful copy is required).
        

---

## 5. Indexed Jobs (Static Sharding)

Defined by `.spec.completionMode: Indexed`.

- **Concept:** Instead of creating identical replicas, each Pod gets a unique index from `0` to `.spec.completions-1`.
    
- **Why?** To replace complex "Work Queues" for static workloads. Instead of a queue, you give each Pod a slice of the data.
    
- **Identification Methods:**
    
    1. **Env Variable:** `JOB_COMPLETION_INDEX` (e.g., `0`, `1`, `2`).
        
    2. **Hostname:** `$(job-name)-$(index)`.
        
- **Scenario:**
    
    - You have 100,000 videos needing rendering.
        
    - Instead of using a complex Message Queue, you statically partition them into 10 groups.
        
    - You create a Job with `completions: 10`.
        
    - Pod index `0` processes the first 10k videos, Pod index `1` processes the second 10k, etc. The application logic reads the Env Var to determine its specific dataset.
        

---

## 6. Failure Handling Strategies

### Level 1: Standard Backoff (Global)

- **Field:** `.spec.backoffLimit` (Default: 6).
    
- **Logic:** If the total number of retries exceeds 6, the Job is marked `Failed`. Uses exponential backoff delay (10s, 20s, 40s...).
    

### Level 2: Backoff Limit Per Index (Indexed Jobs Only)

- **Field:** `.spec.backoffLimitPerIndex`.
    
- **Logic:** Handles retries for each index independently. If Index 5 fails 3 times, only Index 5 is marked failed, others continue.
    

### Level 3: Pod Failure Policy (Granular Control)

- **Field:** `.spec.podFailurePolicy`.
    
- **Purpose:** Ignore certain errors (like preemption) or fail fast on bugs based on **Exit Codes**.
    
- **Actions:**
    
    - `FailJob`: Fail immediately (e.g., app exits with code 42 - Bug).
        
    - `Ignore`: Do not count toward `backoffLimit` (e.g., Node Preemption / Spot Instance termination).
        

---

## 7. Lifecycle & Cleanup

### A. Automatic Cleanup (TTL)

- **Field:** `.spec.ttlSecondsAfterFinished`.
    
- **Function:** Deletes the Job and Pods X seconds after finishing. (Critical to keep ETCD clean).
    

### B. Active Deadline (Timeout)

- **Field:** `.spec.activeDeadlineSeconds`.
    
- **Function:** Hard timeout. If the job takes longer than this, terminate everything.
    

---

## 8. YAML Master Library (Scenarios)

### Scenario 1: Redundant Backup (Parallelism with Single Completion)

_Goal: Run 3 replicas, first one to succeed wins, kill others._

YAML

```
apiVersion: batch/v1
kind: Job
metadata:
  name: redundant-backup
spec:
  completions: 1       # We only need 1 success
  parallelism: 3       # Run 3 at the same time
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: backup-agent
        image: backup-tool:v1
        command: ["/bin/sh", "-c", "upload-to-cloud.sh"]
```

### Scenario 2: Message Queue Consumer (Work Queue)

_Goal: Pods pull from RabbitMQ. No fixed completion count. They exit when queue is empty._

YAML

```
apiVersion: batch/v1
kind: Job
metadata:
  name: queue-consumer
spec:
  # completions: UNSET (Must be removed/nil)
  parallelism: 5        # 5 Workers processing the queue
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: worker
        image: consumer-app:v1
        # App logic: Connect to Queue -> Process -> If Queue Empty -> Exit 0
```

### Scenario 3: Indexed Job (Static Partitioning)

_Goal: Process 3 separate datasets using Index ID._

YAML

```
apiVersion: batch/v1
kind: Job
metadata:
  name: indexed-processing
spec:
  completions: 3
  parallelism: 3
  completionMode: Indexed    # <--- Activates Indexing
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: python:3.9
        command: ["python", "main.py"]
        env:
        # Use this var in code to decide which file to open
        # Pod 0 opens file_0.txt, Pod 1 opens file_1.txt
        - name: MY_INDEX_ID
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
```

### Scenario 4: Smart Failure (Fail Fast & Ignore Preemption)

_Goal: Fail immediately if code bugs (Exit 42), but ignore platform restarts._

YAML

```
apiVersion: batch/v1
kind: Job
metadata:
  name: smart-fail
spec:
  backoffLimit: 6
  podFailurePolicy:
    rules:
    # Rule 1: Application Bug -> Kill Job Immediately
    - action: FailJob
      onExitCodes:
        operator: In
        values: [42]
    # Rule 2: Spot Instance Preemption -> Retry without counting against limit
    - action: Ignore
      onPodConditions:
      - type: DisruptionTarget
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: main
        image: my-app:v1
```

### Scenario 5: Suspended Job

_Goal: Create the Job object but don't run it until triggered._

YAML

```
apiVersion: batch/v1
kind: Job
metadata:
  name: suspended-job
spec:
  suspend: true    # <--- Job is created but paused
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: heavy-task
        image: data-cruncher
```