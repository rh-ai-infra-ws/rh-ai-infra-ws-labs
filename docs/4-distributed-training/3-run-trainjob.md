# 🚀 Run a TrainJob

This exercise submits a minimal distributed PyTorch `TrainJob` using the `torch-distributed` `ClusterTrainingRuntime`. The script initializes NCCL, prints rank and GPU information, and runs `all_reduce` across workers.

## Prerequisites

- Completed [Enable distributed training components](1-enable-distributed-training.md)
- `torch-distributed` `ClusterTrainingRuntime` present (`oc get clustertrainingruntime`)
- GPUs available in the cluster and a project namespace (for example `${USER_NAME}-training`)
- Sufficient quota for at least two nodes with one GPU each (adjust `numNodes` and GPU counts for your environment)


## Verify ClusterTrainingRuntimes

Kubeflow Trainer v2 ships pre-built `ClusterTrainingRuntime` resources. List them:

```bash
oc get clustertrainingruntime
```

* **Sample output:**

```text
NAME                                          AGE
torch-distributed                             5d
torch-distributed-rocm                        5d
torch-distributed-cuda128-torch29-py312       5d
training-hub                                  5d
```

Inspect the default PyTorch runtime:

```bash
oc get clustertrainingruntime torch-distributed -o yaml
```

?> You do not create `torch-distributed` yourself. Platform administrators enable it when `trainingoperator` is `Managed`. Custom project needs can use namespace-scoped `TrainingRuntime` resources (covered in the next section).

## Create a project for training jobs

Create a dedicated namespace for this lab:

```bash
export USER_NAME=<your-openshift-username>
oc new-project "${USER_NAME}-training"
```

Grant yourself administrator access on the project if it was created by an admin on your behalf:

```bash
oc adm policy add-role-to-user admin "${USER_NAME}" -n "${USER_NAME}-training"
```


## Create the training script ConfigMap

1. Set your project namespace:

```bash
export NAMESPACE="${USER_NAME}-training"
```

2. Create a ConfigMap with a minimal distributed training script:

```bash
cat << 'EOF' | oc apply -n "${NAMESPACE}" -f-
apiVersion: v1
kind: ConfigMap
metadata:
  name: minimal-train-script
data:
  train.py: |
    import torch
    import torch.distributed as dist

    dist.init_process_group(backend="nccl")
    rank = dist.get_rank()
    world_size = dist.get_world_size()

    print(f"Rank {rank}/{world_size}")
    print(f"PyTorch: {torch.__version__}, CUDA available: {torch.cuda.is_available()}")
    if torch.cuda.is_available():
        print(f"GPU: {torch.cuda.get_device_name(0)}")

    tensor = torch.ones(1, device="cuda") * rank
    dist.all_reduce(tensor)
    print(f"Rank {rank}: all_reduce result = {tensor.item()}")

    dist.destroy_process_group()
    print("Done.")
EOF
```

## Submit the TrainJob

Create a `TrainJob` that references `torch-distributed`, mounts the script, and requests one GPU per node on two nodes:

```bash
cat << 'EOF' | oc apply -n "${NAMESPACE}" -f-
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: pytorch-minimal-example
  annotations:
    trainer.opendatahub.io/progression-tracking: "true"
spec:
  runtimeRef:
    name: torch-distributed
    kind: ClusterTrainingRuntime
  trainer:
    command: ["torchrun", "/workspace/train.py"]
    numNodes: 2
    resourcesPerNode:
      requests:
        cpu: "2"
        memory: "8Gi"
        nvidia.com/gpu: 1
      limits:
        cpu: "2"
        memory: "8Gi"
        nvidia.com/gpu: 1
  podTemplateOverrides:
    - targetJobs:
        - name: node
      spec:
        volumes:
          - name: script-volume
            configMap:
              name: minimal-train-script
        containers:
          - name: node
            volumeMounts:
              - name: script-volume
                mountPath: /workspace
EOF
```

!> Adjust `numNodes` and `nvidia.com/gpu` to match your cluster capacity. On smaller environments, use `numNodes: 1` to validate the controller and runtime before scaling out.

## Monitor the job

1. Watch `TrainJob` status:

```bash
oc get trainjob -n "${NAMESPACE}"
```

2. List training pods (JobSet creates pods named after the job):

```bash
oc get pods -n "${NAMESPACE}" -l trainer.kubeflow.org/trainjob-name=pytorch-minimal-example
```

3. Stream logs from a worker (replace the pod name with one from the previous command):

```bash
oc logs -n "${NAMESPACE}" -f <training-pod-name>
```

* **Expected log patterns** (ranks and GPU names will differ):

```text
Rank 0/2
PyTorch: 2.x.x, CUDA available: True
GPU: NVIDIA ...
Rank 0: all_reduce result = 1.0
Done.
```

Each rank should report its index, CUDA availability, and a consistent `all_reduce` result after synchronization.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `TrainJob` not created | `oc api-resources \| grep -i trainjob` — CRD installed with `trainingoperator` |
| Pods pending | `oc describe pod <pod> -n "${NAMESPACE}"` — GPU quota, Kueue admission, or node labels |
| NCCL / CUDA errors | GPU operator and device plugin healthy; driver version matches training image |
| Wrong mount path | `podTemplateOverrides` target job `node` and container name `node` match `torch-distributed` |

Inspect Kueue workload state if jobs remain queued:

```bash
oc get workloads -n "${NAMESPACE}"
```

## Clean up

Remove the lab resources when finished:

```bash
oc delete trainjob pytorch-minimal-example -n "${NAMESPACE}" --ignore-not-found
oc delete configmap minimal-train-script -n "${NAMESPACE}" --ignore-not-found
```

## Further reading

- [Running Kubeflow Trainer v2-based distributed training workloads](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.3/html/working_with_distributed_workloads/running-kubeflow-trainerv2_distributed-workloads) — TrainJob API, Training Hub fine-tuning, SDK usage
- [Working with distributed workloads](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html-single/working_with_distributed_workloads/working_with_distributed_workloads) — Ray, Kueue, and infrastructure overview
