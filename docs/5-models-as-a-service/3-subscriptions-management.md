# 🎫 Managing Subscriptions

This lab uses the subscription created in the previous section. A subscription gives a group a token limit for a model. It does not, by itself, allow that group to call the model. The `MaaSAuthPolicy` from the previous lab does that.

Complete the configuration and deployment labs first. Those labs create:

| Kind | Name | Namespace |
| --- | --- | --- |
| LLMInferenceService | `qwen25-05b-llmd` | `project01-model-deploy` |
| MaaSModelRef | `qwen25-05b-llmd` | `project01-model-deploy` |
| MaaSSubscription | `qwen25-05b-llmd-demo` | `models-as-a-service` |
| MaaSAuthPolicy | `qwen25-05b-llmd-auth` | `models-as-a-service` |

The owner group is `system:authenticated:oauth`. That is the group on an `oc login` session, including the account behind `oc whoami -t`.

## Subscription fields

`qwen25-05b-llmd-demo` sets these fields:

| Field | Value in this lab | Purpose |
| --- | --- | --- |
| `spec.owner.groups` | `system:authenticated:oauth` | Group that receives the quota |
| `spec.modelRefs` | `qwen25-05b-llmd` in `project01-model-deploy` | Model the quota applies to |
| `spec.modelRefs[].tokenRateLimits` | `5000000` tokens per `1m` | Rate limit enforced for that model |
| `spec.tokenMetadata` | `costCenter` and `organizationId` | Labels stored with usage |

`oc get maassubscription` prints a `PRIORITY` column. This subscription does not set a priority, so the column is `0`. When a user belongs to more than one subscription and an API key is created without `"subscription"`, MaaS selects the subscription with the highest priority. The key created in the previous lab names `qwen25-05b-llmd-demo`, so that selection is not used.

Leave gaps between priority values (`0`, `10`, `20`) when you add more subscriptions later. A failed subscription still occupies its name. Fix the object instead of creating a second one with the next integer.

## Review the subscription

1. Confirm the subscription is `Active` and the priority is `0`:

```bash
oc get maassubscription qwen25-05b-llmd-demo -n models-as-a-service
```

Expected output:

```text
NAME                   PHASE    PRIORITY   AGE
qwen25-05b-llmd-demo   Active   0          1h
```

2. Read the token limit:

```bash
oc get maassubscription qwen25-05b-llmd-demo -n models-as-a-service \
  -o jsonpath='{.spec.modelRefs[0].tokenRateLimits[0]}{"\n"}'
```

Expected output:

```text
{"limit":5000000,"window":"1m"}
```

`PHASE` must be `Active` before you change the limit or call the model. `Failed` means a model reference or namespace in `spec.modelRefs` does not exist. Check the conditions:

```bash
oc get maassubscription qwen25-05b-llmd-demo -n models-as-a-service \
  -o jsonpath='{.status.conditions}{"\n"}' | jq .
```

## Change the token limit

1. Lower the limit to 100,000 tokens per minute:

```bash
oc patch maassubscription qwen25-05b-llmd-demo -n models-as-a-service --type=json -p '[
  {"op":"replace","path":"/spec/modelRefs/0/tokenRateLimits/0/limit","value":100000}
]'
```

2. Read the limit again and confirm the phase is still `Active`:

```bash
oc get maassubscription qwen25-05b-llmd-demo -n models-as-a-service \
  -o jsonpath='{.status.phase}{" "}{.spec.modelRefs[0].tokenRateLimits[0]}{"\n"}'
```

Expected output:

```text
Active {"limit":100000,"window":"1m"}
```

3. Restore the original limit of 5,000,000 tokens per minute:

```bash
oc patch maassubscription qwen25-05b-llmd-demo -n models-as-a-service --type=json -p '[
  {"op":"replace","path":"/spec/modelRefs/0/tokenRateLimits/0/limit","value":5000000}
]'
```

The gateway copies this limit into a `TokenRateLimitPolicy` named `maas-trlp-qwen25-05b-llmd` in `project01-model-deploy`. After the patch, that policy should exist:

```bash
oc get tokenratelimitpolicy maas-trlp-qwen25-05b-llmd -n project01-model-deploy
```

## Call the model

Use the URL stored on the `LLMInferenceService`. It already contains `/project01-model-deploy/qwen25-05b-llmd`. Appending `/v1/chat/completions` matches the gateway route. A request to `https://maas.<cluster-domain>/v1/chat/completions` does not.

1. Call the model with your OpenShift user token:

```bash
MODEL_URL=$(oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy -o jsonpath='{.status.url}')
TOKEN=$(oc whoami -t)

curl -ksS -X POST "${MODEL_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen25-05b-llmd",
    "messages": [
      {"role": "user", "content": "What is OpenShift?"}
    ],
    "max_tokens": 50
  }' | jq .
```

?> ✅ The response is a chat completion. `choices[0].message.role` is `assistant`, and `usage` reports the token counts. The same URL accepts an `sk-oai-` key created for `qwen25-05b-llmd-demo`.

A `403` means the authorization policy does not include the caller's group. Confirm `qwen25-05b-llmd-auth` is `Active` and its group is `system:authenticated:oauth`:

```bash
oc get maasauthpolicy qwen25-05b-llmd-auth -n models-as-a-service
```

## Reach the token limit

The limit of 5,000,000 tokens per minute accepts the chat call above. Lower it to 10 tokens per minute so the next call in the same minute is rejected. The gateway reads `usage.total_tokens` from a successful response and adds that number to the counter for `qwen25-05b-llmd-demo`.

1. Set the limit to 10 tokens per minute:

```bash
oc patch maassubscription qwen25-05b-llmd-demo -n models-as-a-service --type=json -p '[
  {"op":"replace","path":"/spec/modelRefs/0/tokenRateLimits/0/limit","value":10}
]'
```

2. Wait until the gateway policy uses that limit and is `Enforced`. The limit name includes the value, so it changes from `tokens-5000000-per-1m` to `tokens-10-per-1m`:

```bash
oc get tokenratelimitpolicy maas-trlp-qwen25-05b-llmd -n project01-model-deploy -o json | jq '{
  enforced: ([.status.conditions[] | select(.type=="Enforced") | .status][0]),
  limit: (.spec.limits | to_entries[0] | {name: .key, value: .value.rates[0]})
}'
```

Expected output:

```json
{
  "enforced": "True",
  "limit": {
    "name": "tokens-10-per-1m",
    "value": {
      "limit": 10,
      "window": "1m"
    }
  }
}
```

Send traffic only after `enforced` is `True` and the limit name is `tokens-10-per-1m`. The counter is per subscription and per user, and the window is one minute.

3. Send chat requests until the gateway rejects one. The loop prints the HTTP status and, for `429`, the response body:

```bash
MODEL_URL=$(oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy -o jsonpath='{.status.url}')
TOKEN=$(oc whoami -t)

for i in 1 2 3 4 5 6; do
  CODE=$(curl -ksS -o /tmp/maas-limit-body -w '%{http_code}' \
    -X POST "${MODEL_URL}/v1/chat/completions" \
    -H "Authorization: Bearer ${TOKEN}" \
    -H "Content-Type: application/json" \
    -d '{"model":"qwen25-05b-llmd","messages":[{"role":"user","content":"What is OpenShift?"}],"max_tokens":50}')
  echo "request ${i}: HTTP ${CODE}"
  if [ "${CODE}" = "429" ]; then
    echo "response:"
    cat /tmp/maas-limit-body
    echo
    break
  fi
  jq '{total_tokens: .usage.total_tokens}' /tmp/maas-limit-body
done
```

Expected output:

```text
request 1: HTTP 200
{
  "total_tokens": 74
}
request 2: HTTP 429
response:
Too Many Requests
```

?> ✅ `total_tokens` on the first response is greater than 10, so that call fills the window. The next call in the same minute is HTTP `429` and the body is `Too Many Requests`. The exact token count changes between calls. If request 1 is already `429`, this minute already contains a successful call. Wait one minute and run the loop again.

4. Read the same rejection in the gateway access log. The status on the inference path is `429`:

```bash
oc logs -n openshift-ingress -l gateway.networking.k8s.io/gateway-name=maas-default-gateway -c istio-proxy --tail=200 \
  | grep 'qwen25-05b-llmd/v1/chat/completions'
```

A rejected line contains:

```text
"POST /project01-model-deploy/qwen25-05b-llmd/v1/chat/completions HTTP/2" 429
```

5. Read the Limitador counter. User workload Prometheus scrapes it. The series appears after the next scrape, which can take about a minute:

```bash
THANOS=$(oc get route thanos-querier -n openshift-monitoring -o jsonpath='{.spec.host}')

curl -ksS -H "Authorization: Bearer $(oc whoami -t)" \
  --data-urlencode 'query=limited_calls{limitador_namespace="project01-model-deploy/qwen25-05b-llmd-kserve-route"}' \
  "https://${THANOS}/api/v1/query" | jq '.data.result[].value'
```

?> ✅ The value is a counter. After the `429`, it is `1` or higher. `authorized_hits` for the same route still holds the tokens from the successful call.

6. Open the same rejection in the OpenShift AI dashboard:

```bash
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
echo "https://data-science-gateway.${CLUSTER_DOMAIN}"
```

   a. Click **Observe & monitor** → **Dashboard**.

   b. Open the **Usage** tab.

   c. Set **Time period** to **Last 5 minutes**.

   d. Set **Subscription** to `qwen25-05b-llmd-demo`.

   e. In **Token Consumption by User**, the row for this subscription shows **Tokens** from the successful call and **Rate Limited** greater than zero.

The **Rate Limited** column is the count of HTTP `429` responses. **Success Rate** in the overview leaves those rejections out of the successful share. The same series is on the OpenShift console under **Observe** → **Metrics**, with the `limited_calls` query from the previous step.

7. Restore the original limit of 5,000,000 tokens per minute:

```bash
oc patch maassubscription qwen25-05b-llmd-demo -n models-as-a-service --type=json -p '[
  {"op":"replace","path":"/spec/modelRefs/0/tokenRateLimits/0/limit","value":5000000}
]'

oc get tokenratelimitpolicy maas-trlp-qwen25-05b-llmd -n project01-model-deploy \
  -o jsonpath='{.spec.limits}' | jq 'keys'
```

Expected output:

```json
[
  "tokens-5000000-per-1m"
]
```

## When the limit is not enforced

1. Confirm Kuadrant observability is enabled:

```bash
oc get kuadrant kuadrant -n kuadrant-system -o jsonpath='{.spec.observability.enable}{"\n"}'
```

Expected output:

```text
true
```

2. Confirm Limitador is running:

```bash
oc get pods -n kuadrant-system -l app=limitador
```

3. Confirm the token-limit policy for this model exists:

```bash
oc get tokenratelimitpolicy -n project01-model-deploy
```

The policy `maas-trlp-qwen25-05b-llmd` must be listed. User workload monitoring, enabled in the configuration lab, collects the metrics this policy uses.
