# 🚀 Run a TrainJob (Kubeflow Trainer v2)

In this lab you’ll submit a distributed PyTorch `TrainJob` using the `torch-distributed` `ClusterTrainingRuntime` 🎛️.  
You’ll run a tiny **DDP** training loop (Fashion-MNIST) across **2 nodes** with **1 GPU each** and confirm the pods/logs look healthy ✅

## Prerequisites

- Completed [Enable distributed training components](1-enable-distributed-training.md)
- `torch-distributed` `ClusterTrainingRuntime` present (`oc get clustertrainingruntime`)
- GPUs available in the cluster and a project namespace (for example `${USER_NAME}-training`)
- Sufficient quota for at least two nodes with one GPU each (adjust `numNodes` and GPU counts for your environment)

## ✅ Verify ClusterTrainingRuntimes

Kubeflow Trainer v2 ships pre-built `ClusterTrainingRuntime` resources. List them:

```bash
oc get clustertrainingruntime
```

* **Sample output:**

```text
NAME                                       AGE
torch-distributed                          24h
torch-distributed-cpu                      24h
torch-distributed-cpu-torch210-py312       24h
torch-distributed-cuda128-torch29-py312    24h
torch-distributed-cuda130-torch210-py312   24h
torch-distributed-rocm                     24h
torch-distributed-rocm64-torch29-py312     24h
torch-distributed-rocm64-torch291-py312    24h
training-hub                               24h
training-hub-cpu                           24h
training-hub-rocm                          24h
training-hub-th05-cuda128-torch29-py312    24h
training-hub-th06-cpu-torch210-py312       24h
training-hub-th06-cuda130-torch210-py312   24h
training-hub-th06-rocm64-torch291-py312    24h
```

Inspect the default PyTorch runtime:

```bash
oc get clustertrainingruntime torch-distributed -o yaml
```

?> You do not create `torch-distributed` yourself. Platform administrators enable it when `trainingoperator` is `Managed`. For custom project needs you can use namespace-scoped `TrainingRuntime` resources

## 🏗️ Create a project and a LocalQueue (Kueue admission)

1. Create a dedicated namespace for this lab:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: v1
kind: Namespace
metadata:
  name: <USER_NAME>-kfpv2-train
  labels:
    name: <USER_NAME>-kfpv2-train
    opendatahub.io/dashboard: 'true'
    kueue.openshift.io/managed: 'true'
EOF
```

2. Create a **LocalQueue** so Kueue can admit workloads from your namespace into a shared **ClusterQueue** 🎟️  
   This example routes KF training requests to the `default` cluster queue.

```bash
cat << 'EOF' | oc apply -n <USER_NAME>-kfpv2-train -f -
apiVersion: kueue.x-k8s.io/v1beta2
kind: LocalQueue
metadata:
  namespace: <USER_NAME>-kfpv2-train
  name: kf-train-lq
spec:
  clusterQueue: default
EOF
```

✅ Quick check:

```bash
oc get localqueue -n <USER_NAME>-kfpv2-train
```

?> (Optional) If something gets stuck in `Pending`, Kueue controller logs can help 🔍

```bash
oc logs -n openshift-kueue-operator -l control-plane=controller-manager
```

* Similar logs expected

```text
...
{"level":"Level(-2)","ts":"2026-05-26T13:40:46.795904084Z","logger":"localqueue-reconciler","caller":"core/localqueue_controller.go:215","msg":"LocalQueue create event","replica-role":"leader","localQueue":{"name":"kf-train-lq","namespace":"user1-kfpv2-train"}}
{"level":"Level(-2)","ts":"2026-05-26T13:40:46.796045481Z","logger":"localqueue-reconciler","caller":"core/localqueue_controller.go:167","msg":"Reconcile LocalQueue","replica-role":"leader","namespace":"user1-kfpv2-train","name":"kf-train-lq","reconcileID":"f9a512aa-179c-4e42-bf92-41e41fd12cd1"}
{"level":"Level(-2)","ts":"2026-05-26T13:40:46.801669833Z","logger":"localqueue-reconciler","caller":"core/localqueue_controller.go:254","msg":"Queue update event","replica-role":"leader","localQueue":{"name":"kf-train-lq","namespace":"user1-kfpv2-train"}}
{"level":"Level(-2)","ts":"2026-05-26T13:40:46.801740167Z","logger":"localqueue-reconciler","caller":"core/localqueue_controller.go:167","msg":"Reconcile LocalQueue","replica-role":"leader","namespace":"user1-kfpv2-train","name":"kf-train-lq","reconcileID":"5b870a41-001d-41cd-90f2-7f8d83c3555b"}
```

## 🧠 Create the training script ConfigMap

1. Create a ConfigMap containing a **minimal DDP** Fashion-MNIST script 🧵  
   It will initialize `torch.distributed`, shard data via `DistributedSampler`, and train for a few epochs.

```bash
cat << 'EOF' | oc apply -n <USER_NAME>-kfpv2-train -f-
apiVersion: v1
kind: ConfigMap
metadata:
  name: train-fashion-mnist
data:
  train.py: |
    import os

    import torch
    import torch.nn as nn
    import torch.distributed as dist
    from torch.utils.data import DataLoader
    from torchvision import datasets
    from torchvision.transforms import ToTensor
    from torch.utils.data.distributed import DistributedSampler
    from torch.nn.parallel import DistributedDataParallel as DDP

    class NeuralNetwork(nn.Module):
        def __init__(self):
            super().__init__()
            self.flatten = nn.Flatten()
            self.linear_relu_stack = nn.Sequential(
                nn.Linear(28 * 28, 512),
                nn.ReLU(),
                nn.Linear(512, 512),
                nn.ReLU(),
                nn.Linear(512, 10),
            )

        def forward(self, inputs):
            inputs = self.flatten(inputs)
            logits = self.linear_relu_stack(inputs)
            return logits

    def get_dataset():
        return datasets.FashionMNIST(
            root="/tmp/data",
            train=True,
            download=True,
            transform=ToTensor(),
        )

    def train_func_distributed():
        num_epochs = 3
        batch_size = 64

        # Kubeflow Trainer sets rendezvous env for torchrun; we still need to init the process group.
        backend = "nccl" if torch.cuda.is_available() else "gloo"
        dist.init_process_group(backend=backend)

        dataset = get_dataset()

        # Local rank maps to GPU index on the node.
        local_rank = int(os.environ.get("LOCAL_RANK", "0"))
        if torch.cuda.is_available():
            torch.cuda.set_device(local_rank)
            device = torch.device("cuda", local_rank)
        else:
            device = torch.device("cpu")

        sampler = DistributedSampler(
            dataset,
            num_replicas=dist.get_world_size(),
            rank=dist.get_rank(),
            shuffle=True,
        )

        dataloader = DataLoader(
            dataset,
            batch_size=batch_size,
            sampler=sampler,
            pin_memory=torch.cuda.is_available(),
        )

        model = NeuralNetwork().to(device)

        if torch.cuda.is_available():
            model = DDP(model, device_ids=[local_rank], output_device=local_rank)
        else:
            model = DDP(model)

        criterion = nn.CrossEntropyLoss()
        optimizer = torch.optim.SGD(model.parameters(), lr=0.01)

        for epoch in range(num_epochs):
            sampler.set_epoch(epoch)

            for inputs, labels in dataloader:
                optimizer.zero_grad()
                inputs = inputs.to(device, non_blocking=True)
                labels = labels.to(device, non_blocking=True)
                pred = model(inputs)
                loss = criterion(pred, labels)
                loss.backward()
                optimizer.step()

            # Avoid log spam: have rank 0 print a simple progress line.
            if dist.get_rank() == 0:
                print(f"epoch: {epoch}, loss: {loss.item():.4f}", flush=True)

        dist.destroy_process_group()

    if __name__ == "__main__":
        train_func_distributed()
EOF
```

## 🚀 Submit the TrainJob

Create a `TrainJob` that references `torch-distributed`, mounts the script, and requests one GPU per node on two nodes:

```bash
cat << 'EOF' | oc apply -n <USER_NAME>-kfpv2-train -f-
apiVersion: trainer.kubeflow.org/v1alpha1
kind: TrainJob
metadata:
  name: pytorch-train-fashion-mnist
  annotations:
    trainer.opendatahub.io/progression-tracking: "true"
    kueue.x-k8s.io/queue-name: kf-train-lq
spec:
  runtimeRef:
    name: torch-distributed
    kind: ClusterTrainingRuntime
  trainer:
    command: ["torchrun", "/workspace/train.py"]
    numNodes: 2
    resourcesPerNode:
      requests:
        cpu: "8"
        memory: "32Gi"
        nvidia.com/gpu: 6
      limits:
        cpu: "8"
        memory: "32Gi"
        nvidia.com/gpu: 6
  podTemplateOverrides:
    - targetJobs:
        - name: node
      spec:
        volumes:
          - name: script-volume
            configMap:
              name: train-fashion-mnist
        containers:
          - name: node
            volumeMounts:
              - name: script-volume
                mountPath: /workspace
EOF
```

## 👀 Monitor the job

1. Watch `TrainJob` status:

```bash
oc get trainjob -n <USER_NAME>-kfpv2-train
```

?> You should see something similar

```text
NAME                          STATE   AGE
pytorch-train-fashion-mnist           2m59s
```

2. List training pods (JobSet creates pods named after the job):

```bash
oc get pods -n <USER_NAME>-kfpv2-train -l trainer.kubeflow.org/trainjob-name=pytorch-train-fashion-mnist
```

?> You should see something similar

```text
NAME                                               READY   STATUS    RESTARTS   AGE
pytorch-train-fashion-mnist-node-0-0-d82hr         1/1     Running   0          2m42s
pytorch-train-fashion-mnist-node-0-1-psrt7         1/1     Running   0          2m42s
```

3. Stream logs from a worker (replace the pod name with one from the previous command):

```bash
oc logs -n <USER_NAME>-kfpv2-train -f <training-pod-name>
```

* **Expected log patterns** (exact numbers vary, only rank 0 logs once per epoch):

```text
epoch: 0, loss: 2.30..
epoch: 1, loss: 2.2..
epoch: 2, loss: 2.1..
```

4. After some time, the TrainJob should finish:

```bash
oc get trainjob -n <USER_NAME>-kfpv2-train pytorch-train-fashion-mnist
```

* **Expected output**

```text
NAME                          STATE      AGE
pytorch-train-fashion-mnist   Complete   24m
```

5. (Optional) Review Kueue logs for extra details 🧾

```bash
oc logs -n openshift-kueue-operator -l control-plane=controller-manager
```

* **Expected output**

```text
...
{"level":"Level(-2)","ts":"2026-05-26T13:53:29.684175984Z","logger":"trainjob","caller":"jobframework/reconciler.go:416","msg":"Reconciling Job","replica-role":"leader","namespace":"user1-kfpv2-train","name":"pytorch-train-fashion-mnist-2","reconcileID":"105e31dd-2c46-4a07-8d36-0790eaaee8b5","job":"user1-kfpv2-train/pytorch-train-fashion-mnist-2","gvk":"trainer.kubeflow.org/v1alpha1, Kind=TrainJob"}
{"level":"debug","ts":"2026-05-26T13:53:29.688039037Z","logger":"events","caller":"recorder/recorder.go:104","msg":"Workload 'user1-kfpv2-train/trainjob-pytorch-train-fashion-mnist-2-d7ac0' is declared finished","type":"Normal","object":{"kind":"TrainJob","namespace":"user1-kfpv2-train","name":"pytorch-train-fashion-mnist-2","uid":"3049952c-4558-4391-bf7e-6b89c02cd006","apiVersion":"trainer.kubeflow.org/v1alpha1","resourceVersion":"648114"},"reason":"FinishedWorkload"}
...
```