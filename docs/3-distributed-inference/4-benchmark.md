# Benchmark

In this module you’ll look at the vLLM metrics endpoint and run a short illustrative benchmark to see how an inference service responses in a high performance scenario.

## Deploy `llama-vllm` model

Create the inference service:

[vLLM manifest](_createvllm.md ':include')

## Explore vLLM metrics endpoint

1. vLLM exposes Prometheus metrics on its metrics port. Verify the service is exposing metrics:

```bash
oc port-forward deployment/llama-vllm-single-predictor 8000:8000 &
```

2. Access to Prometheus metrics:

```bash
curl http://localhost:8000/metrics | grep vllm | head -40
```

The following metrics can be shown in the logs:

```text
# HELP vllm:num_requests_running Number of requests in model execution batches.
# TYPE vllm:num_requests_running gauge
vllm:num_requests_running{engine="0",model_name="llama-vllm-single"} 0.0
# HELP vllm:num_requests_waiting Number of requests waiting to be processed.
# TYPE vllm:num_requests_waiting gauge
vllm:num_requests_waiting{engine="0",model_name="llama-vllm-single"} 0.0
# HELP vllm:kv_cache_usage_perc KV-cache usage. 1 means 100 percent usage.
# TYPE vllm:kv_cache_usage_perc gauge
vllm:kv_cache_usage_perc{engine="0",model_name="llama-vllm-single"} 0.0
# HELP vllm:prefix_cache_queries_total Prefix cache queries, in terms of number of queried tokens.
# TYPE vllm:prefix_cache_queries_total counter
vllm:prefix_cache_queries_total{engine="0",model_name="llama-vllm-single"} 0.0
# HELP vllm:prefix_cache_queries_created Prefix cache queries, in terms of number of queried tokens.
# TYPE vllm:prefix_cache_queries_created gauge
vllm:prefix_cache_queries_created{engine="0",model_name="llama-vllm-single"} 1.7769301736864316e+09
# HELP vllm:prefix_cache_hits_total Prefix cache hits, in terms of number of cached tokens.
# TYPE vllm:prefix_cache_hits_total counter
vllm:prefix_cache_hits_total{engine="0",model_name="llama-vllm-single"} 0.0
# HELP vllm:prefix_cache_hits_created Prefix cache hits, in terms of number of cached tokens.
# TYPE vllm:prefix_cache_hits_created gauge
vllm:prefix_cache_hits_created{engine="0",model_name="llama-vllm-single"} 1.7769301736864555e+09
# HELP vllm:num_preemptions_total Cumulative number of preemption from the engine.
# TYPE vllm:num_preemptions_total counter
vllm:num_preemptions_total{engine="0",model_name="llama-vllm-single"} 0.0
# HELP vllm:num_preemptions_created Cumulative number of preemption from the engine.
# TYPE vllm:num_preemptions_created gauge
vllm:num_preemptions_created{engine="0",model_name="llama-vllm-single"} 1.7769301736864786e+09
# HELP vllm:prompt_tokens_total Number of prefill tokens processed.
# TYPE vllm:prompt_tokens_total counter
vllm:prompt_tokens_total{engine="0",model_name="llama-vllm-single"} 0.0
# HELP vllm:prompt_tokens_created Number of prefill tokens processed.
# TYPE vllm:prompt_tokens_created gauge
vllm:prompt_tokens_created{engine="0",model_name="llama-vllm-single"} 1.7769301736864982e+09
# HELP vllm:generation_tokens_total Number of generation tokens processed.
# TYPE vllm:generation_tokens_total counter
vllm:generation_tokens_total{engine="0",model_name="llama-vllm-single"} 0.0
# HELP vllm:generation_tokens_created Number of generation tokens processed.
# TYPE vllm:generation_tokens_created gauge
vllm:generation_tokens_created{engine="0",model_name="llama-vllm-single"} 1.7769301736865187e+09
# HELP vllm:request_success_total Count of successfully processed requests.
```

?>Stop port-forward when ready executing `pkill -f "port-forward"`

## Run GuideLLM benchmark

This is a short, illustrative run—just 30 seconds per concurrency level—to show how a single GPU handles a large volume of concurrent requests. The goal here is simply to observe what happens when one GPU gets overloaded.

1. Execute a benchmark using a specific and repetible contiguration:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: batch/v1
kind: Job
metadata:
  name: guidellm-vllm-single
  namespace: <USER_NAME>
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: guidellm
          image: ghcr.io/vllm-project/guidellm:latest
          env:
            - name: GUIDELLM_TARGET
              value: "http://llama-vllm-single-predictor:8000/v1"
            - name: GUIDELLM_PROFILE
              value: concurrent
            - name: GUIDELLM_RATE
              value: "32,64"
            - name: GUIDELLM_OUTPUTS
              value: json,csv
            - name: GUIDELLM_MAX_SECONDS
              value: "30"
            - name: GUIDELLM_DATA
              value: "prompt_tokens=512,output_tokens=256"
            - name: GUIDELLM_PROCESSOR
              value: "RedHatAI/Meta-Llama-3.1-8B-Instruct-FP8-dynamic"
            - name: HF_HOME
              value: /tmp/hf-cache
            - name: TRANSFORMERS_CACHE
              value: /tmp/hf-cache
            - name: XDG_CACHE_HOME
              value: /tmp
          volumeMounts:
            - name: cache
              mountPath: /tmp/hf-cache
            - name: results
              mountPath: /results
      volumes:
        - name: cache
          emptyDir: {}
        - name: results
          emptyDir: {}
EOF
```

2. After running the Quick Start benchmark, GuideLLM writes several output files to the directory you specified. Each one focuses on a different layer of analysis, ranging from a quick on-screen summary to fully structured data for dashboards and regression pipelines. The console provides a lightweight summary with high-level statistics for each benchmark in the run. It's useful for quick checks to confirm that the server responded correctly, the load sweep completed, and the system behaved as expected. 

```bash
oc logs -f job/guidellm-vllm-single
```

Example of the output:

```text

╭─ Benchmarks ─────────────────────────────────────────────────────────────────╮
│ [1… conc… (… Req:    1.1 req/s,   20.57s Lat,    22.3 Conc,      33 Comp,  … │
│              Tok:  215.7 gen/s,  675.8 tot/s, 1578.2ms TTFT,   74.5ms ITL, … │
│ [1… conc… (… Req:    1.3 req/s,   28.36s Lat,    41.6 Conc,      44 Comp,  … │
│              Tok:  372.8 gen/s, 1168.0 tot/s, 2213.7ms TTFT,  102.5ms ITL, … │
╰──────────────────────────────────────────────────────────────────────────────╯
Generating... ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ (2/2) [ 0:01:12 < 0:00:00 ]

ℹ Run Summary Info
|============|==========|==========|======|======|======|=========|=========|=====|=========|========|=====|
| Benchmark  | Timings                              ||||| Input Tokens          ||| Output Tokens        |||
| Strategy   | Start    | End      | Dur  | Warm | Cool | Comp    | Inc     | Err | Comp    | Inc    | Err |
|            |          |          | Sec  | Sec  | Sec  | Tot     | Tot     | Tot | Tot     | Tot    | Tot |
|------------|----------|----------|------|------|------|---------|---------|-----|---------|--------|-----|
| concurrent | 10:13:04 | 10:13:34 | 30.0 | 0.0  | 0.0  | 18018.0 | 16926.0 | 0.0 | 8448.0  | 7786.0 | 0.0 |
| concurrent | 10:13:45 | 10:14:15 | 30.0 | 0.0  | 0.0  | 24024.0 | 32590.0 | 0.0 | 11264.0 | 5231.0 | 0.0 |
|============|==========|==========|======|======|======|=========|=========|=====|=========|========|=====|


ℹ Text Metrics Statistics (Completed Requests)
|============|=======|=======|========|========|=======|=======|========|========|========|========|=========|=========|
| Benchmark  | Input Tokens                 |||| Input Words                  |||| Input Characters                 ||||
| Strategy   | Per Request  || Per Second     || Per Request  || Per Second     || Per Request    || Per Second       ||
|            | Mdn   | p95   | Mdn    | Mean   | Mdn   | p95   | Mdn    | Mean   | Mdn    | p95    | Mdn     | Mean    |
|------------|-------|-------|--------|--------|-------|-------|--------|--------|--------|--------|---------|---------|
| concurrent | 546.0 | 546.0 | 125.1  | 912.4  | 427.0 | 433.0 | 98.3   | 714.3  | 2862.0 | 2938.0 | 657.7   | 4776.1  |
| concurrent | 546.0 | 546.0 | 2766.6 | 6359.5 | 428.0 | 431.0 | 2172.0 | 4980.8 | 2875.0 | 2948.0 | 14598.8 | 33446.9 |
|============|=======|=======|========|========|=======|=======|========|========|========|========|=========|=========|
| Benchmark  | Output Tokens                |||| Output Words                 |||| Output Characters                ||||
| Strategy   | Per Request  || Per Second     || Per Request  || Per Second     || Per Request    || Per Second       ||
|            | Mdn   | p95   | Mdn    | Mean   | Mdn   | p95   | Mdn    | Mean   | Mdn    | p95    | Mdn     | Mean    |
|------------|-------|-------|--------|--------|-------|-------|--------|--------|--------|--------|---------|---------|
| concurrent | 256.0 | 256.0 | 58.6   | 427.8  | 188.0 | 221.0 | 42.9   | 317.4  | 1217.0 | 1306.0 | 282.4   | 2007.1  |
| concurrent | 256.0 | 256.0 | 1297.1 | 2981.7 | 187.0 | 214.0 | 994.9  | 2166.9 | 1222.0 | 1331.0 | 6323.2  | 14011.8 |
|============|=======|=======|========|========|=======|=======|========|========|========|========|=========|=========|


ℹ Request Token Statistics (Completed Requests)
|============|=======|=======|=======|=======|=======|=======|=======|=======|=========|========|
| Benchmark  | Input Tok    || Output Tok   || Total Tok    || Stream Iter  || Output Tok      ||
| Strategy   | Per Req      || Per Req      || Per Req      || Per Req      || Per Stream Iter ||
|            | Mdn   | p95   | Mdn   | p95   | Mdn   | p95   | Mdn   | p95   | Mdn     | p95    |
|------------|-------|-------|-------|-------|-------|-------|-------|-------|---------|--------|
| concurrent | 546.0 | 546.0 | 256.0 | 256.0 | 802.0 | 802.0 | 259.0 | 259.0 | 1.0     | 1.0    |
| concurrent | 546.0 | 546.0 | 256.0 | 256.0 | 802.0 | 802.0 | 259.0 | 259.0 | 1.0     | 1.0    |
|============|=======|=======|=======|=======|=======|=======|=======|=======|=========|========|


ℹ Request Latency Statistics (Completed Requests)
|============|=========|========|========|========|=======|=======|=======|=======|
| Benchmark  | Request Latency || TTFT           || ITL          || TPOT         ||
| Strategy   | Sec             || ms             || ms           || ms           ||
|            | Mdn     | p95    | Mdn    | p95    | Mdn   | p95   | Mdn   | p95   |
|------------|---------|--------|--------|--------|-------|-------|-------|-------|
| concurrent | 20.4    | 21.8   | 1465.9 | 2797.0 | 74.3  | 76.1  | 79.6  | 85.1  |
| concurrent | 28.3    | 30.3   | 2212.2 | 4038.4 | 102.6 | 103.0 | 110.7 | 118.4 |
|============|=========|========|========|========|=======|=======|=======|=======|


ℹ Server Throughput Statistics (All Requests)
|============|=======|======|=========|==============|===============|==============|
| Benchmark  | Requests             ||| Input Tokens | Output Tokens | Total Tokens |
| Strategy   | Concurrency || Per Sec | Per Sec      | Per Sec       | Per Sec      |
|            | Mdn   | Mean | Mean                                               ||||
|------------|-------|------|---------|--------------|---------------|--------------|
| concurrent | 32.0  | 32.0 | 1.1     | 1553.4       | 414.5         | 1306.9       |
| concurrent | 64.0  | 64.0 | 1.3     | 1873.0       | 545.7         | 2418.8       |
|============|=======|======|=========|==============|===============|==============|
```

## Compare GuideLLM benchmark - Multiple Replicas

1. Now, it is time to increase the inference server replicas in order to try to improve our inference service performance. To do so, please execute the following command:

```bash
oc patch inferenceservice llama-vllm-single -n <USER_NAME> --type=merge -p '{
  "spec": {
    "predictor": {
      "minReplicas": 2,
      "maxReplicas": 2
    }
  }
}'
```

2. Wait for the new inference service pod is ready:

```bash
oc get pods -n <USER_NAME> -l serving.kserve.io/inferenceservice=llama-vllm-single
```

3. Now, execute a new benchmark using the same previous parameters:

```bash
cat << 'EOF' | oc apply -f-
apiVersion: batch/v1
kind: Job
metadata:
  name: guidellm-vllm-single-2-replicas
  namespace: <USER_NAME>
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: guidellm
          image: ghcr.io/vllm-project/guidellm:latest
          env:
            - name: GUIDELLM_TARGET
              value: "http://llama-vllm-single-predictor:8000/v1"
            - name: GUIDELLM_PROFILE
              value: concurrent
            - name: GUIDELLM_RATE
              value: "32,64"
            - name: GUIDELLM_OUTPUTS
              value: json,csv
            - name: GUIDELLM_MAX_SECONDS
              value: "30"
            - name: GUIDELLM_DATA
              value: "prompt_tokens=512,output_tokens=256"
            - name: GUIDELLM_PROCESSOR
              value: "RedHatAI/Meta-Llama-3.1-8B-Instruct-FP8-dynamic"
            - name: HF_HOME
              value: /tmp/hf-cache
            - name: TRANSFORMERS_CACHE
              value: /tmp/hf-cache
            - name: XDG_CACHE_HOME
              value: /tmp
          volumeMounts:
            - name: cache
              mountPath: /tmp/hf-cache
            - name: results
              mountPath: /results
      volumes:
        - name: cache
          emptyDir: {}
        - name: results
          emptyDir: {}
EOF
```

4. After running the Quick Start benchmark, review the new performance metrics this time. 

```bash
oc logs -f job/guidellm-vllm-single-2-replicas
```

> SPOILER: You will see similar performance metric, due to using two replicas are not improving the performance metrics

[vLLM manifest delete](_deletevllm.md ':include')

## Compare GuideLLM benchmark - LLM-d

You’ve established that 2 GPUs with naive scaling that doesn’t significantly reduce P95 tail latency. Now, you deploy a llm-d’s prefix-cache-aware routing on the same hardware and run the identical benchmark.

1. Create now the LLM-d distributed inference service to test this solution:

[vLLM manifest LLM-d](_createllmd.md ':include')

The three routing plugins and their weights:

?>`prefix-cache-scorer (weight 3)` -> Routes requests to the backend with the highest KV cache match for this prompt prefix — the primary tail-latency reducer

?>`queue-scorer (weight 2)` -> Balances queue depth across replicas

?>`active-request-scorer (weight 2)` -> Considers in-flight request count

2. Run the exact same GuideLLM configuration:

```bash
LLMD_URL=$(oc get llminferenceservice llama-llm-d -n <USER_NAME> -o jsonpath='{.status.url}')
cat << 'EOF' | oc apply -f-
apiVersion: batch/v1
kind: Job
metadata:
  name: guidellm-llm-d
  namespace: <USER_NAME>
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: guidellm
          image: ghcr.io/vllm-project/guidellm:latest
          env:
            - name: GUIDELLM_TARGET
              value: "$\{LLMD_URL\}"
            - name: GUIDELLM_PROFILE
              value: concurrent
            - name: GUIDELLM_RATE
              value: "32,64"
            - name: GUIDELLM_OUTPUTS
              value: json,csv
            - name: GUIDELLM_MAX_SECONDS
              value: "30"
            - name: GUIDELLM_DATA
              value: "prompt_tokens=512,output_tokens=256"
            - name: GUIDELLM_PROCESSOR
              value: "RedHatAI/Meta-Llama-3.1-8B-Instruct-FP8-dynamic"
            - name: HF_HOME
              value: /tmp/hf-cache
            - name: TRANSFORMERS_CACHE
              value: /tmp/hf-cache
            - name: XDG_CACHE_HOME
              value: /tmp
          volumeMounts:
            - name: cache
              mountPath: /tmp/hf-cache
            - name: results
              mountPath: /results
      volumes:
        - name: cache
          emptyDir: {}
        - name: results
          emptyDir: {}
EOF
```

3. After running the Quick Start benchmark, review the new performance metrics this time. 

```bash
oc logs -f job/guidellm-llm-d
```

> QUESTION: Do you see any performance improvement using LLM-d?

?>Review [guidellm](https://github.com/vllm-project/guidellm/blob/main/docs/guides/datasets.md) documentation to know more about DATASETs and syntatic data generation.

