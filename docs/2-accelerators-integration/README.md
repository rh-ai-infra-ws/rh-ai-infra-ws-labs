# 🏎️ Accelerators Integration

The efficient execution of modern Artificial Intelligence (AI) and Machine Learning (ML) workloads, such as Natural Language Processing (NLP), deep learning inference, and large-scale model training, is critically dependent on specialized hardware. Accelerators, most commonly Graphics Processing Units (GPUs), are essential to optimize performance for these computationally intensive tasks.

**Enabling Accelerators** 🔧

The lifecycle for integrating specialized AI hardware, like GPUs, into a Kubernetes cluster for AI/ML workloads begins with Node Feature Discovery (NFD), which detects hardware features and labels the cluster nodes, enabling the Kubernetes scheduler to correctly place workloads. Before use, the necessary operating system kernel modules (drivers) must be managed and loaded, often through mechanisms like Kubernetes Machine Management (KMM) or dedicated driver operators. A central role is also played by the GPU Provider Operator (e.g., NVIDIA GPU Operator), which automates the deployment and lifecycle of the complete software stack, including drivers and device plugins, ensuring the Kubernetes kubelet can expose resources (like [nvidia.com/gpu](https://nvidia.com/gpu)) to the container runtime. 

**GPU-As-A-Service** 👨🏼‍💻

GPU-as-a-Service is a crucial paradigm for maximizing the efficiency and security of specialized hardware within AI platforms, enabling secure and efficient multi-tenancy by centralizing resources and employing sophisticated scheduling mechanisms. Hardware Profiles or similar resource allocation mechanisms are implemented to allow workloads to efficiently request and utilize these exposed resources, which is crucial for multi-tenancy, resource governance, and guaranteeing access to specific accelerators. Techniques like GPU Time Slicing allow a single physical GPU to be shared effectively by alternating processing time slots, while Multi-Instance GPU (MIG) offers superior resource and fault isolation through hardware partitioning. This resource management is further enhanced by intelligent scheduling tools such as Kueue, which actively manages quotas and prioritizes mission-critical workloads, ensuring optimal utilization and guaranteed access for diverse AI/ML tasks. 

## Prerequisites

The core focus of this laboratory is to test distributed inference capabilities, which necessitates the deployment of the following required assets:

* Red Hat Openshift 4.21
> The required components are fully available within the "Red Hat OpenShift Container Platform Cluster (AWS)" catalog item on demo.redhat.com. Additionally, it is required to deploy an additional g6.12xlarge node. 
> Finally, it is possible to find the content slides in the following [link](https://docs.google.com/presentation/d/11MfHXDDc_Psr1WX8S1DpppxNvNibpYE3QfrHnRXv-q0/edit?usp=sharing) 
