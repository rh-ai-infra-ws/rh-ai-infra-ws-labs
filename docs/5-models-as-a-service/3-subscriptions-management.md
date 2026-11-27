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
