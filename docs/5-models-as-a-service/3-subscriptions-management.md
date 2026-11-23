# 🎫 Managing Subscriptions

This lab guides you through creating and managing subscriptions for Models-as-a-Service (MaaS). Subscriptions control group access to models and enforce token limits based on user group membership.

## Understanding MaaS Subscriptions

Models-as-a-Service uses **subscription-based access control** to manage quotas and token limits for AI model serving. Subscriptions enable you to:

- **Prevent resource exhaustion** by enforcing token limits per model
- **Provide different access levels** for different user groups
- **Track and allocate costs** based on team consumption
- **Control access** to high-cost or sensitive models
- **Support multi-subscription membership** where users can belong to multiple subscriptions

> **Note:** In OpenShift AI 3.3, MaaS used a tier-based model. Starting with OpenShift AI 3.4, tiers have been replaced with subscriptions, providing more flexibility and GitOps compatibility through custom resources.

### Subscription Properties

Each subscription is defined as a `MaaSSubscription` custom resource with the following properties:

| Property | Description |
| --- | --- |
| **Name** | Unique identifier that becomes the Kubernetes resource name |
| **Description** | Optional human-readable description shown in the dashboard |
| **Groups** | One or more groups whose members can access this subscription |
| **Priority Level** | Numeric value determining subscription precedence (0 = lowest, higher = more priority) |
| **Models** | List of models with configurable token limits for each |

### Subscriptions vs Authorization Policies

Both subscriptions and authorization policies are required for users to access models:

- **Subscription**: Gives groups quota for specific models with token rate limits
- **Authorization Policy**: Grants groups access to model endpoints through the API gateway

> **Important:** Without an authorization policy, users receive `403 Forbidden` errors even if they have a valid subscription.

## Priority Levels

Priority levels determine which subscription is selected when creating an API key **without explicitly specifying a subscription**. When a user belongs to multiple groups with different subscriptions, the subscription with the **highest priority** is selected as the default.

### Recommended Priority Scheme

Use a consistent priority numbering scheme:

| Use Case | Priority | Description |
| --- | --- | --- |
| **Production workloads** | 100 | Customer-facing applications, production APIs, critical business processes |
| **Staging/Pre-production** | 50 | QA testing, UAT, performance testing |
| **Development** | 0 (default) | Exploratory work, prototypes, learning |
| **Personal/Sandbox** | -10 | Individual experimentation, tutorials, non-business use |

?> 💡 **Best Practice:** Use incremental values with gaps (10, 20, 30) rather than consecutive numbers (0, 1, 2) to allow for inserting intermediate priorities later.

### Priority Use Cases

**Separate production and development resources**

- Production subscription (priority 100): Stricter quotas, billed to production cost centers
- Development subscription (priority 0): Generous quotas, billed to R&D
- Production applications automatically use the production subscription
- Developers must explicitly select the development subscription for testing

**Team-based access with overlapping membership**

- "ML Platform Team" (priority 10): Access to all models
- "Analytics Team" (priority 5): Analytics-focused models only
- Users in both teams default to ML Platform Team subscription

**Cost-tiered model access**

- "Standard Models" (priority 10): Cheaper models with higher quotas
- "Premium Models" (priority 0): Expensive models with limited quotas
- Users consume cheaper resources by default
- Explicit selection required for premium models

## Create a Subscription via Dashboard

### Using the Dashboard Wizard

1. Log in to the OpenShift AI dashboard as an administrator.
2. Click **Settings** → **MaaS governance**.
3. The **Subscriptions** tab is displayed by default. Click **Create subscription**.
4. In the **Create subscription** dialog, configure the following:

### Configure Basic Settings

1. **Name**: Enter a unique name (e.g., `data-science-team-premium`)

?> 📎 By default, the resource name matches the subscription name. To customize, click **Edit resource name**.

2. **Description** (optional): Enter a description (e.g., "Production access for data science team with premium models")
3. **Groups**: Select groups to grant access to this subscription
   - Click the dropdown to select existing OpenShift groups
   - Type to add new group names
   - You can select multiple groups

?> 📎 Groups can come from OpenShift Group objects, API key group snapshots, or OIDC token claims depending on your authentication method.

### Configure Priority

1. **Priority**: Enter a numeric priority level
   - Higher numbers = higher priority
   - Default: 0
   - Recommended: Use the priority scheme described above

Example values:

- Production: `100`
- Staging: `50`
- Development: `0`
- Sandbox: `-10`

### Add Models and Token Limits

1. In the **Models** section, click **Add models**.
2. In the **Add models to subscription** dialog:
   - Browse the list of available models
   - Use the search field to filter by name or description
   - Review the **Policies** column to see which authorization policies already include each model
3. For each model to add, click **Add model**.
4. Click **Add models** to add selected models to the subscription.
5. Configure token limits for each model:

   a. In the **Token limit** column for each model, click **Edit limit**.

    b. In the token limit configuration dialog, set:
       - **Tokens per request**: Maximum tokens per single inference request
       - **Tokens per minute**: Token rate limit over time
       - **Tokens per day**: Daily token quota

    ?> 💡 **Example limits:**
   - **Development**: 2048 per request, 10,000 per minute, 100,000 per day
   - **Production**: 8192 per request, 100,000 per minute, 1,000,000 per day

    c. Click **Save**.

### Create Authorization Policy (Optional)

1. **Create matching authorization policy**: Check this box to automatically create an authorization policy with the same groups and models

?> ✅ **Recommended:** Enable this option so users can immediately access models. Without an authorization policy, users will get `403 Forbidden` errors.

### Review and Create

1. Review all settings in the dialog.
2. Click **Create subscription**.
3. The subscription appears in the **Subscriptions** table with **Active** status.

## Monitor Subscriptions

### View All Subscriptions via Dashboard

1. Navigate to **Settings** → **MaaS governance** → **Subscriptions** tab.
2. Review the subscriptions table:

| Column | Description |
| --- | --- |
| **Subscription** | Unique identifier |
| **Status** | Active, Failed, or other status |
| **Groups** | Number of assigned user groups (click to expand) |
| **Models** | Count of models included (click to expand) |
| **Priority** | Priority level for this subscription |

3. **Filter and sort**:
   - Use the **Keyword** dropdown to select filter criteria
   - Enter text in **Filter by name or description**
   - Click column headers to sort
4. Click a subscription name to view details:
   - Full configuration
   - Groups list
   - Models with token limits
   - Synchronization status

### Test Model Access

1. As a user in the subscription group, generate an API key (covered in the next section).
2. Test model access through the MaaS gateway:

```bash
# Get gateway hostname
GATEWAY_HOST=$(oc get gateway maas-default-gateway -n openshift-ingress -o jsonpath='{.spec.listeners[0].hostname}')

# Test with API key
curl -k -X POST "https://${GATEWAY_HOST}/v1/chat/completions" \
  -H "Authorization: Bearer sk-oai-<your-api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2-7b-chat",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_tokens": 50
  }'
```

?> ✅ A successful response returns HTTP 200 with the model completion.

## Troubleshooting Subscriptions

### Subscription Shows Failed Status

**Issue**: Subscription in Failed phase.

**Solution**:

1. Check subscription conditions:

```bash
oc get maassubscription <name> -n models-as-a-service -o jsonpath='{.status.conditions}' | jq '.'
```

2. Common causes:
   - Model reference not found: Verify MaaSModelRef exists
   - Invalid group name: Check OpenShift groups
   - Namespace not found: Verify model namespace exists

### User Cannot Access Model Despite Subscription

**Issue**: User has valid subscription but gets `403 Forbidden`.

**Solution**:

1. Verify authorization policy exists:

```bash
oc get maasauthpolicy -n models-as-a-service
```

2. Check that the policy includes the same groups and models as the subscription.
3. If missing, create a matching authorization policy (see examples above).

### Token Limits Not Enforced

**Issue**: Users can exceed configured token limits.

**Solution**:

1. Verify Kuadrant observability is enabled:

```bash
oc get kuadrant kuadrant -n kuadrant-system -o jsonpath='{.spec.observability.enable}'
```

Expected output: `true`

2. Check Limitador is running:

```bash
oc get pods -n kuadrant-system -l app=limitador
```

3. Verify TokenRateLimitPolicy resources exist:

```bash
oc get tokenratelimitpolicy -n <model-namespace>
```

## Best Practices for Subscriptions

### Naming Conventions

- Use descriptive names: `data-science-production`, `analytics-staging`
- Include team/purpose: `ml-team-premium`, `research-development`
- Avoid generic names: `subscription1`, `test`

### Token Limit Recommendations

| Environment | Tokens/Request | Tokens/Minute | Tokens/Day |
| --- | --- | --- | --- |
| **Development** | 2048 | 10,000 | 100,000 |
| **Staging** | 4096 | 50,000 | 500,000 |
| **Production** | 8192 | 100,000 | 1,000,000 |
| **Enterprise** | 16384 | 500,000 | 10,000,000 |

### Group Management

- Use OpenShift groups for internal users
- Use OIDC groups for external users
- Align groups with organizational structure
- Document group purposes and membership criteria

### Priority Assignment

- Leave gaps between priority values (10, 20, 30 vs 1, 2, 3)
- Document priority scheme in team documentation
- Review and adjust priorities periodically
- Avoid setting all subscriptions to the same priority

**Congratulations!** You have successfully created and managed subscriptions for Models-as-a-Service. In the next section, you will configure authorization policies and manage API keys.
