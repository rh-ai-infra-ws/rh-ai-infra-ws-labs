# 🚀 Deploy Models with Models-as-a-Service

This lab guides you through deploying generative AI models through the OpenShift AI and publishing them to Models-as-a-Service (MaaS) so that administrators can enforce subscription-based quota and token limits.

## Understanding MaaS Model Publishing

When you deploy a model and publish it to MaaS, the deployment wizard creates:

- **LLMInferenceService**: The actual inference server that handles model serving
- **MaaSModelRef**: A custom resource that registers the model with MaaS for subscription assignment

After publishing, administrators must create a **subscription** and **authorization policy** to make the model accessible to user groups.

> **Note:** Models published to MaaS use the LLMInferenceService architecture, which is designed for large language models and integrates with the MaaS gateway for subscription-based quota enforcement. The legacy deployment method uses traditional KServe InferenceService resources with serving runtimes.

## Deploy a Model via Dashboard

In this section, you will deploy a generative AI model using the OpenShift AI dashboard wizard and publish it to MaaS.

### Using the Dashboard Wizard

1. Log in to the OpenShift AI dashboard.
2. In the left navigation menu, click **Projects**.
3. Click **Create Project** and type the name `project01-model-deploy`.

### Deploy the model using LLM-D

1. Create the LLMInferenceService using the vLLM optimized for CPU and referencing the model `qwen2.5-0.5b-instruct`.


```bash
cat << 'EOF' | oc apply -f-
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: qwen25-05b-llmd
  namespace: project01-model-deploy
  labels:
    app.kubernetes.io/part-of: llm-d
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8000"
    prometheus.io/path: "/metrics"
spec:
  model:
    uri: "oci://quay.io/redhat-ai-services/modelcar-catalog:qwen2.5-0.5b-instruct"
    name: qwen25-05b-llmd
  router:
    scheduler: {}
    route: {}
    gateway:
      refs:
        - name: maas-default-gateway
          namespace: openshift-ingress
  template:
    containers:
      - name: main
        image: registry.redhat.io/rhaii/vllm-cpu-rhel9@sha256:0ec0f5ff9a5ff263f58db0cfb19e3d7eef24b94c2d8629fe5145f5a8747e28c9
        args:
          - "--dtype=bfloat16"       # half the weight footprint vs float32
          - "--max-model-len=8192"   # cap context (default would be 32768)
          - "--max-num-seqs=8"       # cap concurrent sequences (default 256); also makes requests
                                     # queue sooner so the KEDA num_requests_waiting trigger fires
          - "--enable-force-include-usage"  # always emit token-usage in responses (MaaS telemetry)
          - "--enforce-eager"        # skip graph capture / extra buffers
        env:
          - name: VLLM_CPU_KVCACHE_SPACE
            value: "2"               # GiB reserved for the CPU KV cache
          - name: OMP_NUM_THREADS
            value: "4"
          - name: VLLM_CPU_OMP_THREADS_BIND
            value: "0-3"
        resources:
          requests:
            cpu: "1"
            memory: 2Gi
          limits:
            cpu: "4"
            memory: 8Gi
EOF
```

## Verify Model Deployment

After deploying your model, verify that all components are properly configured.

### Verify LLMInferenceService

1. Check the status of your LLMInferenceService:

```bash
oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy
```

Expected output:

```text
NAME              URL                                                                                             READY   REASON   AGE
qwen25-05b-llmd   https://maas.apps.cluster-qnsf7.dyn.redhatworkshops.io/project01-model-deploy/qwen25-05b-llmd   True             4m45s
```

?> ✅ The **READY** column should show **True**.

2. Verify that the inference pods are running:

```bash
oc get pods -n project01-model-deploy -l app.kubernetes.io/name=qwen25-05b-llmd
```

Expected output:

```text
NAME                                                       READY   STATUS    RESTARTS   AGE
qwen25-05b-llmd-kserve-5446bcdcb7-q9tdt                    2/2     Running   0          11m
qwen25-05b-llmd-kserve-router-scheduler-5b4c54fbf4-lx6m6   1/1     Running   0          11m
```

## Create the respective MaaS Objects

1. Create the MaaSModelRef

```bash
cat << 'EOF' | oc apply -f-
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata:
  name: qwen25-05b-llmd
  namespace: project01-model-deploy
spec:
  modelRef:
    kind: LLMInferenceService
    name: qwen25-05b-llmd
EOF
```

2. Create the MaaSSubscription

```bash
cat << 'EOF' | oc apply -f-
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata:
  name: qwen25-05b-llmd-demo
  namespace: models-as-a-service
spec:
  owner:
    groups:
      - name: system:authenticated:oauth
  tokenMetadata:
    costCenter: na
    organizationId: na
  modelRefs:
    - name: qwen25-05b-llmd
      namespace: project01-model-deploy
      tokenRateLimits:
        - limit: 5000000
          window: 1m
EOF
```

3. Create the MaaSAuthPolicy

```bash
cat << 'EOF' | oc apply -f-
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata:
  name: qwen25-05b-llmd-auth
  namespace: models-as-a-service
spec:
  subjects:
    groups:
      - name: system:authenticated:oauth
  modelRefs:
    - name: qwen25-05b-llmd
      namespace: project01-model-deploy
EOF
```

## Verify MaaS Objects

1. Verify that the MaaSModelRef was created:

```bash
oc get maasmodelref qwen25-05b-llmd -n project01-model-deploy
```

Expected output:

```text
NAME              PHASE     ENDPOINT   HTTPROUTE                      GATEWAY                AGE
qwen25-05b-llmd   Pending              qwen25-05b-llmd-kserve-route   maas-default-gateway   7s
```

2. Verify that the MaaSAuthPolicy was created:

```bash
oc get maassubscription qwen25-05b-llmd-demo -n models-as-a-service
```

Expected output:

```text
NAME                   PHASE    PRIORITY   AGE
qwen25-05b-llmd-demo   Active   0          7m25s
```

3. Verify that the MaaSAuthPolicy was created:

```bash
oc get maasauthPolicy qwen25-05b-llmd-auth -n models-as-a-service
```

Expected output:

```text
NAME                   PHASE    AGE
qwen25-05b-llmd-auth   Active   8m25s
```

?> 📎 Look for the `status.conditions` sections to verify the model is properly registered with MaaS.

### Test Model Endpoint (Openshift User Token based)

Following this setup, we should proceed to validate the model endpoint leveraging MaaS alongside OpenShift user tokens for authentication and authorization.

```bash
# Get the model URL
MODEL_URL=$(oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy -o jsonpath='{.status.url}')

# Get your token
TOKEN=$(oc whoami -t)

# Test the model endpoint
curl -k -X POST "${MODEL_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen25-05b-llmd",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is OpenShift?"}
    ],
    "max_tokens": 100,
    "temperature": 0.7
  }' | jq .
```

Expected output:

```yaml
{
  "id": "chatcmpl-f405f395-2516-4988-ad9d-12998fd50b25",
  "object": "chat.completion",
  "created": 1795692587,
  "model": "qwen25-05b-llmd",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "OpenShift is an open-source container orchestration platform developed by Red Hat. It provides a comprehensive set of tools and services for deploying, managing, and scaling applications on Kubernetes. OpenShift allows developers to create, manage, and deploy containerized applications using a variety of programming languages, including Go, Java, Python, Ruby, and Node.js. The platform also supports various deployment strategies such as rolling updates, auto-scaling, and automatic scaling based on demand. Additionally, itoffers features like version control",
        "refusal": null,
        "annotations": null,
        "audio": null,
        "function_call": null,
        "tool_calls": [],
        "reasoning": null
      },
      "logprobs": null,
      "finish_reason": "length",
      "stop_reason": null,
      "token_ids": null
    }
  ],
  "service_tier": null,
  "system_fingerprint": null,
  "usage": {
    "prompt_tokens": 24,
    "total_tokens": 124,
    "completion_tokens": 100,
    "prompt_tokens_details": null
  },
  "prompt_logprobs": null,
  "prompt_token_ids": null,
  "kv_transfer_params": null
}
```

?> ✅ A successful response returns JSON with the model's completion.

!> ⚠️ **Note:** Once subscriptions and authorization policies are configured, users will access models through the MaaS gateway with enforced token limits.


### Test Model Endpoint (API Key based)

Now, we should also validate the model endpoint leveraging MaaS alongside API Key for authentication and authorization.

1. Generate the respective API Key:

```bash
# Get the model URL
MODEL_URL=$(oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy -o jsonpath='{.status.url}')

# Get a new API Key
API_KEY=$(curl -sSk -X POST "${HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -d '{"name":"workshop-key","expiresIn":"1h","subscription":"qwen25-05b-llmd-demo"}' | jq -r .key)
```

2. Test the model endpoint using the created API Key:

```bash
# Test the model endpoint
curl -k -X POST "${MODEL_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen25-05b-llmd",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is OpenShift?"}
    ],
    "max_tokens": 100,
    "temperature": 0.7
  }' | jq .
```

Expected output:

```yaml
{
  "id": "chatcmpl-48fb6385-dcf0-4a83-91d3-ac676694cfdb",
  "object": "chat.completion",
  "created": 1795692188,
  "model": "qwen25-05b-llmd",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "OpenShift is an open-source container orchestration platform developed by Red Hat. It provides a simple and intuitive way to manage and deploy applications across multiple Kubernetes clusters. OpenShift supports various programming languages, including Java, Python, Go, and Ruby, making it easy for developers to build and deploy applications quickly and easily.\nSome key features of OpenShift include:\n\n  * Containerized applications: Applications can be built and deployed using Docker images, which are packaged as containers that run on Kubernetes clusters.\n  *",
        "refusal": null,
        "annotations": null,
        "audio": null,
        "function_call": null,
        "tool_calls": [],
        "reasoning": null
      },
      "logprobs": null,
      "finish_reason": "length",
      "stop_reason": null,
      "token_ids": null
    }
  ],
  "service_tier": null,
  "system_fingerprint": null,
  "usage": {
    "prompt_tokens": 24,
    "total_tokens": 124,
    "completion_tokens": 100,
    "prompt_tokens_details": null
  },
  "prompt_logprobs": null,
  "prompt_token_ids": null,
  "kv_transfer_params": null
}
```