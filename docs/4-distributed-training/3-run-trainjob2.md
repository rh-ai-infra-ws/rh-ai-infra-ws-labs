# 🧩 Kubeflow Trainer

Kubeflow Trainer v2 separates **what you train** (your `TrainJob` spec) from **how distributed training runs** (the training runtime). This lab explains `ClusterTrainingRuntime`, `TrainingRuntime`, and the `runtimeRef` field.

## ClusterTrainingRuntime vs TrainingRuntime

| Kind | Scope | Who creates it | Typical use |
| --- | --- | --- | --- |
| `ClusterTrainingRuntime` | Cluster | Platform (pre-built) | Standard PyTorch distributed, Training Hub fine-tuning |
| `TrainingRuntime` | Namespace | Project admin | Custom images, env vars, or default resources for a team |

**ClusterTrainingRuntime** templates replace the per-job pod and `torchrun` boilerplate that Training Operator v1 required on every `PyTorchJob`. They configure:

- `torchrun` entrypoint and `numProcPerNode`
- `MASTER_ADDR`, `MASTER_PORT`, and node DNS patterns
- Default training container images from Red Hat OpenShift AI

**TrainingRuntime** is for project-specific overrides: a custom image with extra libraries, default CPU/memory/GPU requests, or shared volume mounts.

## Pre-built ClusterTrainingRuntimes

Red Hat OpenShift AI provides runtimes such as:

| Runtime | Description |
| --- | --- |
| `torch-distributed` | General-purpose PyTorch distributed training with CUDA |
| `torch-distributed-rocm` | PyTorch distributed for AMD ROCm GPUs |
| `torch-distributed-cuda128-torch29-py312` | PyTorch 2.9, CUDA 12.8, Python 3.12 |
| `training-hub` | Training Hub with built-in fine-tuning algorithms (OSFT, SFT) |
| `training-hub-th05-cuda128-torch29-py312` | Training Hub with pinned CUDA/PyTorch stack |

List runtimes available on your cluster:

```bash
oc get clustertrainingruntime
```

## How runtimeRef connects a TrainJob to a runtime

In a `TrainJob`, `spec.runtimeRef` selects the template:

```yaml
spec:
  runtimeRef:
    name: torch-distributed
    kind: ClusterTrainingRuntime
```

- `kind: ClusterTrainingRuntime` — cluster-wide pre-built runtime.
- `kind: TrainingRuntime` — namespace-scoped custom runtime in your project.

The `trainer` section then defines your workload:

- `command` — for example `["torchrun", "/workspace/train.py"]`
- `numNodes` — total distributed nodes (replaces separate Master/Worker replica counts from v1)
- `resourcesPerNode` — same requests/limits applied to every node

## Example runtime structure

The following excerpt shows how `torch-distributed` configures distributed PyTorch (abbreviated):

```yaml
apiVersion: trainer.kubeflow.org/v1alpha1
kind: ClusterTrainingRuntime
metadata:
  name: torch-distributed
spec:
  mlPolicy:
    torch:
      numProcPerNode: auto
  template:
    spec:
      replicatedJobs:
        - name: Node
          template:
            spec:
              template:
                spec:
                  containers:
                    - name: trainer
                      env:
                        - name: MASTER_ADDR
                          value: "$(JOB_NAME)-node-0-0.$(JOB_NAME)"
                        - name: MASTER_PORT
                          value: "29500"
                      command:
                        - torchrun
```

?> When using `podTemplateOverrides`, the target job name is often `node` (lowercase) to match the default `torch-distributed` runtime template.

## Create a custom TrainingRuntime (optional)

For most labs, `torch-distributed` is sufficient. Create a `TrainingRuntime` only when you need a project-specific image or defaults.

1. In the **Administrator** perspective, go to **Home** → **Search**, select your project, and create a **TrainingRuntime** resource.

2. Example custom runtime with a pinned image and default resources:

```bash
cat << 'EOF' | oc apply -n "${USER_NAME}-training" -f-
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainingRuntime
metadata:
  name: custom-torch-runtime
spec:
  mlPolicy:
    torch:
      numProcPerNode: auto
  template:
    spec:
      replicatedJobs:
        - name: Node
          template:
            spec:
              template:
                spec:
                  containers:
                    - name: trainer
                      image: registry.redhat.io/rhoai/odh-training-cuda128-torch28-py312-rhel9:v3.0
                      env:
                        - name: MASTER_ADDR
                          value: "$(JOB_NAME)-node-0-0.$(JOB_NAME)"
                        - name: MASTER_PORT
                          value: "29500"
                      command:
                        - torchrun
                      resources:
                        requests:
                          cpu: "4"
                          memory: "16Gi"
                        limits:
                          cpu: "4"
                          memory: "16Gi"
EOF
```

Verify:

```bash
oc get trainingruntime -n "${USER_NAME}-training"
```

Reference it from a `TrainJob` with `kind: TrainingRuntime` and `name: custom-torch-runtime`.

## Next steps

Continue to [Run a TrainJob](3-run-trainjob.md) to submit a minimal multi-node PyTorch training job.
