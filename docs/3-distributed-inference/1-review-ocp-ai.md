# Review OpenShift AI Components

This module reviews the end-to-end technical workflow for establishing a robust model-serving architecture within Red Hat OpenShift AI (RHOAI). It begins with the foundational infrastructure, detailing the necessary Operators (such as the GPU and Node Feature Discovery operators) and Data Science Project (DSP) configurations required to prep the environment for machine learning workloads.

## Red Hat OpenShift AI — Operators and requirements

Review the operator installation and OpenShift Data Science Project configuration following the next steps:

1. Open the [OpenShift console](https://console-openshift-console.<CLUSTER_DOMAIN>) and confirm that the OpenShift AI operator is installed: **Ecosystem** → **Installed Operators** → **Red Hat OpenShift AI**

![openshift-ai-operator.png](images/openshift-ai-operator.png)

2. In addition to the console, verify the operator by checking its pods in the target namespace:

```bash
oc get pods -n redhat-ods-operator
```

* A block similar to the following is expected to appear...

```bash
NAME                              READY   STATUS    RESTARTS         AGE
rhods-operator-58775b456f-5s7b9   1/1     Running   0                44h
rhods-operator-58775b456f-l2fv9   1/1     Running   0                44h
rhods-operator-58775b456f-qknw7   1/1     Running   0                44h
```

## Red Hat OpenShift AI — DSP configuration

Review the OpenShift Data Science configuration for KServe following the next procedure.

1. Confirm that KServe is set to `Managed`:

```bash
oc get dsc -n redhat-ods-operator default-dsc -o jsonpath='{.spec.components.kserve.managementState}'
```

!> `managementState` must be **Managed** to enable KServe.

For field documentation for KServe under `dsc.spec.components`, run:

```bash
oc explain dsc.spec.components.kserve
```

2. When `managementState` is `Managed`, the KServe controller manager is active. Verify that the controller pods are running:

```bash
oc get pod -n redhat-ods-applications -l app.kubernetes.io/name=kserve-controller-manager
```

* A block similar to the following is expected to appear...

```bash
NAME                                         READY   STATUS    RESTARTS   AGE
kserve-controller-manager-7d4cbdff89-cb9wz   1/1     Running   0          44h
```

?>It will be reviewed in next sections but it is important to take special attention to this pod. It represents the Control Plane of our inference services.

## `serving.kserve.io/ServingRuntime` & `serving.kserve.io/InferenceService` 

The core of the model serving is focuses on the KServe stack, specifically the interaction between the ServingRuntime Custom Resource Definition (CRD), which defines the execution environment or "template" for models and the InferenceService CRD, which acts as the primary orchestrator for deploying a specific model version. 

By examining these components, the review explains how the Predictor, Transformer, and Explainer work in tandem to provide scalable, high-performance inference endpoints that abstract away the underlying complexity of the Kubernetes networking and storage layers.

1. Please create a `ServingRuntime` to make available a new runtime for deploying models and verify the respective configuration:

```bash
cat << 'EOF' | oc apply -n <USER_NAME> -f-
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  annotations:
    opendatahub.io/apiProtocol: REST
    opendatahub.io/recommended-accelerators: '["nvidia.com/gpu"]'
    opendatahub.io/runtime-version: v3.2.4
    opendatahub.io/serving-runtime-scope: global
    openshift.io/display-name: RHAIIS (vLLM) NVIDIA GPU ServingRuntime for KServe
    openshift.io/template-display-name: RHAIIS (vLLM) NVIDIA GPU ServingRuntime for KServe
    opendatahub.io/template-name: rhaiis-cuda-template
  labels:
    opendatahub.io/dashboard: "true"
  name: rhaiis-cuda
spec:
  annotations:
    prometheus.io/path: /metrics
    prometheus.io/port: "8000"
  containers:
  - args:
    - --port=8000
    - --model=/mnt/models
    - --served-model-name={{.Name}}
    - --max-model-len=16000
    - --disable-uvicorn-access-log
    command:
    - python
    - -m
    - vllm.entrypoints.openai.api_server
    env:
    - name: HF_HOME
      value: /tmp/hf_home
    - name: HF_HUB_OFFLINE
      value: "1"
    - name: VLLM_NO_USAGE_STATS
      value: "1"
    image: registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.4
    name: kserve-container
    ports:
    - containerPort: 8000
      protocol: TCP
  multiModel: false
  supportedModelFormats:
  - autoSelect: true
    name: vLLM
EOF
```

2. Verify the runtime was properly created:

```bash
oc get ServingRuntime -n <USER_NAME>
```

* A block similar to the following is expected to appear...

```bash
NAMESPACE        NAME                   DISABLED   MODELTYPE   CONTAINERS         AGE
<USER_NAME>         rhaiis-cuda                       vLLM        kserve-container   118s
```

3. Once the runtime is created, the next step is to create an `InferenceService`. To begin, create a new namespace in Openshift:

```bash
oc new-project <USER_NAME>
```

4. As mentioned, it is time to create the inference service that links a specific model with a runtime, defining the main configuration for the service. Please create the following example and verify the configuration:

```bash
cat << 'EOF' | oc apply -n <USER_NAME> -f -
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama-vllm-single
  annotations:
    openshift.io/display-name: Llama 3.1 8B FP8 Single
    security.opendatahub.io/enable-auth: "false"
    serving.kserve.io/deploymentMode: RawDeployment
    opendatahub.io/hardware-profile-name: gpu-profile
    opendatahub.io/hardware-profile-namespace: redhat-ods-applications
spec:
  predictor:
    automountServiceAccountToken: false
    maxReplicas: 1
    minReplicas: 1
    deploymentStrategy:
      type: Recreate
    model:
      modelFormat:
        name: vLLM
      name: ''
      resources:
        limits:
          cpu: '4'
          memory: 8Gi
          nvidia.com/gpu: '1'
        requests:
          cpu: '4'
          memory: 8Gi
          nvidia.com/gpu: '1'
      runtime: rhaiis-cuda
      storageUri: 'oci://registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5'
EOF
```

5. Verify the inference service was properly created:

```bash
oc get InferenceService -n <USER_NAME>
```

* A block similar to the following is expected to appear...

```bash
NAMESPACE        NAME                   DISABLED   MODELTYPE   CONTAINERS         AGE
<USER_NAME>         rhaiis-cuda                       vLLM        kserve-container   118s
```

?> It takes a few minutes for the model to load and the state to change to "Ready". Leave this running in your terminal and continue reading the concepts below. We’ll come back to test the deployment at the end of this module.

## Inference service components

When an `InferenceService` is created, it is possible to see specific components and signal that represent the new inference service creation. Please follow next steps to understand the staff behind the scenes.

1. Review inference services pods:

```bash
oc get pods -n <USER_NAME>
```

* A block similar to the following is expected to appear...

```bash
NAME                                           READY   STATUS    RESTARTS   AGE
llama-vllm-single-predictor-6b98f867cc-6kf2k   2/2     Running   0          5m
```

2. Open the [OpenShift console](https://console-openshift-console.<CLUSTER_DOMAIN>) and see also the containers inside the pod: **Worloads** → **Pods** → **Logs (tab)**

![openshift-ai-inferenceservice-pod.png](images/openshift-ai-inferenceservice-pod.png)

It is important to review the different containers and their mission:

?> `modelcar-init` -> The primary goal of the modelcar-init container is to pre-fetch and extract model files from an Open Container Initiative (OCI) registry before the model serving pod fully starts (Note that the modelcar-init container is only present for deployments using OCI registries; it is not used if your model is stored in an S3-compatible object storage or a Persistent Volume Claim (PVC), as those methods allow the main container to access the model files directly)

?> `kserver-container` -> The primary goal of the kserve-container is to act as the main container within a KServe inference service pod. Its core responsibilities are to load the AI model into memory and run the actual inference server runtime, such as vLLM, OpenVINO, or custom runtimes.

?> `modelcar` -> The primary goal of the modelcar container is to act as a sidecar that provides the main inference server (the kserve-container) with continuous access to the model files extracted from an Open Container Initiative (OCI) image throughout the entire lifetime of the pod (The modelcar sidecar is exclusively used when you deploy models stored as OCI container images. If you deploy a model from S3-compatible object storage or a Persistent Volume Claim (PVC), this container is not present).

3. Finally, test the new inference service:

* Port forward to the pod

```bash
oc port-forward deployment/llama-vllm-single-predictor 8000:8000 &
```
* Check the list of models:

```bash
curl http://localhost:8000/v1/models | jq .
```

Example output:

```bash
{
  "object": "list",
  "data": [
    {
      "id": "llama-vllm-single",
      "object": "model",
      "created": 1776872323,
      "owned_by": "vllm",
      "root": "/mnt/models",
      "parent": null,
      "max_model_len": 16000,
      "permission": [
        {
          "id": "modelperm-ea7f91dd59f0432fa3317729142015b7",
          "object": "model_permission",
          "created": 1776872323,
          "allow_create_engine": false,
          "allow_sampling": true,
          "allow_logprobs": true,
          "allow_search_indices": false,
          "allow_view": true,
          "allow_fine_tuning": false,
          "organization": "*",
          "group": null,
          "is_blocking": false
        }
      ]
    }
  ]
}
```

* Finally, test inference:

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-vllm-single",
    "prompt": "San Francisco is a",
    "max_tokens": 50,
    "temperature": 0.7
  }' | jq .
```

Example output:

```bash
...
  "model": "llama-vllm-single",
  "choices": [
    {
      "index": 0,
      "text": " top tourist destination in the United States, attracting millions of visitors each year. From its iconic Golden Gate Bridge to its vibrant neighborhoods like Haight-Ashbury and Fisherman's Wharf, there's no shortage of things to see and do in this",
...
```

!> It is important to clean-up the created content to continue

```bash
oc delete InferenceService llama-vllm-single -n <USER_NAME>
```
