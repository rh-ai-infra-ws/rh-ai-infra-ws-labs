# 🎮 NVIDIA GPU Operator

This section uses the [NVIDIA GPU Operator on OpenShift](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/introduction.html) and demonstrates the configuration needed to enable GPU partitioning strategies. The primary focus is on **MIG (Multi-Instance GPU)** for production environments, with time slicing covered as a fallback for hardware that does not support MIG.

> **Lab context:** The GPUs available in this lab are two **AWS NVIDIA A10G Tensor Core GPUs** with 24 GB of memory per GPU. The A10G does *not* support MIG, so time slicing is used in hands-on exercises. The MIG sections reflect production best practice.

> **Note — Time slicing vs MIG for Model Serving**
>
> NVIDIA time slicing is a valuable strategy for serving vLLM models when you need to maximize GPU utilization on older or non-MIG capable hardware, have mixed or bursty workloads, or prioritize cost efficiency. For newer GPUs and scenarios demanding strong performance isolation and predictable resource allocation, **NVIDIA MIG is the preferred choice**.

## Introduction

Installing the NVIDIA GPU Operator on OpenShift involves a series of steps to ensure your cluster can properly utilize NVIDIA GPUs for accelerated workloads. The **Node Feature Discovery (NFD) Operator** must be installed first — it labels nodes so the GPU Operator can install the correct drivers on the right nodes.
