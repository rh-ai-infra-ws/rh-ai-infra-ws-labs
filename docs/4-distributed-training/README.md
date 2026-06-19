# 🏋️ Distributed Training

Distributed training runs machine learning workloads across multiple cluster nodes and accelerators in parallel. This approach is essential when models or datasets exceed the capacity of a single GPU or node, and when teams need to iterate faster on large-scale experiments.

**Distributed workloads infrastructure** ⚙️

Red Hat OpenShift AI integrates several components for distributed data processing and training:

- **Red Hat build of Kueue** — quota management and queueing so training jobs start when cluster resources are available.
- **KubeRay** — Ray clusters for Python-centric distributed compute workloads.
- **Kubeflow Trainer v2** — unified `TrainJob` API and pre-built `ClusterTrainingRuntime` templates (replaces framework-specific CRDs such as `PyTorchJob` from Training Operator v1).
- **CodeFlare SDK** and **Training Operator SDK** — included in select workbench images for defining jobs from notebooks or pipelines.

**Hands-on TrainJob workflow** 🚀

The lab exercises walk through enabling distributed training components in the `DataScienceCluster`, verifying different components availability, and submitting a minimal exercises for collective training tasks across GPUs.

## Prerequisites

This module builds on a cluster with OpenShift AI and GPU accelerators already configured:

* Red Hat OpenShift 4.21+
* Red Hat OpenShift AI 3.3+
* Accelerators configured (NVIDIA GPU Operator and cluster policy)
* Node Feature Discovery Operator installed

> The required components are fully available within the "Red Hat OpenShift AI 3" catalog item on demo.redhat.com and it is possible to find the content slides in the following [link](https://docs.google.com/presentation/d/1huYUx2eKdKCgHiuiWHNJM_-3IHhdsR022Sm_Yug5Kf0/edit?usp=sharing) 

!> If you are following the full laboratory using "Red Hat OpenShift Container Platform Cluster (AWS)", including "Accelerators Integration" chapter, it is required to create an additional g6.12xlarge node (2 in total)