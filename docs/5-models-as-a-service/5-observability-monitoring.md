# 📊 Observability & Monitoring

This lab guides you through enabling and using the Models-as-a-Service (MaaS) observability dashboard to monitor subscription-level usage metrics, token consumption, and rate-limit violations for cost attribution and showback reporting.

!> 🧪 **Technology Preview:** The MaaS observability dashboard is a Technology Preview feature in Red Hat OpenShift AI 3.5. It is not supported with Red Hat production SLAs and might not be functionally complete.

## Understanding MaaS Observability

The MaaS observability dashboard provides insights into how models are being used across your organization, enabling:

- **Cost attribution**: Track token consumption by user, subscription, and model
- **Capacity planning**: Identify usage patterns and resource needs
- **Showback reporting**: Export usage data for finance teams
- **Performance monitoring**: Track request rates, error rates, and latency
- **Rate limit analysis**: Identify users hitting token limits

### Dashboard vs. Billing Systems

> **Note:** The observability dashboard is designed for **showback reporting**, not as a billing-grade metering system. For production chargeback workflows requiring precise billing data, access the Limitador metrics endpoint directly.

### Dashboard Capabilities

| Capability | Description |
| --- | --- |
| **Overview Metrics** | Total tokens, requests, errors, success rate, active users |
| **Filtering** | Filter by user, subscription, and model |
| **Time Range Selection** | From 5 minutes to 14 days, or custom range |
| **Token Consumption Details** | Detailed table by user, subscription, and model |
| **Data Export** | CSV export for cost attribution and reporting |

### Access Control

Access to the observability dashboard is restricted to **cluster administrators** only.

## Enable MaaS Telemetry

Telemetry configures the MaaS gateway to generate Prometheus metrics about model usage.

### Enable via OpenShift Console

1. In the OpenShift console, navigate to **Administration** → **CustomResourceDefinitions**.
2. Search for `Tenant` and click the resource name.
3. Click the **Instances** tab.
4. Click **default-tenant**.
5. Click the **YAML** tab.
6. Locate the `spec` section and add or update the telemetry configuration:

```yaml
spec:
  telemetry:
    enabled: true
    metrics:
      captureOrganization: true
      captureUser: false
      captureGroup: false
      captureModelUsage: true
```

?> 📎 **Field descriptions:**

- `enabled: true` - Activates TelemetryPolicy and Istio Telemetry
- `captureOrganization` - Include organization identifiers (default: true)
- `captureUser` - Include user labels (default: false due to cardinality)
- `captureGroup` - Include group labels (default: false to reduce cardinality)
- `captureModelUsage` - Track model-specific usage (default: true)

!> ⚠️ **Cardinality Warning:** Enabling `captureUser: true` with many users can significantly increase Prometheus database size. Only enable for small deployments or specific monitoring needs.

7. Click **Save**.

### Enable via CLI

1. Patch the Tenant custom resource:

```bash
oc patch tenants.maas.opendatahub.io default-tenant -n models-as-a-service \
  --type merge \
  -p '{
    "spec": {
      "telemetry": {
        "enabled": true,
        "metrics": {
          "captureOrganization": true,
          "captureUser": false,
          "captureGroup": false,
          "captureModelUsage": true
        }
      }
    }
  }'
```

### Verify Telemetry

1. In the OpenShift console, navigate to **Observe** → **Metrics**.
2. Run the query:

```promql
authorized_calls_total
```

3. If telemetry is enabled, the query returns MaaS usage metrics with labels such as `subscription`, `limitador_namespace`, and optionally `model`.

## View the MaaS Observability Dashboard

After enabling all prerequisites, access the MaaS observability dashboard to monitor usage.

### Access the Dashboard

1. Log in to the OpenShift AI dashboard as a **cluster administrator**.
2. In the left navigation menu, click **Models** → **Models-as-a-Service**.
3. Click the **Observability** tab.

?> 📎 The dashboard is embedded using Perses and queries metrics from Prometheus.

### Overview Metrics

The dashboard displays high-level statistics at the top:

| Metric | Description |
| --- | --- |
| **Total Tokens** | Total tokens consumed (input + output) during selected time period |
| **Total Requests** | Total API requests to MaaS models |
| **Total Errors** | Failed requests (model errors, timeouts, server errors) |
| **Success Rate** | Percentage of successful requests (excludes 429 rate-limited) |
| **Active Users** | Unique users who made at least one request |

### Filter and Time Range

1. **Time Range**: Select from dropdown:
   - Last 5 minutes
   - Last 15 minutes
   - Last 30 minutes
   - Last 1 hour
   - Last 3 hours
   - Last 6 hours
   - Last 12 hours
   - Last 24 hours
   - Last 3 days
   - Last 7 days
   - Last 14 days
   - Custom date range
2. **Filters**: Use dropdowns to filter by:
   - **User**: Specific username
   - **Subscription**: Specific subscription name
   - **Model**: Specific model name

?> 💡 **Tip:** Start with a short time range (last 1 hour) to verify metrics are being collected, then expand to longer ranges for analysis.

### Token Consumption Table

The detailed table shows per-user, per-subscription, per-model usage:

| Column           | Description                                       | Sortable |
| ---------------- | ------------------------------------------------- | -------- |
| **User**         | Username                                          | Yes      |
| **Subscription** | Subscription used                                 | Yes      |
| **Model**        | Model accessed (format: `<endpoint>/<model-id>`)  | Yes      |
| **Tokens**       | Total tokens consumed                             | Yes      |
| **Requests**     | Number of API requests                            | Yes      |
| **Rate Limited** | Requests rejected due to rate limiting (HTTP 429) | Yes      |

?> 📎 Click column headers to sort. Multiple rows per user appear if they belong to multiple subscriptions.

## Understand Prometheus Metrics

The observability dashboard queries the following Prometheus metrics:

### MaaS Core Metrics

| Metric | Description | Labels |
| --- | --- | --- |
| `authorized_hits_total` | Total tokens consumed | `subscription`, `model`, `limitador_namespace` |
| `authorized_calls_total` | Total API requests | `subscription`, `limitador_namespace` |
| `limited_calls_total` | Rate-limited requests (HTTP 429) | `limitador_namespace` |
| `istio_request_duration_milliseconds_bucket` | Request latency at gateway | `subscription` |
| `auth_server_authconfig_duration_seconds` | Authorino auth/authz time | `authpolicy` |

### Per-Tenant Metrics

In multitenant deployments, the `maas-api` instance for each tenant exposes metrics with a `tenant_name` label:

| Metric | Description | Labels |
| --- | --- | --- |
| `maas_api_http_requests_total` | HTTP requests to maas-api | `tenant_name`, `method`, `path`, `status` |
| `maas_api_http_request_duration_seconds` | Request duration histogram | `tenant_name`, `method`, `path` |
| `maas_api_key_validation_total` | API key validation attempts | `tenant_name`, `result` |
| `maas_api_token_mint_total` | API key creation events | `tenant_name` |

### Query Metrics Directly

1. In OpenShift console, navigate to **Observe** → **Metrics**.
2. Example queries:

**Total tokens consumed by subscription:**

```promql
sum by (subscription) (authorized_hits_total)
```

**Request rate for a specific tenant:**

```promql
rate(maas_api_http_requests_total{tenant_name="red-team"}[5m])
```

**Error rate by tenant:**

```promql
sum by (tenant_name) (rate(maas_api_http_requests_total{status=~"5.."}[5m]))
```

**Rate limit violations:**

```promql
limited_calls_total
```

**Top 5 token consumers:**

```promql
topk(5, sum by (subscription) (authorized_hits_total))
```

## Monitor Subscription Usage

### Check Token Consumption by Subscription

1. In the observability dashboard, select a time range (e.g., Last 24 hours).
2. Use the **Subscription** filter to select a specific subscription.
3. Review:
   - Total tokens consumed
   - Number of users
   - Models accessed
   - Rate limit violations

### Identify Users Hitting Rate Limits

1. In the Token Consumption table, sort by **Rate Limited** column (descending).
2. Users with high rate-limited request counts are exceeding their token limits.
3. **Actions**:
   - Increase token limits for their subscription
   - Contact users to optimize their usage patterns
   - Review if they need a higher-tier subscription

### Analyze Model Usage Patterns

1. Use the **Model** filter to select a specific model.
2. Review:
   - Which subscriptions are using this model
   - Total tokens consumed
   - Request rates
   - Cost implications
3. **Insights**:
   - Identify most popular models for capacity planning
   - Determine if expensive models need quota restrictions
   - Plan for scaling high-demand models

## Troubleshooting Observability

### Dashboard Shows No Data

**Symptom**: Observability dashboard is blank or shows zero metrics.

**Causes and Solutions**:

1. **No model access yet**
   - Metrics only appear after users make requests
   - Test with an API key to generate initial metrics
2. **User Workload Monitoring not enabled**
   ```bash
    oc get pods -n openshift-user-workload-monitoring
   ```
   - Enable User Workload Monitoring (see above)
3. **Kuadrant observability not enabled**
   ```bash
    oc get kuadrant kuadrant -n kuadrant-system -o jsonpath='{.spec.observability.enable}'
   ```
   - Should return `true`
4. **Telemetry not enabled**
   ```bash
    oc get tenants.maas.opendatahub.io default-tenant -n models-as-a-service -o jsonpath='{.spec.telemetry.enabled}'
   ```
   - Should return `true`
5. **Prometheus not scraping metrics**
   - Check PodMonitor exists:

### Metrics Missing User Labels

**Symptom**: Token Consumption table doesn't show individual users.

**Cause**: `captureUser` is disabled by default due to cardinality concerns.

**Solution** (only for small deployments):

```bash
oc patch tenants.maas.opendatahub.io default-tenant -n models-as-a-service --type=merge --patch '
spec:
  telemetry:
    metrics:
      captureUser: true
'
```

!> ⚠️ **Warning:** Enabling `captureUser` with hundreds or thousands of users can significantly increase Prometheus storage requirements.

### Metrics Missing Model Labels

**Symptom**: Cannot see which specific models are being used.

**Cause**: Model label is only available on `authorized_hits_total` metric due to Kuadrant wasm-shim limitations.

**Workaround**: Use filters in the dashboard or query `authorized_hits_total` directly:

```promql
sum by (model) (authorized_hits_total)
```

### Dashboard Access Denied

**Symptom**: Cannot access the Observability tab in the dashboard.

**Cause**: Only cluster administrators can access the observability dashboard.

**Solution**: Verify your user has cluster-admin role:

```bash
oc get clusterrolebinding | grep cluster-admin
```

## Common Use Cases

### Cost Showback to Finance

1. Select time range: **Last 30 days**
2. Export CSV with all data
3. Aggregate tokens by subscription
4. Apply cost-per-token calculation
5. Generate monthly cost report for each team

### Identify Over-Provisioned Subscriptions

1. Review subscriptions with zero or low usage
2. Check if teams have unused token allocations
3. Reduce token limits or consolidate subscriptions
4. Reallocate capacity to high-usage teams

### Capacity Planning for New Models

1. Monitor usage trends for existing models
2. Identify peak usage times and patterns
3. Estimate resources for new similar models
4. Plan infrastructure scaling proactively

### Troubleshoot User Access Issues

1. Search for user in Token Consumption table
2. If user missing: No successful requests (check auth)
3. If rate limited column high: User hitting quota
4. If requests but no tokens: Requests failing early

**Congratulations!** You have successfully enabled and configured observability for Models-as-a-Service. You can now monitor usage, track costs, and optimize your model serving infrastructure.



You have completed the Models-as-a-Service training module!
