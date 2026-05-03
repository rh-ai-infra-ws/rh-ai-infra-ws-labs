# 🔍 Validation & Troubleshooting

## Verify Time Slicing

After labeling the nodes, verify that time slicing was applied and the node reports 8 GPU replicas:

```bash
oc get node --selector=nvidia.com/gpu.product=NVIDIA-A10G-SHARED -o json | jq '.items[0].status.capacity'
```

Expected output:

```json
{
  "cpu": "8",
  "ephemeral-storage": "104266732Ki",
  "hugepages-1Gi": "0",
  "hugepages-2Mi": "0",
  "memory": "32499872Ki",
  "nvidia.com/gpu": "8",
  "pods": "250"
}
```

Check the NVIDIA labels applied by the GPU Operator:

```bash
oc get node --selector=nvidia.com/gpu.product=NVIDIA-A10G-SHARED -o json \
  | jq '.items[0].metadata.labels' | grep nvidia
```

Key labels to look for:

```
"nvidia.com/device-plugin.config": "time-sliced",
"nvidia.com/gpu.count": "1",
"nvidia.com/gpu.product": "NVIDIA-A10G-SHARED",
"nvidia.com/gpu.replicas": "8",
"nvidia.com/gpu.sharing-strategy": "time-slicing",
"nvidia.com/mig.capable": "false",
```

---

## Configure Hardware Profiles in OpenShift AI

MIG technology enables a single physical GPU to be partitioned into multiple isolated instances. These configurations are encapsulated in **Hardware Profiles** in OpenShift AI, which abstract complex resource configurations so data scientists can request appropriate hardware without deep Kubernetes expertise.

**Taints and Tolerations** ensure GPU nodes are reserved for AI workloads. GPU nodes are tainted to prevent general workloads from being scheduled on them; Hardware Profiles automatically apply tolerations to allow AI workloads through.

> **Warning — MIG hardware profile without MIG enabled**
>
> Hardware Profiles referencing MIG resources (e.g. `nvidia.com/mig-2g.20gb`) can be created even when MIG is not configured on the cluster. However, pods using such profiles will stay in **Pending** state indefinitely as the scheduler cannot find the requested resource.

### Create a Hardware Profile via YAML

Hardware Profiles can be created via the RHOAI dashboard or as YAML for GitOps integration:

```yaml
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  annotations:
    opendatahub.io/dashboard-feature-visibility: '["model-serving"]'
  name: small
  namespace: redhat-ods-applications
spec:
  description: MIG 2g.20gb for model serving
  displayName: small
  enabled: true
  identifiers:
    - defaultCount: 2
      displayName: CPU
      identifier: cpu
      maxCount: 4
      minCount: 1
      resourceType: CPU
    - defaultCount: 4Gi
      displayName: Memory
      identifier: memory
      maxCount: 8Gi
      minCount: 2Gi
      resourceType: Memory
    - defaultCount: 1
      displayName: nvidia.com/mig-2g.20gb
      identifier: nvidia.com/mig-2g.20gb
      maxCount: 2
      minCount: 1
      resourceType: Accelerator
  nodeSelector: {}
  tolerations: []
```

> **Note — AcceleratorProfiles are deprecated**
>
> `AcceleratorProfiles` will be replaced by `HardwareProfiles`. Use `HardwareProfiles` for all new configurations.

---

## Verify the Configuration with a Model Deployment

Use the ModelCar available at `oci://quay.io/redhat-ai-services/modelcar-catalog:granite-3.3-2b-instruct` to deploy a model and validate both GPU resource types.

Two models will be deployed:
- **`granite-3.3-2b-instruct`** — uses `nvidia.com/gpu` (time slicing, should run)
- **`granite-3.3-2b-instruct-mig`** — uses `nvidia.com/mig-2g.20gb` (MIG, will stay Pending without MIG)

**Steps:**

1. Create a new Project in OpenShift AI.
2. Create a `Data Connection` within the project.
3. Deploy the first model using the default GPU hardware profile (`nvidia.com/gpu`).
4. Deploy the second model using the `small` hardware profile (`nvidia.com/mig-2g.20gb`).

### Inspect Resource Requests

The MIG model pod will stay Pending. Inspect the resource request to confirm:

```bash
oc get pod granite-33-2b-instruct-mig-predictor-00001-deployment-<uuid> -n granite -oyaml \
  | grep -B 3 'nvidia.com/mig-2g.20gb: "1"'
```

Expected output:

```yaml
      limits:
        cpu: "2"
        memory: 4Gi
        nvidia.com/mig-2g.20gb: "1"
      requests:
        cpu: "2"
        memory: 4Gi
        nvidia.com/mig-2g.20gb: "1"
```

Because `nvidia.com/mig-2g.20gb` is not present in the cluster, the scheduler cannot place the pod and it remains **Pending**.

### Clean Up

```bash
oc delete project granite
```

---

## References

- Red Hat. *OpenShift Documentation, v4.19 — NVIDIA GPU Architecture.* [docs.redhat.com](https://docs.redhat.com/en/documentation/openshift_container_platform/4.19/html/hardware_accelerators/nvidia-gpu-architecture)
- Red Hat Developers. *Build and deploy a ModelCar container in OpenShift AI.* [developers.redhat.com](https://developers.redhat.com/articles/2025/01/30/build-and-deploy-modelcar-container-openshift-ai)
