# 🔑 Authorization and API keys

This lab uses the authorization policy created with the model. That policy allows a group to call the model. An API key is a separate credential, bound to one subscription, that you send on the inference URL.

Complete the configuration, deployment, and subscription labs first. Those labs create:

| Kind | Name | Namespace |
| --- | --- | --- |
| LLMInferenceService | `qwen25-05b-llmd` | `project01-model-deploy` |
| MaaSModelRef | `qwen25-05b-llmd` | `project01-model-deploy` |
| MaaSSubscription | `qwen25-05b-llmd-demo` | `models-as-a-service` |
| MaaSAuthPolicy | `qwen25-05b-llmd-auth` | `models-as-a-service` |

The group on the policy and the subscription is `system:authenticated:oauth`.

| Object | What it controls |
| --- | --- |
| `MaaSAuthPolicy` | Which groups can call which models |
| `MaaSSubscription` | The token limit for those models |
| API key | A bearer token, prefixed `sk-oai-`, bound to one subscription |

A call needs both the policy and the subscription. The gateway treats the two bearer credentials differently:

- A token from `oc whoami -t` is checked with Kubernetes TokenReview. Use it for `${HOST}/maas-api/...`.
- A key that starts with `sk-oai-` is checked by the MaaS API. Use it for the inference URL from the `LLMInferenceService`.

The plaintext key is returned only by the create response. Later list and get calls return the key id, name, subscription, and status.

## Review the authorization policy

1. Confirm `qwen25-05b-llmd-auth` is `Active`:

```bash
oc get maasauthpolicy qwen25-05b-llmd-auth -n models-as-a-service
```

Expected output:

```text
NAME                   PHASE    AGE
qwen25-05b-llmd-auth   Active   1h
```

2. Read the group and the model reference:

```bash
oc get maasauthpolicy qwen25-05b-llmd-auth -n models-as-a-service \
  -o jsonpath='group={.spec.subjects.groups[*].name}{"\n"}model={.spec.modelRefs[*].namespace}/{.spec.modelRefs[*].name}{"\n"}'
```

Expected output:

```text
group=system:authenticated:oauth
model=project01-model-deploy/qwen25-05b-llmd
```

`Failed` means a model reference or namespace in `spec.modelRefs` does not exist. Read the conditions:

```bash
oc get maasauthpolicy qwen25-05b-llmd-auth -n models-as-a-service \
  -o jsonpath='{.status.conditions}{"\n"}' | jq .
```

## Create an API key

Set the gateway host once. The same variables are used by the later steps.

```bash
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
HOST="https://maas.${CLUSTER_DOMAIN}"
```

1. Read the longest expiration the API accepts:

```bash
curl -ksS "${HOST}/maas-api/v1/api-keys/config" \
  -H "Authorization: Bearer $(oc whoami -t)" | jq .
```

?> ✅ The response includes `max_expiration_days`. `expiresIn` accepts values such as `1h`, `24h`, and `30d`. A duration past that maximum is rejected.

2. Create a key bound to `qwen25-05b-llmd-demo`. The create call uses your OpenShift token:

```bash
export API_KEY=$(curl -ksS -X POST "${HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -d '{"name":"lab-key","description":"Lab key for qwen25-05b-llmd","expiresIn":"1h","subscription":"qwen25-05b-llmd-demo"}' \
  | jq -er .key)

echo "${API_KEY}" | grep -q '^sk-oai-' && echo "API key created"
```

?> ✅ The command prints `API key created`. Store `${API_KEY}` before the shell exits. A value of `null` means the create request failed. Run the curl again without `jq -er .key` and correct the error before continuing.

The body field is `subscription`, and the duration field is `expiresIn`. Omitting `subscription` selects the highest-priority subscription for your groups. This lab names `qwen25-05b-llmd-demo`, which has priority `0`.

## List the key

Search with your OpenShift token. The response lists metadata. It does not include the `sk-oai-` secret.

```bash
curl -ksS -X POST "${HOST}/maas-api/v1/api-keys/search" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -d '{"filters":{"status":["active"]},"sort":{"by":"created_at","order":"desc"}}' \
  | jq '.data[] | select(.name=="lab-key") | {id,name,subscription,status,expirationDate}'
```

Expected output:

```json
{
  "id": "<key-id>",
  "name": "lab-key",
  "subscription": "qwen25-05b-llmd-demo",
  "status": "active",
  "expirationDate": "<timestamp>"
}
```

`groups` on the same object is the membership recorded when the key was created.

## Call the model

Use the URL stored on the `LLMInferenceService`. It already contains `/project01-model-deploy/qwen25-05b-llmd`. Appending `/v1/chat/completions` matches the gateway route.

```bash
MODEL_URL=$(oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy -o jsonpath='{.status.url}')

curl -ksS -X POST "${MODEL_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen25-05b-llmd",
    "messages": [
      {"role": "user", "content": "What is OpenShift?"}
    ],
    "max_tokens": 50
  }' | jq .
```

?> ✅ The response is a chat completion. `choices[0].message.role` is `assistant`, and `usage` reports the token counts.

The same call from an OpenAI client uses that inference URL as `base_url`, with `/v1` added. The cluster serving certificate is not in the default trust store, so this client skips verification the same way `curl -k` does. Run it from the same shell, after `API_KEY` and `MODEL_URL` are set:

```bash
export API_KEY2=$(curl -ksS -X POST "${HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -d '{"name":"lab-key-py","description":"Lab key for qwen25-05b-llmd using Python and OpenAI","expiresIn":"1h","subscription":"qwen25-05b-llmd-demo"}' \
  | jq -er .key)

export MODEL_URL=$(oc get llminferenceservice qwen25-05b-llmd -n project01-model-deploy -o jsonpath='{.status.url}')

python3 - << 'PY'
import os
import httpx
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["API_KEY2"],
    base_url=os.environ["MODEL_URL"].rstrip("/") + "/v1",
    http_client=httpx.Client(verify=False),
)
response = client.chat.completions.create(
    model="qwen25-05b-llmd",
    messages=[{"role": "user", "content": "What is OpenShift?"}],
    max_tokens=50,
)
print(response.choices[0].message.content)
PY
```

Install `openai` and `httpx` first if that command reports a missing module. The curl above is the check for this lab.

## Revoke the key

1. Take the id of the active `lab-key`:

```bash
export KEY_ID2=$(curl -ksS -X POST "${HOST}/maas-api/v1/api-keys/search" \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -d '{"filters":{"status":["active"]}}' \
  | jq -er '[.data[] | select(.name=="lab-key-py")][0].id')

echo "${KEY_ID2}"
```

2. Revoke that id with your OpenShift token:

```bash
curl -ksS -X DELETE "${HOST}/maas-api/v1/api-keys/${KEY_ID2}" \
  -H "Authorization: Bearer $(oc whoami -t)" | jq '{id,name,status}'
```

Expected output:

```json
{
  "id": "<key-id>",
  "name": "lab-key-py",
  "status": "revoked"
}
```

3. Repeat the chat request with the saved `${API_KEY}`. The gateway rejects the revoked key.

```bash
python3 - << 'PY'
import os
import httpx
from openai import OpenAI

client = OpenAI(
    api_key=os.environ["API_KEY2"],
    base_url=os.environ["MODEL_URL"].rstrip("/") + "/v1",
    http_client=httpx.Client(verify=False),
)
response = client.chat.completions.create(
    model="qwen25-05b-llmd",
    messages=[{"role": "user", "content": "What is OpenShift?"}],
    max_tokens=50,
)
print(response.choices[0].message.content)
PY
```

Expected output:

```text
Traceback (most recent call last):
  File "<stdin>", line 10, in <module>
  File "/Users/acidonpe/Documents/laptop-configuration/cursor/ai50-labs-instruction/.venv/lib/python3.9/site-packages/openai/_utils/_utils.py", line 298, in wrapper
    return func(*args, **kwargs)
  File "/Users/acidonpe/Documents/laptop-configuration/cursor/ai50-labs-instruction/.venv/lib/python3.9/site-packages/openai/resources/chat/completions/completions.py", line 1284, in create
    return self._post(
  File "/Users/acidonpe/Documents/laptop-configuration/cursor/ai50-labs-instruction/.venv/lib/python3.9/site-packages/openai/_base_client.py", line 1360, in post
    return cast(ResponseT, self.request(cast_to, opts, stream=stream, stream_cls=stream_cls))
  File "/Users/acidonpe/Documents/laptop-configuration/cursor/ai50-labs-instruction/.venv/lib/python3.9/site-packages/openai/_base_client.py", line 1133, in request
    raise self._make_status_error_from_response(err.response) from None
openai.PermissionDeniedError: Error code: 403
```

## When a call fails

| Result | What to check |
| --- | --- |
| `403` on the inference URL | `qwen25-05b-llmd-auth` is `Active`, its group is `system:authenticated:oauth`, and its model reference is `qwen25-05b-llmd` in `project01-model-deploy` |
| `AUTH_FAILURE` from `/maas-api/...` | The `Authorization` header is `Bearer` plus the token from `oc whoami -t` |
| Rejected `sk-oai-` key on the inference URL | Search for the key name and confirm `status` is `active` and `expirationDate` is still in the future |
| Create request returns `invalid_subscription` | `qwen25-05b-llmd-demo` is `Active` in `models-as-a-service` |

A request to `https://maas.<cluster-domain>/v1/chat/completions` has no gateway route. Send chat requests to `${MODEL_URL}/v1/chat/completions`.
