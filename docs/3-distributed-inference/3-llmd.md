# LLMs Distributed Inference

In Red Hat OpenShift AI, distributed inference is powered by the llm-d open-source framework, which serves as an intelligent orchestration layer above model servers like vLLM.

## Create the LLMInferenceService

1. First of all a GatewayClass and a Gateway for the inference service should be created.

```bash
cat << 'EOF' | oc apply -f-
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-default
spec:
  controllerName: openshift.io/gateway-controller/v1
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: openshift-ai-inference
  namespace: openshift-ingress
spec:
  gatewayClassName: openshift-default
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
EOF
```

2. Create the llm-d configuration with 2 replicas and prefix-cache-aware routing:

[vLLM manifest LLM-d](_createllmd.md ':include')

3.  Once the LLMInferenceService shows Ready, get the inference URL:

```bash
# Get the llm-d inference URL
LLMD_URL=$(oc get llminferenceservice llama-llm-d -n <USER_NAME> -o jsonpath='{.status.url}')
echo "llm-d URL: ${LLMD_URL}"

# Test with a simple request
curl ${LLMD_URL}/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-3-1-8b-instruct-fp8",
    "prompt": "San Francisco is a",
    "max_tokens": 50,
    "temperature": 0.7
  }' | jq .
```