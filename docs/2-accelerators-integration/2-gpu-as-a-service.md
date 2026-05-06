# 👨🏼‍💻 GPU As A Service

GPU-as-a-Service is a fundamental approach for maximizing the efficiency and security of specialized AI hardware. It facilitates secure and effective multi-tenancy by centralizing resources and utilizing advanced scheduling mechanisms.

- Hardware Profiles / Resource Allocation: Once the hardware is detected and the drivers/plugins are in place, workloads need a mechanism to request and utilize these resources. Hardware Profiles (often implemented via Custom Resource Definitions or configuration within the accelerator operator) facilitate the allocation of specific accelerator resources for AI workloads. This ensures that different types of workloads (e.g., a massive training job requiring multiple high-end GPUs versus a small inference service needing a fraction of a single GPU) can be scheduled and run efficiently with guaranteed resource access. This mechanism is crucial for multi-tenancy and resource governance in an AI/ML platform.

- GPU Time Slicing or Multi-Instance GPU (MIG): These techniques enables effective sharing of a single physical GPU by dynamically alternating processing time slots among different workloads or, MIG, offering superior resource and fault isolation by providing hardware-based partitioning of the GPU.

- Enhanced Scheduling: Intelligent scheduling tools like Kueue further optimize resource management. Kueue actively manages resource quotas and prioritizes mission-critical workloads, ensuring optimal utilization and guaranteed access for a diverse range of AI/ML tasks.

## Intelligent GPU orchestration

A key strategy is needed to prevent non-AI workloads from inadvertently using GPU nodes, while also allowing users to select specific hardware profiles (such as an NVIDIA L4 or A100) to match their model's computational demands. Red Hat OpenShift, together with Red Hat OpenShift AI, provides the necessary mechanisms—including node taints, tolerations, custom labels and hardware profiles to define and enforce a solution for this challenge.

The following procedure implement this strategy creating the required resources in the Openshift cluster:

1. First of all, it is required to reserve the worker nodes with GPUs for only AI workloads. For this goal, it is required to create the following taints:

```bash
oc get nodes
oc adm taint node <NODE_NAME> ai/workloads=Exists:NoSchedule --overwrite 
```

!> If any pod related to the NVIDIA operator is restarted, then it couldn't start again because of this tains. It is possible to try...😋
 
2. Modify the NVIDIA ClusterPolicy to include the respective tolerations in the pods deployed by the operator:

```bash
oc patch clusterpolicy gpu-cluster-policy -n nvidia-gpu-operator --type=merge -p '
{
  "spec": {
    "daemonsets": {
      "tolerations": [
        {
          "key": "ai/workloads",
          "operator": "Exists",
          "effect": "NoSchedule"
        }
      ]
    }
  }
}
'
```

Verify that the daemonsets are now running on the tainted node:

```bash
oc get pods -n nvidia-gpu-operator -o wide -w
```

!> All pods should be restarted to change tolerations in their definition

```text
NAME                                           READY   STATUS      RESTARTS   AGE   IP            NODE                                        NOMINATED NODE   READINESS GATES
gpu-feature-discovery-gp8bp                    0/1     Init:0/1    0          42s   10.129.0.33   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
gpu-operator-6d8475969f-lbfwl                  1/1     Running     0          21m   10.129.0.10   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-container-toolkit-daemonset-vrxt7       0/1     Init:0/1    0          42s   10.129.0.37   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-cuda-validator-vm4fq                    0/1     Completed   0          14m   10.129.0.25   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-dcgm-exporter-gz42l                     0/1     Init:0/1    0          42s   10.129.0.35   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-dcgm-zbznf                              0/1     Init:0/1    0          42s   10.129.0.36   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-device-plugin-daemonset-gpkm6           0/1     Init:0/1    0          42s   10.129.0.34   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-driver-daemonset-9.6.20260420-0-nmwmw   1/2     Running     0          89s   10.129.0.32   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-node-status-exporter-cffd5              1/1     Running     0          90s   10.129.0.31   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-operator-validator-kb9dm                0/1     Init:0/4    0          42s   10.129.0.38   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
```

?> A successful deployment shows a Running status after some minutes

```text
gpu-feature-discovery-gp8bp                    1/1     Running     0               4m34s   10.129.0.33   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
gpu-operator-6d8475969f-lbfwl                  1/1     Running     0               25m     10.129.0.10   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-container-toolkit-daemonset-vrxt7       1/1     Running     0               4m34s   10.129.0.37   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-cuda-validator-r947z                    0/1     Completed   1               2m23s   10.129.0.39   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-dcgm-exporter-gz42l                     0/1     Running     1 (2m17s ago)   4m34s   10.129.0.35   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-dcgm-zbznf                              1/1     Running     0               4m34s   10.129.0.36   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-device-plugin-daemonset-gpkm6           1/1     Running     0               4m34s   10.129.0.34   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-driver-daemonset-9.6.20260420-0-nmwmw   2/2     Running     0               5m21s   10.129.0.32   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-node-status-exporter-cffd5              1/1     Running     0               5m22s   10.129.0.31   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
nvidia-operator-validator-kb9dm                1/1     Running     0               4m34s   10.129.0.38   ip-10-0-15-196.us-east-2.compute.internal   <none>           <none>
```

## Multi-Instance GPU (MIG)

## Hardware Profiles

## Intelligent Scheduling


