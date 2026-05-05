# GPU As A Service

GPU-as-a-Service is a fundamental approach for maximizing the efficiency and security of specialized AI hardware. It facilitates secure and effective multi-tenancy by centralizing resources and utilizing advanced scheduling mechanisms.

- Hardware Profiles / Resource Allocation: Once the hardware is detected and the drivers/plugins are in place, workloads need a mechanism to request and utilize these resources. Hardware Profiles (often implemented via Custom Resource Definitions or configuration within the accelerator operator) facilitate the allocation of specific accelerator resources for AI workloads. This ensures that different types of workloads (e.g., a massive training job requiring multiple high-end GPUs versus a small inference service needing a fraction of a single GPU) can be scheduled and run efficiently with guaranteed resource access. This mechanism is crucial for multi-tenancy and resource governance in an AI/ML platform.

- GPU Time Slicing or Multi-Instance GPU (MIG): These techniques enables effective sharing of a single physical GPU by dynamically alternating processing time slots among different workloads or, MIG, offering superior resource and fault isolation by providing hardware-based partitioning of the GPU.

- Enhanced Scheduling: Intelligent scheduling tools like Kueue further optimize resource management. Kueue actively manages resource quotas and prioritizes mission-critical workloads, ensuring optimal utilization and guaranteed access for a diverse range of AI/ML tasks.

## Intelligent GPU orchestration

A key strategy is needed to prevent non-AI workloads from inadvertently using GPU nodes, while also allowing users to select specific hardware profiles (such as an NVIDIA L4 or A100) to match their model's computational demands. Red Hat OpenShift, together with Red Hat OpenShift AI, provides the necessary mechanisms—including node taints, tolerations, custom labels and hardware profiles to define and enforce a solution for this challenge.


## Multi-Instance GPU (MIG)

## Hardware Profiles

## Intelligent Scheduling


