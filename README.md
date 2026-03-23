# Kubernetes Knowledge Base

## Overview
This repository contains a comprehensive, structured collection of personal notes, guides, and concepts for Kubernetes administration and architecture. It is designed to serve as a central knowledge base (compatible with markdown-based tools like Obsidian) covering everything from core cluster architecture to advanced troubleshooting, security, and workload management.

## Table of Contents

### Kubernetes Architecture
- Control Plane Installation Guide
- CRDs, Controllers, and Operators
- Kubernetes Architecture
- NameSpaces
- Worker Node Provisioning Join

### Workloads Management
- Daemonset
- Deployment
- Horizontal Pod Autoscaler (HPA)
- ReplicaSet
- StatefulSet
- Vertical Pod Autoscaling (VPA)

### Pod Fundamentals
- Multi Container Pods
- Requests and Limits
- Static Pods

### Scheduling
- Configuring Scheduler Profiles
- Manual Scheduling
- Multiple Schedulers
- Node Affinity
- PriorityClass
- Taints and Tolerations

### Services and Networking
- ClusterIP
- EndpointSlices
- Gateway API
- Ingress
- NodePort

### Storage
- Local & Ephemeral Volumes
- PV and PVC
- Storage Class

### Security and RBAC
- **Admission Control**: Admission Control concepts, Validating and Mutating Admission Controllers
- **Security**: TLS Certificates, Certificates API and User Onboarding, KUBECONFIG
- Image Security
- Network Policies
- RBAC Master Guide
- Security Contexts
- Service Accounts

### ConfigMap and Secrets
- ConfigMaps
- Encrypting Confidential Data at Rest
- Secrets

### Cluster Maintenance
- Backup and Restore Methods
- Cluster upgrade
- Node Drain, Cordon, and Uncordon

### Troubleshooting
- 2 Tier Application
- API Server Troubleshooting
- kubectl Debug Command
- Workernode

### Helm and Kustomize
- Helm
- Kustomize

### Jobs and CronJobs
- CronJobs
- Jobs

### Monitoring and Logging
- Metrics Server

### General Concepts
- ENTRYPOINT vs CMD
- Kubectl Cheat Sheet
- Labels and Annotations

## How to Use
These notes are written in standard Markdown (`.md`). You can read them directly on GitHub or clone the repository to view them locally. 

If you are using a knowledge management tool like Obsidian, you can clone this repository and open the root directory as your vault to take advantage of local graph views and file linking.

```bash
git clone [https://github.com/AdhamAyad/k8s-notes.git](https://github.com/AdhamAyad/k8s-notes.git)
