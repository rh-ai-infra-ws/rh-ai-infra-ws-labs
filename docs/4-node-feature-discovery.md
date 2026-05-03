# ⚙️ Node Feature Discovery

Node Feature Discovery (NFD) is a Kubernetes operator that detects hardware features and system configuration, then labels nodes accordingly. For GPU workloads, NFD is a prerequisite for the NVIDIA GPU Operator — it labels GPU nodes with PCI vendor/device IDs (e.g., `feature.node.kubernetes.io/pci-10de.present=true`) that the GPU Operator uses to target the right nodes.

> Work in progress.
