# ⚙️ Kueue Configuration

This lab demonstrates how to use Kueue to implement a multi-tiered service model for GPU workloads, simulating "reserved" and "on-demand" pricing plans.

## Prerequisites

1. **OpenShift AI Operator** — ensure it is installed on your cluster.
2. **GPU Worker Node** — at least one worker node with a time-sliced NVIDIA A10G GPU (configured in the previous lab). On AWS, a `g5.2xlarge` instance is suitable.
3. **GPU Node Taint** — the GPU node must be tainted.

> **Note**
>
> This was done during the bootstrap process. To reapply the taint manually:
>
> ```bash
> oc adm taint nodes <your-gpu-node-name> nvidia.com/gpu=Exists:NoSchedule --overwrite
> ```

---

## Use Case Description

A common business requirement is to offer different service levels for expensive GPU resources. This lab simulates two tiers:

- **Reserved Plan** — for premium customers who pay for guaranteed, immediate access to their allocated GPU resources. These workloads are critical and cannot wait in a queue.
- **On-Demand Plan** — for general users who pay per use. These workloads are less critical and can wait for GPU capacity to become available.

Kueue enforces these service levels, ensuring reserved workloads are prioritized and on-demand jobs are admitted based on available capacity.

> **Note** — While this lab focuses on GPUs, Kueue can manage quotas for any resource type, including CPU, memory, and custom resources.

**Reserved capacity** is suited for workloads that are critical, have strict deadlines, or require predictable performance — for example: financial services high-frequency trading, scientific simulations, or media rendering. Reserved capacity eliminates the risk of waiting and ensures the compute power is always available.

**On-demand capacity** is suited for intermittent, non-critical, or variable workloads — for example: ad-hoc data analysis, dev/test environments, or temporary demand spikes. Users pay only for what they use, with the trade-off that they may wait during peak times.

### Plan Summary

| Plan | Guarantee | Priority |
|---|---|---|
| **Reserved** 🔒 | Guaranteed quota — admitted immediately | High |
| **On-Demand** ⏳ | Admitted when capacity is available | Normal |

---

## Solution Overview

We use Kueue's `ResourceFlavor` and `ClusterQueue` objects to create the two-tiered system.

1. **ResourceFlavor** — targets the time-sliced A10G GPUs via a `nodeLabel` selector, ensuring GPU requests land on the correct hardware.
2. **ClusterQueues for tiers:**
   - **Reserved tier** — each reserved customer (`customer-a`, `customer-b`) gets a dedicated `ClusterQueue`. The `nominalQuota` defines their guaranteed GPU slice count.
   - **On-demand tier** — a single `ClusterQueue` serves all on-demand users with a shared pool of GPUs.

> **Note — Cohorts and resource sharing**
>
> Kueue's `Cohort` feature allows different `ClusterQueues` to borrow unused resources from each other. For this lab we intentionally **do not** use Cohorts — the goal is **strict, non-shareable quotas** that simulate hard-fenced reserved plans. `customer-a` cannot borrow from `customer-b` and vice versa.

> From the [Kueue documentation](https://kueue.sigs.k8s.io/docs/overview/) (v0.13.4):
>
> *"Resources in a cluster are typically not homogeneous. Resources could differ in pricing and availability (spot vs. on-demand), architecture (x86 vs. ARM), or brands and models (Radeon 7000 vs. Nvidia A100 vs. T4). A ResourceFlavor is an object that represents these resource variations and allows you to associate them with cluster nodes through labels, taints and tolerations."*

---

## Kueue Configuration

### ResourceFlavor

Create a `ResourceFlavor` targeting the time-sliced A10G nodes:

```bash
cat <<EOF | oc apply -f -
apiVersion: kueue.x-k8s.io/v1beta1
kind: ResourceFlavor
metadata:
  name: nvidia-a10g-shared
spec:
  nodeLabels:
    nvidia.com/gpu.product: NVIDIA-A10G-SHARED
EOF
```

> **Note — Cohorts (optional)**
>
> To allow ClusterQueues to borrow unused resources from each other, configure a `Cohort`:
>
> ```yaml
> apiVersion: kueue.x-k8s.io/v1beta1
> kind: Cohort
> metadata:
>   name: gpu-sharing-cohort
> ```
>
> Then reference it in each `ClusterQueue` via `cohort: gpu-sharing-cohort`. With a Cohort, every time a GPU within a `ClusterQueue` is idle it can be borrowed by another — but must be released as soon as the owning queue needs it. See [Red Hat Kueue Cohorts documentation](https://docs.redhat.com/en/documentation/red_hat_build_of_kueue/1.0/html/cohorts_and_advanced_configurations/using-cohorts).

### ClusterQueue — Reserved Customer A (4 GPUs)

```bash
cat <<EOF | oc apply -f -
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
EOF
```

### ClusterQueue — Reserved Customer B (4 GPUs)

```bash
cat <<EOF | oc apply -f -
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
EOF
```

### ClusterQueue — On-Demand (8 GPUs shared)

```bash
cat <<EOF | oc apply -f -
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
EOF
```

With this configuration each reserved customer has 4 guaranteed virtual GPUs, and 8 virtual GPUs are available in the shared on-demand pool.

---

## Verify the Solution

### Create Namespaces and LocalQueues

Create a `Namespace` and a `LocalQueue` pointing to the correct `ClusterQueue` for each customer:

```bash
cat <<EOF | oc apply -f -
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
EOF
```

```bash
cat <<EOF | oc apply -f -
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
EOF
```

```bash
cat <<EOF | oc apply -f -
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
EOF
```

### Example Job Template

> **Replace `<namespace>` and `<local-queue-name>` before applying. Use `oc create -f <file>.yaml`.**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: test-job-
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
- ⌛️ Add a memory allocation limit of `1Gi` to Customer A's `ClusterQueue` and verify it is enforced.
- ➕ Add another on-demand team and verify both teams can independently consume up to 8 GPUs when the other team is idle.

### Kueue Dashboard

Use the Kueue dashboard for visibility into queue states and workload admission:

```bash
kubectl -n kueue-system port-forward svc/kueue-kueueviz-backend 8080:8080 &
kubectl -n kueue-system set env deployment kueue-kueueviz-frontend REACT_APP_WEBSOCKET_URL=ws://localhost:8080
kubectl -n kueue-system port-forward svc/kueue-kueueviz-frontend 3000:8080
```

Open [http://localhost:3000](http://localhost:3000) in the browser.

---

## Cheat Sheet

<details>
<summary>Create all test jobs at once</summary>

```bash
cat <<EOF | oc create -f -
apiVersion: batch/v1
kind: Job
metadata:
  generateName: reserved-capacity-team-a-
  namespace: reserved-team-a
  labels:
    kueue.x-k8s.io/queue-name: reserved-team-a
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
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
  backoffLimit: 4
---
apiVersion: batch/v1
kind: Job
metadata:
  generateName: reserved-capacity-team-b-
  namespace: reserved-team-b
  labels:
    kueue.x-k8s.io/queue-name: reserved-team-b
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
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
  backoffLimit: 4
---
apiVersion: batch/v1
kind: Job
metadata:
  generateName: on-demand-team-a-
  namespace: on-demand-team-a
  labels:
    kueue.x-k8s.io/queue-name: on-demand-team-a
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
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
  backoffLimit: 4
EOF
```

</details>

---

## References

- [Kueue Documentation](https://kueue.sigs.k8s.io/docs/overview/) — Version May 15, 2025
- [AI on OpenShift Contrib Repo — Kueue Preemption Example](https://github.com/opendatahub-io-contrib/ai-on-openshift)
