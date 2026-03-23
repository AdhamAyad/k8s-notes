# Kubernetes Scheduling: Multiple Schedulers

**Tags:** #kubernetes #scheduling #multiple-schedulers #cka

**Status:** Core Concept

## 1. Core Concept: Why and When to Use Multiple Schedulers?

Kubernetes comes with a default scheduler (`default-scheduler`) that distributes Pods evenly across nodes based on standard rules like Taints, Tolerations, Node Affinity, and resource availability. 

However, you might encounter scenarios where the default logic is not enough. You use **Multiple Schedulers** when:
- **Custom Scheduling Algorithms:** You have specific workloads that require a completely custom algorithm to determine node placement.
- **Extra Verification:** An application needs to perform external checks or extra verification steps (e.g., checking hardware availability, licensing, or external API conditions) before being placed on a specific node.

Instead of completely replacing the default scheduler, Kubernetes allows you to deploy your custom scheduler *alongside* the default one. You can then tell specific Pods to be handled by your custom scheduler, while the rest of the cluster continues using the default one.



## 2. Configuration: The KubeSchedulerConfiguration Object

Before deploying a custom scheduler, you must define its identity using a configuration file of kind `KubeSchedulerConfiguration`. 

> [!IMPORTANT] The Golden Rule of Multiple Schedulers
> 
> Every additional scheduler running in the cluster **MUST** have a strictly unique name. The default is conventionally named `default-scheduler`. If you do not assign a unique name in the configuration file, it will conflict with the default scheduler and cause cluster-wide scheduling failures.

**Example: Custom Scheduler Config File (`my-scheduler-config.yaml`)**
```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: my-custom-scheduler
# Optional: Disable leader election if running only a single instance of this custom scheduler
    leaderElection:
      leaderElect: false
````

> [!WARNING] Leader Election Concept
> 
> `leaderElection` is a crucial configuration for High-Availability (HA) environments. If you deploy multiple replicas of your _custom scheduler_ to ensure it doesn't go down, leader election ensures that only one replica actively schedules Pods at any given time, preventing race conditions.

## 3. Deployment Methods

A custom scheduler is essentially just another instance of the `kube-scheduler` binary running with a different configuration file. It can be deployed in three ways:

1. **Systemd Service (Binary):** Installed directly on the host machine.
    
2. **Static Pod / Standard Pod:** Running inside the `kube-system` namespace.
    
3. **Deployment:** Running as an HA Deployment. This requires configuring RBAC (ServiceAccount, ClusterRole, ClusterRoleBinding) so the custom scheduler has the API permissions needed to read cluster states and bind Pods.
    

**Example: Deploying as a Pod inside the cluster**

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
    - name: kube-scheduler
      image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
      command:
        - kube-scheduler
        - --address=127.0.0.1
        - --kubeconfig=/etc/kubernetes/scheduler.conf
        # Pointing the binary to our custom config file
        - --config=/etc/kubernetes/my-scheduler-config.yaml
```

## 4. Assigning Workloads to the Custom Scheduler

To instruct a Pod to bypass the default scheduler and use your custom one, you must explicitly declare the `schedulerName` inside the Pod's `spec`.

**Example: Pod Definition**

YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom-scheduled
spec:
  # This tells the default scheduler to ignore this Pod
  schedulerName: my-custom-scheduler
  containers:
    - name: nginx
      image: nginx
```

> [!INFO] Pending State Troubleshooting
> 
> If you specify a `schedulerName` that does not exist in the cluster, or if your custom scheduler is crashed/misconfigured, the Pod will remain permanently in the `Pending` state.

## 5. Verification and Troubleshooting

After deploying a Pod with a custom scheduler, you need to verify which scheduler actually handled the binding process.

**Method A: Check Namespace Events**

The `Source` column in the events list will clearly state the name of the scheduler that assigned the Pod to a node.

Bash

```
kubectl get events -o wide
```

_Expected Output Snippet:_

Plaintext

```
REASON      SOURCE                 MESSAGE
Scheduled   my-custom-scheduler    Successfully assigned default/nginx to node01
```

**Method B: Check Custom Scheduler Logs**

If things go wrong, view the logs of your custom scheduler Pod running in the `kube-system` namespace.

Bash

```
kubectl logs my-custom-scheduler --namespace=kube-system
```