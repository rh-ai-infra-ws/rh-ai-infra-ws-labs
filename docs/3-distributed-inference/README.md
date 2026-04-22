
# Module 3 - Distributed Inference

Distributed inference is the process of scaling and serving machine learning models—particularly massive Large Language Models (LLMs) with billions of parameters—across multiple hardware accelerators (such as GPUs) and cluster nodes in parallel. This approach is essential when a model is too large to fit into the memory of a single GPU or node, or when an application needs to handle high-scale, unpredictable real-world traffic efficiently. By distributing the inference workload, systems can significantly increase throughput, decrease latency, and maximize expensive hardware utilization. Key techniques and patterns used in distributed inference include:

**Model Serving Lifecycle**

The Model Serving Lifecycle covers both the Inner Loop (model creation) and the Outer Loop (model serving), specifically focusing on deployment, inference, and monitoring. **Deployment** involves configuring a model server, uploading the model to storage (S3, OCI, or URI), setting resource requirements, and choosing the deployment mode (Raw Deployment). The **Inference** process routes client requests to the model server, handles preprocessing, executes the model, and streams or returns predictions via HTTP REST or gRPC APIs. **Monitoring** is crucial for continuous performance and accuracy, tracking infrastructure metrics (CPU, throughput) and AI-specific metrics (data drift, model bias), often configured using the TrustyAI component. RHOAI uses two primary Kubernetes Custom Resource Definitions (CRDs): `ServingRuntime` to define the execution environment and container templates, and `InferenceService` to create the server, specify the model's location and format, and define its endpoints.

**Serving GenAI and Predictive models**

The section on Serving GenAI and Predictive models details RHOAI's platform options and KServe's capabilities for different AI workloads. The standard Model serving platform is recommended for production and serving large models like Large Language Models (LLMs) and Generative AI, while the NVIDIA NIM-model serving platform is available for models using NVIDIA Inference Microservices. KServe provides optimized backends for both types of AI, supporting multi-framework Predictive AI (TensorFlow, PyTorch) with advanced features like explainability (TrustyAI) and intelligent routing. For Generative AI, KServe integrates optimizations like vLLM and llm-d for high-performance LLM serving, offering GPU acceleration, model caching, and request-based autoscaling. RHOAI offers built-in inference runtimes, including OpenVINO (optimized for Intel hardware and predictive tasks like object detection) and vLLM (a fast, memory-efficient engine for LLM inference).

**LLMs Distributed Inference**

The course then shifts to **LLMs Distributed Inference**, focusing on scaling large language models using `llm-d`, a high-performance distributed serving stack optimized for Kubernetes and datacenter accelerators. `llm-d` functions as an intelligent scheduler that maximizes GPU throughput with features like prefix-cache aware routing, disaggregated serving (prefill and decode servers), and tiered KV prefix caching with CPU/storage offload. Deploying this distributed architecture requires the `LLMInferenceService` CRD, which replaces the default `InferenceService` and creates a scheduler and inference pool. This advanced setup necessitates prerequisite operators, including the Cert-manager, Red Hat Connectivity Link, and Red Hat Leader Worker Set Operator, to manage authentication, networking, and sharding across multiple nodes.

**Benchmark**

The final module covers **Benchmark** frameworks for evaluating LLM performance. For general LLM testing, the course introduces **GuideLLM**, a platform that evaluates language models under realistic, end-to-end workloads. GuideLLM captures granular metrics like latency and token-level statistics, generates configurable traffic patterns, and produces standardized reports for capacity planning. For specifically benchmarking the `llm-d` stack, the presentation details **llm-d-benchmark**, an automated workflow that provides declarative infrastructure and workload configurations, allowing for structured experiments and multi-harness testing to ensure deployment reproducibility.

## Prerequisites

The core focus of this laboratory is to test distributed inference capabilities, which necessitates the deployment of the following required assets:

* Red Hat Openshift 4.20
* Red Hat Openshift AI 3.3.2
* Accelerators Configured (NVIDIA Operator + gpu-cluster-policy)
* Node Feature Discovery Operator + nfd-instance
* Red Hat OpenShift Service Mesh 3.1.0 + openshift-gateway control plane

> The required components are fully available within the "Red Hat OpenShift AI 3" catalog item on demo.redhat.com and it is possible to find the content slides in the following [link](https://docs.google.com/presentation/d/1huYUx2eKdKCgHiuiWHNJM_-3IHhdsR022Sm_Yug5Kf0/edit?usp=sharing) 