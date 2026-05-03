# 📖 GPU as a Service — Introduction

This section explores GPU slicing and its application in workload prioritization. To fully grasp the significance of these topics, a solid understanding of workload sizing is essential. We begin by demonstrating the vRAM calculation required to serve an `ibm-granite/granite-3.3-2b-instruct` model — this example underscores why slicing GPUs into smaller units is critical for maximizing ROI.

---

## Model Size

vLLM (Red Hat Inference Server) uses [PagedAttention](https://arxiv.org/abs/2309.06180) to optimize GPU memory management and reduce the memory required to serve models. To further enhance GPU utilization and maximize ROI on expensive hardware, additional optimizations are crucial. The following calculation shows that **an entire GPU is not always necessary** — without slicing, this excess capacity is wasted.

Two main components consume GPU memory when serving a model:

- **Model Weights** — memory required to load the model's parameters into the GPU
- **KV Cache (Key-Value Cache)** — dynamic memory for attention keys and values for active sequences. vLLM's PagedAttention optimizes this, but it still consumes significant memory with high concurrency and long context lengths

### Model Weights

The `granite-3.3-2b-instruct` model has ~2.5B parameters. Memory usage depends on precision:

**FP16 / BF16 (Half Precision)** — 2 bytes per parameter, most common for inference:

```
2.5B parameters × 2 bytes = 5 GB
```

**INT8 (8-bit Quantization)** — 1 byte per parameter:

```
2.5B parameters × 1 byte = 2.5 GB
```

vLLM defaults to FP16 (or `bfloat16` if supported), so model weights consume approximately **5 GB**.

<details>
<summary>💡 Is <code>bfloat16</code> the same as <code>float16</code>?</summary>

`bfloat16` stands for **Brain Floating Point** format — a 16-bit floating-point type designed for deep learning.

- `float16` (FP16): higher precision (10-bit mantissa) but smaller numerical range (5-bit exponent)
- `bfloat16`: same 8-bit exponent as FP32 (wider range), but less precision (7-bit mantissa)

`bfloat16` is preferred for deep learning training because it handles larger values and avoids numerical instability without requiring the gradient scaling that FP16 often needs.

</details>

### KV Cache

KV Cache memory usage is dynamic and depends on:

- `max_model_len` — maximum sequence length (prompt + output). Longer = larger KV Cache
- **Attention heads and hidden dimensions** — model architecture parameters
- **Concurrent requests** — more concurrent requests = more KV Cache entries
- `gpu-memory-utilization` — vLLM parameter (default: 0.9 = 90% of GPU memory)

For a **2.5B model** processing ~2,048 tokens in FP16, the KV Cache is roughly **~0.16 GB per sequence** — but this grows quickly with longer contexts.

The most reliable source for architectural parameters is the model's `config.json` on Hugging Face: [granite-3.3-2b-instruct config](https://huggingface.co/ibm-granite/granite-3.3-2b-instruct/tree/main).

<details>
<summary>📄 granite-3.3-2b-instruct <code>config.json</code></summary>

```json
{
  "architectures": ["GraniteForCausalLM"],
  "hidden_size": 2048,
  "num_attention_heads": 32,
  "num_hidden_layers": 40,
  "num_key_value_heads": 8,
  "max_position_embeddings": 131072,
  "intermediate_size": 8192,
  "torch_dtype": "bfloat16",
  "vocab_size": 49159
}
```

</details>

<details>
<summary>🔢 Exact KV Cache size calculation</summary>

Using the parameters from `config.json`:

| Parameter | Value |
|---|---|
| Hidden size (`h`) | 2048 |
| Number of layers (`L`) | 40 |
| KV attention heads | 8 |
| Max context length | 131,072 tokens |
| KV data type size | 2 bytes (BF16) |

**Step 1 — head_dim:**
```
head_dim = hidden_size / num_attention_heads = 2048 / 32 = 64
```

**Step 2 — KV Cache per token (across all layers):**
```
KV Cache per token = 2 × L × num_kv_heads × head_dim × bytes
                   = 2 × 40 × 8 × 64 × 2
                   = 81,920 bytes/token  (~0.078 MiB/token)
```

**Step 3 — Total KV Cache at max context (single sequence):**
```
Total KV Cache = 81,920 bytes/token × 131,072 tokens
               = 10,737,418,240 bytes
               = ~10 GiB
```

This **10 GiB** is for a *single sequence* at the maximum 131k token context window.

**Total estimated GPU memory (FP16):**

| Component | Memory |
|---|---|
| Model weights (FP16) | ~5 GB |
| KV Cache (max single sequence) | ~10 GiB |
| **Total minimum** | **~15–16 GB** |

For comfortable production serving:
- **16 GB vRAM** — barely fits with strict concurrency and context limits
- **24 GB vRAM** (A10G, A100) — good headroom for concurrent requests
- **Quantization** (INT8 → 2.5 GB weights, INT4 → ~1.25 GB) — required for smaller GPUs

**Context length vs. concurrency trade-off:**

| Context length | KV Cache per request |
|---|---|
| 131,072 tokens (max) | ~10 GiB |
| 2,048 tokens (typical) | ~0.16 GB |

At max context, a single request's KV Cache is *twice the size of the model weights*.

</details>

A quick way to estimate KV Cache requirements: [gaunernst/kv-cache-calculator](https://huggingface.co/spaces/gaunernst/kv-cache-calculator) on Hugging Face.

---

## GPU Optimization — Why Slicing Matters

The example above shows ~16 GB vRAM for a single user at maximum context. Targeting **20 GB** to support a few concurrent queries: an **H100 with 80 GB** of vRAM can easily host this model — but leaves **60 GB unused**. That is a poor return on investment.

By using **MIG (Multi-Instance GPU)**, the H100 can be split into up to four independent 20 GB instances, serving **four models simultaneously** on the same physical GPU.

> See also the [GPU partitioning guide](https://github.com/rh-aiservices-bu/gpu-partitioning-guide) by `rh-aiservices-bu`.

<details>
<summary>🎛️ NVIDIA GPU Sharing Options — Full Comparison</summary>

### 1. Time-Slicing (Software-based)

The GPU scheduler allocates time slices to each process in round-robin fashion. Each process gets a turn to use the GPU.

**Pros:** Cost efficient, broad hardware compatibility, simple to configure via GPU Operator
**Cons:** No memory or fault isolation, context switching overhead, no fixed resource guarantees

---

### 2. Multi-Instance GPU (MIG)

Hardware-level partitioning (Ampere architecture and newer) — each MIG instance has dedicated compute cores, memory, and memory bandwidth.

**Pros:** Hardware isolation, predictable performance, fault isolation, dynamic partitioning
**Cons:** Only on Ampere/Hopper+ GPUs, coarse profiles, fixed allocation per instance, more complex setup

---

### 3. Multi-Process Service (MPS)

A CUDA feature that consolidates multiple CUDA contexts into a single server process, reducing context-switching overhead.

**Pros:** Improves utilization, reduces overhead vs. time-slicing, enables concurrent kernel execution
**Cons:** No memory protection, CUDA-only, not compatible with MIG in the GPU Operator

---

### 4. Default Exclusive Access

Each pod gets an entire physical GPU.

**Pros:** Simple, maximum performance for one workload, full isolation
**Cons:** Low utilization for small workloads, high cost at scale

---

### Summary

| Feature | Time-Slicing | MIG | MPS | Exclusive |
|---|---|---|---|---|
| Method | Software time sharing | Hardware partitioning | Context consolidation | Full GPU |
| Isolation | None | Hardware-enforced | None | Full |
| Predictable perf | Low | High | Medium | High |
| GPU utilization | High | High | High | Low |
| Hardware req. | All NVIDIA | Ampere/Hopper+ | Most NVIDIA | All NVIDIA |
| Use case | Small non-critical workloads | Mixed workloads needing isolation | Concurrent CUDA apps | Large critical workloads |
| Complexity | Medium | High | Medium | Low |

</details>

> **Tip — Combining MIG and Time-Slicing**
>
> You can enable time-slicing *within* a MIG instance. The MIG instance provides hardware isolation from other MIG instances, and time-slicing then allows multiple pods to share that specific MIG instance.

### Multi-Instance GPU (MIG)

MIG partitions a single physical GPU into multiple smaller, completely isolated, independent GPU instances — like slicing a large cake into individual portions that can be consumed independently.

The GPU cannot be split arbitrarily — there are predefined **MIG profiles** per GPU type. For the **H100**, for example, a valid configuration is `3× MIG 3g.40gb` + `1× MIG 1g.20gb`. See the official [H100 MIG Profiles](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/index.html#h100-mig-profiles) documentation for all options.

With this configuration, multiple models can be served in parallel, with smaller slices available for experimentation.

**Supported GPUs:** A30, A100, H100, H200, GH200, B200

MIG profiles are configured via the [NVIDIA GPU Operator ClusterPolicy](https://docs.nvidia.com/datacenter/cloud-native/openshift/latest/introduction.html).

---

## Fair Resource Sharing with Kueue

Building on optimized serving runtimes and MIG-sliced GPU utilization, [Kueue](https://kueue.sigs.k8s.io/docs/overview/) addresses fair resource sharing and workload prioritization across teams.

### Use Cases

**Use Case 1 — Enforcing Fair GPU Quotas (Preventing Resource Hogging)**

Team A's optimized serving runtimes could consume all available MIG-sliced GPU resources, leaving nothing for Team B's critical workloads. Kueue enforces per-team quotas to prevent this.

**Use Case 2 — Prioritizing Critical Workloads with Preemption**

When the cluster is under heavy load, new business-critical serving instances get stuck waiting because lower-priority experimental jobs (training, hyperparameter sweeps) hold the resources. Kueue preempts lower-priority work to make room.

**Use Case 3 — Burst Capacity for Sporadic High-Priority Jobs**

Some jobs (urgent retraining, large analytics) sporadically need to exceed a team's typical quota. Kueue allows borrowing unused quota from other teams for these bursts.

**Use Case 4 — Spot Instance Pricing Model**

Infrastructure providers can offer discounted GPU resources for on-demand/training jobs in exchange for preemption risk. Customers use unused GPU capacity at lower cost; higher-priority workloads can reclaim it when needed.

### Core Kueue Concepts

| Concept | Description |
|---|---|
| **ResourceFlavor** | Describes a set of nodes with specific hardware characteristics (e.g. A10G GPUs) |
| **ClusterQueue** | Cluster-scoped queue with resource quotas that consumes ResourceFlavors |
| **LocalQueue** | Namespace-scoped queue that points to a ClusterQueue |
| **WorkloadPriorityClass** | Defines priority levels for competing workloads |
