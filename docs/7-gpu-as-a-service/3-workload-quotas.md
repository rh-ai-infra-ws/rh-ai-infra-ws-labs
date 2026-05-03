# 📊 GPU Pricing Tiers with Kueue

This lab implements a multi-tiered GPU pricing model using Kueue — separating **reserved** (guaranteed) and **on-demand** (pay-per-use) access to GPU resources.

## Prerequisites

1. **OpenShift AI Operator** — ensure it is installed on your cluster.
2. **GPU Worker Node** — at least one worker node with a GPU. On AWS, a `g5.2xlarge` instance is suitable.
3. **GPU Node Taint** — the GPU node must be tainted.

> **Note**
>
> The taint was applied during the bootstrap process. To reapply it manually:
>
> ```bash
> oc adm taint nodes <your-gpu-node-name> nvidia.com/gpu=Exists:NoSchedule
> ```

---

## Use Case Description

A customer has an OpenShift cluster and needs to implement Kueue to manage GPU workloads with two distinct pricing models:

- **Reserved plan** — for premium customers who pay for guaranteed, immediate access to their allocated GPU resources.
- **On-demand plan** — for general users who are charged per use but may have to wait in a queue until GPU capacity becomes available.

**Reserved capacity** suits workloads that are critical, have strict deadlines, or require predictable performance — financial services high-frequency trading, scientific simulations, media rendering and encoding. Reserved capacity ensures users always have the compute power they've paid for.

**On-demand capacity** suits intermittent, non-critical, or variable workloads — ad-hoc data analysis, dev/test environments, temporary demand spikes from a product launch. Users pay only for what they use, with the trade-off that they may wait during peak times.

> **Note** — While this lab focuses on GPUs, Kueue supports quotas for any resource type including CPU, memory, and custom resources.

### Plan Summary

| Plan | Guarantee | Scheduling behaviour |
|---|---|---|
| **Reserved** 🔒 | Guaranteed quota — admitted immediately | High priority |
| **On-Demand** ⏳ | Shared pool — admitted when capacity is available | Normal priority |

---

## Solution Overview

For each **reserved** customer, create a dedicated `ClusterQueue` with a `nominalQuota` defining their guaranteed GPU slice count. This provides hard isolation — Customer A cannot consume Customer B's quota.

For **on-demand** users, a single shared `ClusterQueue` manages all requests, admitting jobs only when GPU capacity is available.

> From the [Kueue documentation](https://kueue.sigs.k8s.io/docs/overview/) (v0.13.4):
>
> *"Resources in a cluster are typically not homogeneous. Resources could differ in pricing and availability (spot vs. on-demand VMs), architecture (x86 vs. ARM CPUs), or brands and models (Radeon 7000 vs. Nvidia A100 vs. T4 GPUs). A ResourceFlavor is an object that represents these resource variations and allows you to associate them with cluster nodes through labels, taints and tolerations."*

---

## Configuration

### ResourceFlavor

The label selector targets the time-sliced A10G nodes:

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: nvidia-a10g-shared
spec:
  nodeLabels:
    nvidia.com/gpu.product: NVIDIA-A10G-SHARED
```

> **Note — Cohorts (not yet supported by the Operator)**
>
> To maximise GPU utilisation, quota can be borrowed between `ClusterQueues` via Cohorts. An idle GPU in one queue can be borrowed by another, but must be released as soon as the owning queue needs it.
>
> From the Kueue docs: *"Cohorts give you the ability to organize your Quotas. ClusterQueues within the same Cohort can share resources with each other."*
>
> ```yaml
> apiVersion: kueue.x-k8s.io/v1beta1
> kind: Cohort
> metadata:
>   name: gpu-sharing-cohort
> ```
>
> Reference it in a `ClusterQueue` with `cohort: gpu-sharing-cohort`.

### ClusterQueue — Reserved Customer A (4 GPUs)

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: reserved-capacity-customer-a
spec:
  # cohort: gpu-sharing-cohort
  namespaceSelector: {}
  resourceGroups:
    - coveredResources:
        - "nvidia.com/gpu"
      flavors:
        - name: nvidia-a10g-shared
          resources:
            - name: "nvidia.com/gpu"
              nominalQuota: 4
```

### ClusterQueue — Reserved Customer B (4 GPUs)

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: reserved-capacity-customer-b
spec:
  # cohort: gpu-sharing-cohort
  namespaceSelector: {}
  resourceGroups:
    - coveredResources:
        - "nvidia.com/gpu"
      flavors:
        - name: nvidia-a10g-shared
          resources:
            - name: "nvidia.com/gpu"
              nominalQuota: 4
```

### ClusterQueue — On-Demand (8 GPUs shared)

```yaml
apiVersion: kueue.x-k8s.io/v1beta1
kind: ClusterQueue
metadata:
  name: on-demand-capacity
spec:
  # cohort: gpu-sharing-cohort
  namespaceSelector: {}
  resourceGroups:
    - coveredResources:
        - "nvidia.com/gpu"
      flavors:
        - name: nvidia-a10g-shared
          resources:
            - name: "nvidia.com/gpu"
              nominalQuota: 8
```

---

## Verify the Solution

### Create Namespaces and LocalQueues

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: reserved-team-a
  labels:
    kubernetes.io/metadata.name: reserved-team-a
    kueue.openshift.io/managed: 'true'
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  namespace: reserved-team-a
  name: reserved-team-a
spec:
  clusterQueue: reserved-capacity-customer-a
```

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: reserved-team-b
  labels:
    kubernetes.io/metadata.name: reserved-team-b
    kueue.openshift.io/managed: 'true'
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  namespace: reserved-team-b
  name: reserved-team-b
spec:
  clusterQueue: reserved-capacity-customer-b
```

```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: on-demand-team-a
  labels:
    kubernetes.io/metadata.name: on-demand-team-a
    kueue.openshift.io/managed: 'true'
---
apiVersion: kueue.x-k8s.io/v1beta1
kind: LocalQueue
metadata:
  namespace: on-demand-team-a
  name: on-demand-team-a
spec:
  clusterQueue: on-demand-capacity
```

### Example Job

> **Replace `<namespace>` and `<local-queue-name>` before applying.**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: reserved-capacity-customer-a-
  namespace: <namespace>
  labels:
    kueue.x-k8s.io/queue-name: <local-queue-name>
spec:
  template:
    spec:
      containers:
      - name: sleeper
        image: registry.access.redhat.com/ubi9/ubi:latest
        command: ["/bin/sleep"]
        args: ["300"]
        resources:
          limits:
            nvidia.com/gpu: 1
          requests:
            nvidia.com/gpu: 1
      restartPolicy: Never
  backoffLimit: 4
```

### Tasks

- 🔎 Verify that each customer cannot exceed their assigned GPU quota.
- ❌ Remove the label `kueue.x-k8s.io/queue-name` from the `Job` and try to submit jobs that exceed the quota — confirm Kueue blocks them.
- ⌛️ Add a memory limit of `1Gi` to Customer A's `ClusterQueue` and verify it is enforced.
- ➕ Add another on-demand team and verify both teams can independently consume all 8 GPUs when the other is idle.

### Kueue Dashboard

```bash
kubectl -n kueue-system port-forward svc/kueue-kueueviz-backend 8080:8080 &
kubectl -n kueue-system set env deployment kueue-kueueviz-frontend REACT_APP_WEBSOCKET_URL=ws://localhost:8080
kubectl -n kueue-system port-forward svc/kueue-kueueviz-frontend 3000:8080
```

Open [http://localhost:3000](http://localhost:3000) in the browser.

---

## References

- [Kueue Documentation](https://kueue.sigs.k8s.io/docs/overview/) — Version May 15, 2025
- [AI on OpenShift Contrib Repo — Kueue Preemption Example](https://github.com/opendatahub-io-contrib/ai-on-openshift)
