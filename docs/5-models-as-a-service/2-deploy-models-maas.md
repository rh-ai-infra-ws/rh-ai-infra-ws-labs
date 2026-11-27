# 🚀 Deploy Models with Models-as-a-Service

This lab deploys a generative model on OpenShift AI and publishes it to Models-as-a-Service (MaaS). A subscription and an authorization policy then give a user group a token limit for that model.

## What this procedure creates

The model is an `LLMInferenceService`. MaaS does not discover it until you also create:

- **MaaSModelRef**: registers the inference service with MaaS
- **MaaSSubscription**: gives a group a token limit for that model
- **MaaSAuthPolicy**: allows that group to call the model through the gateway

This group is `system:authenticated:oauth`. Every user who logged in with `oc login` is a member, including the account you use for `oc whoami -t`. A service-account token is not a member of that group.

?> 📎 The gateway hostname is `maas.<cluster-domain>`, from the previous lab. Inference calls use the URL on the `LLMInferenceService`, which includes the project and model name. A request to `https://maas.<cluster-domain>/v1/chat/completions` does not match a route.

## Create the project

1. Create the project that will host the model:

```bash
oc new-project project01-model-deploy
```

If the project already exists, the command reports that. Continue with the next step.

## Deploy the model

1. Create an `LLMInferenceService` for `qwen2.5-0.5b-instruct`. It uses the CPU vLLM image and attaches to `maas-default-gateway`:

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
          - "--dtype=bfloat16"
          - "--max-model-len=8192"
          - "--max-num-seqs=8"
          - "--enable-force-include-usage"
          - "--enforce-eager"
        env:
          - name: VLLM_CPU_KVCACHE_SPACE
            value: "2"
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

?> 📎 The container arguments keep the CPU footprint small: `bfloat16` weights, a context limit of 8192, at most 8 concurrent sequences, and no CUDA graph capture. `VLLM_CPU_KVCACHE_SPACE` reserves 2 GiB for the KV cache.

## Verify the model

The first start downloads the model image. Wait until `READY` is `True` before you create the MaaS objects or call the API.

1. Check the inference service:

```bash
oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy
```

Expected output:

```text
NAME              URL                                                                                  READY   REASON   AGE
qwen25-05b-llmd   https://maas.apps.<cluster-domain>/project01-model-deploy/qwen25-05b-llmd           True             5m
```

2. Check the pods:

```bash
oc get pods -n project01-model-deploy -l app.kubernetes.io/name=qwen25-05b-llmd
```

Expected output:

```text
NAME                                                       READY   STATUS    RESTARTS   AGE
qwen25-05b-llmd-kserve-<id>                                2/2     Running   0          5m
qwen25-05b-llmd-kserve-router-scheduler-<id>               1/1     Running   0          5m
```

?> ✅ Both pods must be `Running` and fully ready. The model pod has two containers.

## Create the MaaS objects

1. Register the inference service:

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

2. Create a subscription for `system:authenticated:oauth`. The limit is 5,000,000 tokens per minute:

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

3. Allow that same group to call the model:

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

## Verify the MaaS objects

1. Wait until the model reference is `Ready`:

```bash
oc get maasmodelref qwen25-05b-llmd -n project01-model-deploy
```

Expected output:

```text
NAME              PHASE   ENDPOINT                                         HTTPROUTE                      GATEWAY                AGE
qwen25-05b-llmd   Ready   https://maas.apps.<cluster-domain>/              qwen25-05b-llmd-kserve-route   maas-default-gateway   1m
```

`Pending` means the HTTP route is not attached yet. Run the command again. Do not call the model while the phase is `Pending`.

2. Confirm the subscription is `Active`:

```bash
oc get maassubscription qwen25-05b-llmd-demo -n models-as-a-service
```

Expected output:

```text
NAME                   PHASE    PRIORITY   AGE
qwen25-05b-llmd-demo   Active   0          1m
```

3. Confirm the authorization policy is `Active`:

```bash
oc get maasauthpolicy qwen25-05b-llmd-auth -n models-as-a-service
```

Expected output:

```text
NAME                   PHASE    AGE
qwen25-05b-llmd-auth   Active   1m
```

## Call the model with an OpenShift token

`oc whoami -t` returns your user token. The gateway sends any bearer token that does not start with `sk-oai-` through Kubernetes TokenReview. Your user is in `system:authenticated:oauth`, so the subscription and the authorization policy both allow the call.

1. Send a chat request to the inference service URL:

```bash
MODEL_URL=$(oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy -o jsonpath='{.status.url}')
TOKEN=$(oc whoami -t)

curl -ksS -X POST "${MODEL_URL}/v1/chat/completions" \
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

?> ✅ The response is a JSON chat completion. `choices[0].message.content` contains the answer, and `usage` reports the token counts. The wording of the answer changes between calls:

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

## Call the model with an API Key

Create the key with your OpenShift token. The key is bound to `qwen25-05b-llmd-demo`. Use the key on the inference URL from the previous section. Do not send it to `${HOST}/v1/chat/completions` or to `${HOST}/maas-api/v1/models`: the first path has no route, and the models API requires identity headers that this gateway does not set for an API key.

1. Create a key that expires in one hour:

```bash
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
HOST="https://maas.${CLUSTER_DOMAIN}"

API_KEY=$(curl -ksS -X POST "${HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -d '{"name":"workshop-key","expiresIn":"1h","subscription":"qwen25-05b-llmd-demo"}' | jq -er .key)

echo "${API_KEY}" | grep -q '^sk-oai-' && echo "API key created"
```

?> ✅ The command prints `API key created`. The plaintext key is shown only by this response. Store it if you need it again. A value of `null` means the create request failed. Print the curl response without `jq -er .key` and correct the error before continuing.

2. Call the model with that key:

```bash
MODEL_URL=$(oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy -o jsonpath='{.status.url}')

curl -ksS -X POST "${MODEL_URL}/v1/chat/completions" \
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

?> ✅ A successful response has the same shape as the OpenShift-token call: `object` is `chat.completion`, and `choices[0].message.role` is `assistant`.
