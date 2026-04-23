
```bash
cat << 'EOF' | oc apply -f-
apiVersion: serving.kserve.io/v1alpha1
kind: LLMInferenceService
metadata:
  name: llama-llm-d
  namespace: <USER_NAME>
  annotations:
    opendatahub.io/model-type: generative
    openshift.io/display-name: llama-llm-d
    security.opendatahub.io/enable-auth: 'false'
    prometheus.io/path: /metrics
    prometheus.io/port: "8000"
spec:
  replicas: 2
  model:
    uri: oci://registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5
    name: llama-3-1-8b-instruct-fp8
  router:
    scheduler:
      template:
        containers:
          - name: main
            env:
              - name: TOKENIZER_CACHE_DIR
                value: /tmp/tokenizer-cache
              - name: HF_HOME
                value: /tmp/tokenizer-cache
              - name: TRANSFORMERS_CACHE
                value: /tmp/tokenizer-cache
              - name: XDG_CACHE_HOME
                value: /tmp
            args:
              - '--cert-path'
              - /var/run/kserve/tls
              - --pool-group
              - inference.networking.x-k8s.io
              - '--pool-name'
              - '{{ ChildName .ObjectMeta.Name `-inference-pool` }}'
              - '--pool-namespace'
              - '{{ .ObjectMeta.Namespace }}'
              - '--zap-encoder'
              - json
              - '--grpc-port'
              - '9002'
              - '--grpc-health-port'
              - '9003'
              - '--secure-serving'
              - '--model-server-metrics-scheme'
              - https
              - --kv-cache-usage-percentage-metric
              - "vllm:kv_cache_usage_perc"
              - '--config-text'
              - |
                apiVersion: inference.networking.x-k8s.io/v1alpha1
                kind: EndpointPickerConfig
                plugins:
                - type: single-profile-handler
                - type: queue-scorer
                - type: active-request-scorer
                - type: prefix-cache-scorer
                schedulingProfiles:
                - name: default
                  plugins:
                  - pluginRef: queue-scorer
                    weight: 2
                  - pluginRef: active-request-scorer
                    weight: 2
                  - pluginRef: prefix-cache-scorer
                    weight: 3
            volumeMounts:
              - name: tokenizer-cache
                mountPath: /tmp/tokenizer-cache
              - name: cachi2-cache
                mountPath: /cachi2
        volumes:
          - name: tokenizer-cache
            emptyDir: {}
          - name: cachi2-cache
            emptyDir: {}
    route: {}
    gateway: {}
  template:
    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
    containers:
      - name: main
        env:
          - name: VLLM_ADDITIONAL_ARGS
            value: "--disable-uvicorn-access-log --max-model-len=16000"
        resources:
          limits:
            cpu: '4'
            memory: 8Gi
            nvidia.com/gpu: "1"
          requests:
            cpu: '4'
            memory: 8Gi
            nvidia.com/gpu: "1"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
            scheme: HTTPS
          initialDelaySeconds: 120
          periodSeconds: 30
          timeoutSeconds: 30
          failureThreshold: 5
EOF
```

Verify the LLM-D inference service was properly created:

```bash
oc get LLMInferenceService -n <USER_NAME>
```

* **Sample `oc get LLMInferenceService` output** (add `-o wide` for the prediction URL; column names can vary by CLI/operator version):

```text
NAME          URL                                                                                                   READY   REASON   AGE
llama-llm-d   http://a12c42c41273f426bbaed253b3739f9c-1239963383.us-east-2.elb.amazonaws.com/acidonpe/llama-llm-d   True             79m
```

?> It takes a few minutes for the model to load and the state to change to "Ready". Leave this running in your terminal and continue reading the concepts below. We’ll come back to test the deployment at the end of this module.