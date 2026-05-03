# 📦 Installation & Configuration

## Prerequisites

1. **OpenShift AI Operator:** Ensure the OpenShift AI Operator is installed on your cluster.
2. **GPU Worker Node:** You need at least one worker node with a minimum of 1 GPU. On AWS, a `g5.2xlarge` instance or similar is suitable.
3. **GPU Node Taint:** The GPU node must be tainted to ensure only GPU-tolerant workloads are scheduled on it.

> **Check current GPU nodes:**
>
> ```bash
> oc get nodes
> oc describe node/<your-gpu-node-name> | grep -A10 "Taints:"
> ```
>
> If you need to apply the taint:
>
> ```bash
> oc adm taint nodes <your-gpu-node-name> nvidia.com/gpu=Exists:NoSchedule
> ```

## Node Feature Discovery (NFD) Operator

OpenShift (and Kubernetes in general) is designed to be hardware-agnostic — it does not inherently know what specific hardware components (like NVIDIA GPUs) are present on its nodes. The NFD Operator fills this gap.

**Standardized labeling:** Once NFD discovers a specific hardware feature like an NVIDIA GPU, it applies a standardized label to that node. For NVIDIA GPUs the most common label is:

```
feature.node.kubernetes.io/pci-10de.present=true
```

Where:
- `feature.node.kubernetes.io/` — standard prefix for NFD-generated labels
- `pci-10de` — the PCI vendor ID for NVIDIA Corporation (`10de`), uniquely identifying NVIDIA hardware
- `.present=true` — indicates a device with this PCI ID is present on the node

## NVIDIA GPU Operator

The GPU Operator uses NFD labels as a selector and deploys its components **only** to nodes that have the `feature.node.kubernetes.io/pci-10de.present=true` label, ensuring resources are not wasted on non-GPU nodes.

These labels are also fundamental for the Kubernetes scheduler. When a Pod requests GPU resources (e.g. `resources.limits.nvidia.com/gpu: 1`), the scheduler uses NFD labels to find nodes with the necessary GPU capacity.

> **From the OpenShift documentation (v4.19):**
>
> *"In addition, the worker nodes can host one or more GPUs, but they must be of the same type. For example, a node can have two NVIDIA A100 GPUs, but a node with one A100 GPU and one T4 GPU is not supported. The NVIDIA Device Plugin for Kubernetes does not support mixing different GPU models on the same node."*

**Multiple nodes with different GPU types** — the recommended approach — uses dedicated worker nodes per GPU model:

| Node | GPUs |
|---|---|
| Node 1 | Two NVIDIA A100 GPUs |
| Node 2 | Four NVIDIA T4 GPUs |
| Node 3 | CPU-only |

The maximum number of GPUs per node is limited by the number of [PCIe slots](https://www.hp.com/us-en/shop/tech-takes/what-are-pcie-slots-pc) on the motherboard.

> **Important — Do NOT deploy the NVIDIA Network Operator in this lab.**
>
> <details>
> <summary>About the NVIDIA Network Operator</summary>
>
> The NVIDIA Network Operator manages high-performance networking for AI/HPC workloads on OpenShift. It enables:
>
> - **RDMA (Remote Direct Memory Access):** Direct memory transfers between machines, bypassing the OS — reducing latency and CPU overhead.
> - **GPUDirect RDMA:** Direct data path between NVIDIA GPUs and RDMA-capable network adapters, bypassing CPU and system memory entirely. Critical for distributed deep learning.
> - **SR-IOV:** Shares a single physical NIC across multiple containers as if they had dedicated hardware.
> - **High-speed secondary networks:** Dedicated network interfaces for application traffic separate from the cluster primary network.
>
> </details>

## Verify the GPU Operator is Running

After installation, verify the GPU Operator by running `nvidia-smi` inside a driver toolkit pod.

**Short version:**

```bash
oc exec -it -n nvidia-gpu-operator \
  $(oc get pod -o wide -l openshift.driver-toolkit=true \
    -o jsonpath="{.items[0].metadata.name}" -n nvidia-gpu-operator) \
  -- nvidia-smi
```

**Step by step:**

```bash
oc get pod -o wide -l openshift.driver-toolkit=true -n nvidia-gpu-operator
```

Expected output:

```
NAME                                           READY   STATUS    RESTARTS   AGE   IP            NODE
nvidia-driver-daemonset-9.6.20250811-0-ch2j2   2/2     Running   0          19m   10.130.0.9    ip-10-0-61-182.us-east-2.compute.internal
nvidia-driver-daemonset-9.6.20250811-0-gdwn8   2/2     Running   0          19m   10.129.0.14   ip-10-0-45-75.us-east-2.compute.internal
```

Run `nvidia-smi` inside one of the pods:

```bash
oc exec -it -n nvidia-gpu-operator nvidia-driver-daemonset-9.6.20250811-0-ch2j2 -- nvidia-smi
```

Expected output:

```
Sat Sep 13 13:50:41 2025
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.82.07              Driver Version: 580.82.07      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA A10G                    On  |   00000000:00:1E.0 Off |                    0 |
|  0%   26C    P8             24W /  300W |       0MiB /  23028MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

Since there are two GPU-enabled nodes, both could have different configurations — it is worth checking both.
