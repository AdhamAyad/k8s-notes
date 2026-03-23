# Kubernetes Controllers: CronJob

**Tags:** #kubernetes #cronjob #batch #controller #scheduling #CKA

**Status:** Core Concept (Stable v1.21+)

---

## 1. Overview

A **CronJob** creates **[[Jobs]]` on a time-based schedule. It is essentially the Kubernetes equivalent of a Unix `crontab` line. It is designed for periodic tasks like backups, report generation, or email sending.

### Core Constraints

- **Naming Limit:** A CronJob name must be **max 52 characters**.
    
    - _Reason:_ The controller appends 11 characters to the generated `[[Jobs]]` name, and `[[Jobs]]` names are capped at 63 characters.
        
- **Idempotency:** Your code should be idempotent (safe to run multiple times).
    
    - _Warning:_ Rarely, Kubernetes might create _two_ `[[Jobs]]` for the same schedule or _zero_ `[[Jobs]]`.
        
- **Modification:** Changing a CronJob spec only affects **future** executions. Existing running `[[Jobs]]` are untouched.
    

---

## 2. Schedule & Time Configuration

### A. Cron Syntax

Defined in `.spec.schedule`. (Required)

Format: `Minute Hour DayOfMonth Month DayOfWeek`

|**Field**|**Range**|**Example**|**Description**|
|---|---|---|---|
|**Minute**|0-59|`*/5`|Every 5 minutes|
|**Hour**|0-23|`0`|Midnight|
|**Day of Month**|1-31|`1`|1st of the month|
|**Month**|1-12|`*`|Every month|
|**Day of Week**|0-6|`5`|Friday (0 is Sunday)|

### B. Macros (Shortcuts)

|**Entry**|**Equivalent**|**Description**|
|---|---|---|
|`@hourly`|`0 * * * *`|Run once an hour|
|`@daily`|`0 0 * * *`|Run once a day (midnight)|
|`@weekly`|`0 0 * * 0`|Run once a week (Sunday)|
|`@monthly`|`0 0 1 * *`|Run once a month|
|`@yearly`|`0 0 1 1 *`|Run once a year|

### C. Time Zones (v1.27+ Stable)

- **Default:** Kube-controller-manager's local time (usually UTC).
    
- **Config:** Set `.spec.timeZone`.
    
- **Example:** `timeZone: "Africa/Cairo"` or `timeZone: "Etc/UTC"`.
    
- **Note:** Do **not** use `CRON_TZ` or `TZ` inside the schedule string; this is unsupported and will cause validation errors.
    

---

## 3. Concurrency Policy

Defined in `.spec.concurrencyPolicy`. Determines what happens if a new `[[Jobs]]` is scheduled to start while the **previous one is still running**.

| **Policy**            | **Behavior**                                                                         | **Use Case**                                  |
| --------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------- |
| **`Allow`** (Default) | The new `[[Jobs]]` starts concurrently. You might have 5 `[[Jobs]]` running at once. | Fast, stateless tasks where overlap is fine.  |
| **`Forbid`**          | The new `[[Jobs]]` is **skipped**. It waits for the next schedule.                   | Database backups (prevent locking conflicts). |
| **`Replace`**         | The old running `[[Jobs]]` is **terminated**, and the new one starts.                | Real-time dashboards (old data is useless).   |

---

## 4. Deadlines & History Limits

### A. Starting Deadline

- **Field:** `.spec.startingDeadlineSeconds`.
    
- **Purpose:** If the `[[Jobs]]` cannot start on time (e.g., cluster down, controller busy), how long should it keep trying?
    
- **Behavior:**
    
    - **If Set (e.g., 200):** If the current time is > 200s past the scheduled time, **skip** this run.
        
    - **If Unset:** It generally tries to catch up, unless too many were missed (>100).
        
- **The "100 Missed" Rule:** If the controller is down for a long time and misses >100 start times:
    
    - If `startingDeadlineSeconds` is **Unset**: The CronJob stops and logs an error.
        
    - If `startingDeadlineSeconds` is **Set**: It only checks for missed `[[Jobs]]` _within that window_, allowing the CronJob to resume normally.
        

### B. History Limits (Cleanup)

Kubernetes does not purge old `[[Jobs]]` by default unless configured.

- **`.spec.successfulJobsHistoryLimit`**: (Default: 3). How many "Completed" `[[Jobs]]` to keep.
    
- **`.spec.failedJobsHistoryLimit`**: (Default: 1). How many "Failed" `[[Jobs]]` to keep.
    
- **Recommendation:** Set these explicitly to avoid flooding `kubectl get jobs`.
    

---

## 5. Master YAML Reference

This configuration demonstrates a production-ready CronJob with TimeZone, Concurrency controls, and History limits.

YAML

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-daily
spec:
  # --- 1. SCHEDULE & TIME ---
  schedule: "0 3 * * *"       # Every day at 3:00 AM
  timeZone: "Africa/Cairo"    # Explicit TimeZone
  
  # --- 2. CONTROLLER LOGIC ---
  concurrencyPolicy: Forbid   # Don't run if previous backup is stuck
  startingDeadlineSeconds: 200 # If 3:00 AM is missed, give up at 3:03:20
  suspend: false              # Active immediately
  
  # --- 3. CLEANUP ---
  successfulJobsHistoryLimit: 3 # Keep last 3 successes
  failedJobsHistoryLimit: 1     # Keep last 1 failure
  
  # --- 4. JOB TEMPLATE ---
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup-tool
            image: my-backup:v1
            args: ["/bin/sh", "-c", "backup.sh"]
          restartPolicy: OnFailure
```

---

## 6. CronJob Pattern Library (Cookbook)

Here are the specific YAML patterns for different use cases/options found in the documentation.

### Scenario A: The "Don't Overlap" (Forbid Concurrency)

**Use Case:** Database migrations or Backups where running two instances at once causes corruption.

YAML

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: heavy-migration
spec:
  schedule: "*/30 * * * *"
  #  Option: Skip new run if old one is still going
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: migrator
            image: db-tool
```

### Scenario B: The "Freshness First" (Replace Concurrency)

**Use Case:** A task that pushes the "current status" to a dashboard. If a run is slow, the data is stale, so kill it and start fresh.

YAML

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: status-updater
spec:
  schedule: "* * * * *" # Every minute
  #  Option: Kill old, Start new
  concurrencyPolicy: Replace
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: updater
            image: curl-image
```

### Scenario C: The "Strict Window" (Deadline)

**Use Case:** A report that MUST be generated between 8:00 and 8:15. If the cluster is down until 8:30, do NOT generate the report (it's too late).

YAML

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: morning-report
spec:
  schedule: "0 8 * * *"
  #  Option: If you can't start within 15 mins (900s), skip it.
  startingDeadlineSeconds: 900
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: report
            image: reporter
```

### Scenario D: The "Paused" CronJob (Suspend)

**Use Case:** You want to deploy the manifest to the cluster, but don't want it to run yet (e.g., waiting for a maintenance window).

YAML

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: on-hold-job
spec:
  schedule: "@midnight"
  #  Option: Deploy now, activate later
  suspend: true 
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: worker
            image: worker:v2
```

_(To activate: `kubectl patch cronjob on-hold-job -p '{"spec":{"suspend":false}}'`)_

### Scenario E: Aggressive Cleanup (History Limits)

**Use Case:** A `[[Jobs]]` that runs every minute. Default settings would keep 3 successes, but maybe you don't want _any_ history to save space.

YAML

```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: noisy-job
spec:
  schedule: "* * * * *"
  #  Option: Delete pods immediately after finish (keep 0 history)
  successfulJobsHistoryLimit: 0
  failedJobsHistoryLimit: 0
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: echo
            image: busybox
```