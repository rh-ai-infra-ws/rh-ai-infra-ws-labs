# 🔑 Authorization & API Keys

This lab guides you through managing authorization policies and API keys for Models-as-a-Service (MaaS). You will learn how to control API gateway access and provide users with programmatic access to models.

## Understanding Authorization and Authentication

MaaS uses a two-layer access control model:

### Authorization Policies (Gateway Access)

**Authorization policies** grant groups access to model endpoints through the API gateway. Without an authorization policy, users receive `403 Forbidden` errors even if they have a valid subscription.

### Subscriptions (Quota Management)

**Subscriptions** define quota for models with token rate limits. Both a subscription and an authorization policy are required for users to access models.

| Component | Purpose | Controls |
| --- | --- | --- |
| **Subscription** | Quota management | Token limits, models available, priority |
| **Authorization Policy** | Gateway access | Which groups can access which model endpoints |

!> ⚠️ **Critical:** Users need BOTH a subscription (for quota) AND an authorization policy (for gateway access) to use models.

### Gateway-Scoped Authentication

In OpenShift AI 3.5+, MaaS uses a **single gateway-scoped AuthPolicy** per tenant gateway. This AuthPolicy aggregates authentication and authorization rules for all models served through the tenant's gateway.

The gateway-scoped AuthPolicy enforces:

- API key validation and OIDC token verification
- Subscription checks for model access
- Group membership validation
- Tenant-gateway isolation

## Manage Authorization Policies

### View Authorization Policies via Dashboard

1. Log in to the OpenShift AI dashboard as an administrator.
2. Click **Settings** → **MaaS governance**.
3. Click the **Authorization policies** tab.
4. Review the authorization policies table:

| Column | Description |
| --- | --- |
| **Authorization policy** | Name of the policy |
| **Status** | Current status (Active, Failed, Unknown) |
| **Groups** | Number of authorized user groups (click to expand) |
| **Models** | Number of model endpoints (click to expand) |

5. **Filter and organize**:
   - Use **Keyword** dropdown to select filter criteria
   - Enter text in **Filter by name or description**
   - Click column headers to sort
6. Click an authorization policy name to view details:
   - Full configuration
   - Groups list
   - Models with namespaces
   - Synchronization status

### Create Authorization Policy via Dashboard

1. Navigate to **Settings** → **MaaS governance** → **Authorization policies** tab.
2. Click **Create authorization policy**.
3. Configure the policy:

   **Name**: Enter a unique name (e.g., `ml-team-gateway-access`)

      ?> 📎 Click **Edit resource name** to customize the underlying Kubernetes resource name.

      **Description** (optional): Enter a description (e.g., "Gateway access for ML team to production models")

      **Groups**: Select groups to authorize
   - Click dropdown to select existing OpenShift groups
   - Type to add new group names
   - Multiple groups supported

     ?> 📎 Groups can be OpenShift groups, API key group snapshots, or OIDC token claims.
4. Add models:

   a. Click **Add models** in the Models section.

      b. In the **Add models to authorization policy** dialog:
         - Browse available models
         - Use search field to filter
         - Review **Subscriptions** and **Policies** columns

      c. For each model to add, click **Add model**.

      d. Click **Add models** to confirm.
5. Click **Create authorization policy**.

## Manage API Keys

### Understanding API Keys

API keys enable programmatic access to models through MaaS. Each key:

- Has prefix `sk-oai-` for compatibility with OpenAI clients
- Is bound to a specific subscription (determines accessible models and token limits)
- Snapshots user group membership at creation time
- Can have custom expiration periods
- Is shown only once during creation (cannot be retrieved later)

!> 🔒 **Security:** API keys snapshot group membership at creation. If a user is removed from a group, their existing API keys continue to grant access until revoked.

### View All API Keys (Administrator)

**Via Dashboard:**

1. Navigate to **Gen AI studio** → **API keys**.
2. Review the API keys table:

| Column | Description |
| --- | --- |
| **Name** | Key name assigned during creation |
| **Status** | Active, Expired, or Revoked |
| **Subscription** | Subscription scoping the key's access |
| **Owner** | Username of the key owner |
| **Created** | Creation date |
| **Last used** | Date of last model access |
| **Expires** | Expiration date |

3. **Filter and organize**:
   - Filter by **Status**: Active, Expired, or Revoked
   - Search by username
   - Sort by clicking column headers
   - Use pagination controls for large lists

### Create API Key for a User (Administrator)

**Via Dashboard:**

1. Navigate to **Gen AI studio** → **API keys**.
2. Click **Create API key**.
3. Configure the key:

   **Name**: Enter a descriptive name (e.g., `production-api-client-jan2025`)

      **Description** (optional): Additional details about key purpose

      **Subscription**: Select which subscription determines:
   - Which models the key can access
   - Token limits applied to requests

     ?> 📎 The **Models** section displays models and limits from the selected subscription.

     **Expiration**: Select expiration period:
   - 30 days
   - 60 days
   - 90 days (default)
   - 180 days
   - 1 year
   - Custom (1-365 days)

     ?> 📎 Administrators can set a maximum expiration limit in the Tenant CR. Default maximum is 90 days.
4. Click **Create**.
5. In the **API key created** dialog:

   !> 🔒 **Critical:** The plaintext key is displayed **only once** and cannot be retrieved later. Save it immediately in a secrets manager.

      a. Click the copy icon to copy the key.

      b. Store the key securely (e.g., HashiCorp Vault, AWS Secrets Manager).

      c. Provide the key to the user through a secure channel (not email or chat).
6. Click **Close**.

### Create API Key via MaaS API

1. Get your authentication token:

```bash
# For OpenShift users
TOKEN=$(oc whoami -t)
```

2. Get the MaaS gateway hostname:

```bash
GATEWAY_HOST=$(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.spec.listeners[0].hostname}')
```

3. Create an API key:

```bash
curl -k -X POST "https://${GATEWAY_HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-api-key",
    "description": "Key for production workloads",
    "subscription_name": "data-science-team-premium",
    "expiration_days": 90
  }'
```

?> ✅ The response includes the plaintext API key. Save it immediately.

### Configure API Key Expiration Limit

Administrators can set a maximum expiration period for all API keys.

**Via CLI:**

1. Edit the Tenant custom resource:

```bash
oc edit tenants.maas.opendatahub.io default-tenant -n models-as-a-service
```

2. Add or update the `apiKeyConfig` section:

```yaml
spec:
  apiKeyConfig:
    maxExpirationDays: 180
```

?> 📎 Common values:

- **30 days**: High-security environments
- **90 days**: Standard security (default)
- **180 days**: Moderate security
- **365 days**: Low-security or service accounts

3. Save and exit.

**Via Patch:**

```bash
oc patch tenants.maas.opendatahub.io default-tenant -n models-as-a-service --type=merge --patch '
spec:
  apiKeyConfig:
    maxExpirationDays: 180
'
```

**Verification:**

Test that the limit is enforced by attempting to create a key exceeding the maximum:

```bash
curl -k -X POST "https://${GATEWAY_HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test-long-key",
    "subscription_name": "my-subscription",
    "expiration_days": 400
  }'
```

Expected response: `400 Bad Request` with error message about exceeding maximum.

## Test Model Access with API Key

After creating an API key, test model access through the MaaS gateway.

### Get Gateway Hostname

```bash
GATEWAY_HOST=$(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.spec.listeners[0].hostname}')
```

### Test Chat Completions (Body-Based Routing)

```bash
curl -k -X POST "https://${GATEWAY_HOST}/v1/chat/completions" \
  -H "Authorization: Bearer sk-oai-<your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2-7b-chat",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is Kubernetes?"}
    ],
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

?> ✅ **Recommended:** Body-based routing is compatible with OpenAI SDKs and clients.

### Test with Python OpenAI Client

```python
from openai import OpenAI

# Initialize client
client = OpenAI(
    api_key="sk-oai-<your-api-key>",
    base_url=f"https://{GATEWAY_HOST}/v1"
)

# Make request
response = client.chat.completions.create(
    model="llama-2-7b-chat",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is OpenShift?"}
    ],
    max_tokens=100
)

print(response.choices[0].message.content)
```

## Troubleshooting

### 403 Forbidden Errors

**Symptom**: User has valid API key and subscription but gets `403 Forbidden`.

**Causes**:

1. **No authorization policy** for the user's groups
2. **Model not included** in the authorization policy
3. **User removed from group** but key not revoked (group snapshot issue)

**Solutions**:

1. Verify authorization policy exists:

```bash
oc get maasauthpolicy -n models-as-a-service
```

2. Verify policy includes user's groups:

```bash
oc get maasauthpolicy <policy-name> -n models-as-a-service -o jsonpath='{.spec.groups}' | jq '.'
```

3. Verify policy includes the model:

```bash
oc get maasauthpolicy <policy-name> -n models-as-a-service -o jsonpath='{.spec.models}' | jq '.'
```

4. Check user's current group membership:

```bash
oc get user <username> -o jsonpath='{.groups}' | jq '.'
```

5. If user was removed from group, revoke their API keys.

### 401 Unauthorized Errors

**Symptom**: API requests return `401 Unauthorized`.

**Causes**:

1. **Invalid or expired API key**
2. **Revoked API key**
3. **Missing Authorization header**

**Solutions**:

1. Verify key status:

```bash
# Via dashboard: Gen AI studio → API keys → check Status column
# Via CLI: Query MaaS API
curl -k "https://${GATEWAY_HOST}/maas-api/v1/api-keys" \
  -H "Authorization: Bearer ${TOKEN}" | jq '.[] | select(.name=="<key-name>")'
```

2. Check Authorization header format:

```bash
# Correct format
Authorization: Bearer sk-oai-<your-key>

# Incorrect formats
Authorization: sk-oai-<your-key>  # Missing "Bearer "
Bearer sk-oai-<your-key>           # Missing "Authorization: "
```

3. Generate a new key if current key is expired or lost.

### Authorization Policy Shows Failed Status

**Symptom**: Policy status is `Failed`.

**Solutions**:

1. Check policy conditions:

```bash
oc get maasauthpolicy <policy-name> -n models-as-a-service -o jsonpath='{.status.conditions}' | jq '.'
```

2. Common issues:
   - Model reference not found
   - Invalid group name
   - Model namespace doesn't exist
3. Verify referenced models exist:

```bash
oc get maasmodelref <model-name> -n <model-namespace>
```

**Congratulations!** You have successfully configured authorization policies and API key management for Models-as-a-Service. In the next section, you will enable observability and monitor usage metrics.
