# 🧩 MIG & Time-Slicing

## MIG — Multi-Instance GPU (Production Recommended)

NVIDIA's Multi-Instance GPU (MIG) technology allows you to partition a single compatible GPU (A100, H100, A30, etc.) into multiple smaller, **fully isolated and independent** GPU instances. Each instance has its own dedicated compute, memory, and cache resources — providing hardware-level guarantees.

MIG advantages over time slicing:
- **Hardware-level isolation and security** — instances cannot interfere with each other
- **Predictable performance and QoS** — dedicated resources per instance
- **Maximized GPU utilization and cost efficiency** — multiple tenants on one GPU
- **Fine-grained resource allocation** — choose from predefined instance profiles
- **Simplified management** in Kubernetes via the NVIDIA Device Plugin

For further configuration options see the [Custom MIG Configuration During Installation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html#example-custom-mig-configuration-during-installation) documentation.

> **Note — Lab hardware limitation**
>
> The GPUs in this lab (**AWS NVIDIA A10G**) do **not** support MIG. The MIG configuration below is for production reference only — do not apply it in this lab. Time slicing is used instead for the hands-on exercises.

### ConfigMap for MIG

Create a `ConfigMap` to specify the MIG configuration. It **must** be named `custom-mig-config` and reside in the `nvidia-gpu-operator` namespace.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-mig-config
  namespace: nvidia-gpu-operator
data:
  config.yaml: |
    version: v1
    mig-configs:
      all-disabled:
        - devices: all
          mig-enabled: false

      custom-mig:
        - devices: all  # target individual GPUs by index if needed
          mig-enabled: true
          mig-devices:
            "1g.5gb": 2
            "2g.10gb": 1
            "3g.20gb": 1
```

Always use a [supported MIG profile configuration](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#a100-mig-profiles) for your GPU model.

### Patch the ClusterPolicy for MIG

Modify the `gpu-cluster-policy` in the `nvidia-gpu-operator` namespace to enable MIG:

1. If the custom configuration specifies more than one instance profile, set the strategy to `mixed`:

    ```bash
    oc patch clusterpolicies.nvidia.com/cluster-policy \
        --type='json' \
        -p='[{"op":"replace", "path":"/spec/mig/strategy", "value":"mixed"}]'
    ```

2. Point the MIG Manager to the custom ConfigMap:

    ```bash
    oc patch clusterpolicies.nvidia.com/cluster-policy \
        --type='json' \
        -p='[{"op":"replace", "path":"/spec/migManager/config/name", "value":"custom-mig-config"}]'
    ```

3. Label the nodes with the desired MIG profile:

    ```bash
    oc label nodes <node-name> nvidia.com/mig.config=custom-mig --overwrite
    ```

---

## Time Slicing (Lab Configuration)

> **This is what is configured in the lab** due to A10G hardware constraints.

NVIDIA time slicing allows multiple processes to share a single GPU, where each process gets a time slice to access GPU resources. It is easier to configure than MIG but provides no hardware-level isolation between workloads.

**When to use time slicing:**
- Hardware does not support MIG (e.g. A10G, T4)
- Mixed or bursty lightweight workloads
- Cost efficiency is prioritized over isolation
- Trusted multi-tenant environment

### ConfigMap for Time Slicing

Create the ConfigMap — it can have any name but must reside in the `nvidia-gpu-operator` namespace. Set `replicas` to the number of virtual GPU slices per physical GPU.

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: device-plugin-config
  namespace: nvidia-gpu-operator
data:
  time-sliced: |-
    version: v1
    sharing:
      timeSlicing:
        resources:
          - name: nvidia.com/gpu
            replicas: 8
EOF
```

### Patch the ClusterPolicy for Time Slicing

Enable GPU Feature Discovery and point the Device Plugin to the ConfigMap:

```bash
oc patch clusterpolicy gpu-cluster-policy \
    -n nvidia-gpu-operator --type json \
    -p '[{"op": "replace", "path": "/spec/gfd/enable", "value": true}]'
```

```bash
oc patch clusterpolicy gpu-cluster-policy \
  -n nvidia-gpu-operator --type merge \
  -p '{"spec": {"devicePlugin": {"config": {"name": "device-plugin-config"}}}}'
```

### Label the Nodes

Label GPU nodes to trigger the new configuration. The GPU Operator detects this label and applies time slicing.

```bash
oc label --overwrite node \
    --selector=nvidia.com/gpu.product=NVIDIA-A10G-SHARED \
    nvidia.com/device-plugin.config=time-sliced
```

> **Note — Label selector**
>
> The selector `nvidia.com/gpu.product=NVIDIA-A10G-SHARED` must match the GPU product name as labeled by the GPU Operator's NFD component.
