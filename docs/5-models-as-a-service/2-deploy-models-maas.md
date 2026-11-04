# 🚀 Deploy Models with Models-as-a-Service

This lab guides you through deploying generative AI models through the OpenShift AI and publishing them to Models-as-a-Service (MaaS) so that administrators can enforce subscription-based quota and token limits.

## Understanding MaaS Model Publishing

When you deploy a model and publish it to MaaS, the deployment wizard creates:

- **LLMInferenceService**: The actual inference server that handles model serving
- **MaaSModelRef**: A custom resource that registers the model with MaaS for subscription assignment

After publishing, administrators must create a **subscription** and **authorization policy** to make the model accessible to user groups.

> **Note:** Models published to MaaS use the LLMInferenceService architecture, which is designed for large language models and integrates with the MaaS gateway for subscription-based quota enforcement. The legacy deployment method uses traditional KServe InferenceService resources with serving runtimes.

## Deploy a Model via Dashboard

In this section, you will deploy a generative AI model using the OpenShift AI dashboard wizard and publish it to MaaS.

### Using the Dashboard Wizard

1. Log in to the OpenShift AI dashboard.
2. In the left navigation menu, click **Projects**.
3. Click the name of your project where you want to deploy the model (or create a new project if needed).
4. Click the **Deployments** tab.
5. Click **Deploy model** to open the deployment wizard.

### Configure Model Details

1. In the **Model details** section, configure your model storage:
   - **Storage connection**: Select an existing data connection or create a new one pointing to your model storage (S3, OCI, or other supported storage)
   - **Model path**: Enter the path to your model within the storage (e.g., `models/llama-2-7b/`)
   - **Model type**: Select **Generative AI model (Example, LLM)**
2. Verify that the **Use legacy deployment method** checkbox is **unchecked**.

?> 📎 The legacy deployment method uses KServe InferenceService. For MaaS integration, you must use the new LLMInferenceService architecture.

3. Click **Next**.

### Configure Model Deployment

1. In the **Model deployment** section, configure the deployment settings:
   - **Model deployment name**: Enter a unique name using lowercase letters, numbers, and hyphens (e.g., `llama-2-7b-chat`)
2. Select the **deployment resource**:
   - **Distributed inference with llm-d**: Recommended for production deployments and multi-GPU models
3. Configure **Number of replicas**:
   - For development: 1 replica
4. Click **Next**.

### Configure Advanced Settings and MaaS Publishing

1. In the **Advanced settings** section, configure MaaS publishing:

   a. Under **Model availability**, select **Publish as MaaS**.
   ```
   !> ⚠️ **Important:** Publishing as MaaS creates a MaaSModelRef object that registers the model with MaaS. After publishing, an administrator must create a subscription and authorization policy to make the model accessible to users.

   b. **Optional**: Configure custom runtime settings if needed:
      - Click **Add custom runtime arguments** to add model-specific arguments (e.g., `--max-model-len 4096`)
      - Click **Add custom runtime environment variables** to add environment variables
   ```
2. Click **Next**.

### Review and Deploy

1. In the **Review** section, verify your configuration:
   - Model storage and path
   - Deployment name and hardware profile
   - MaaS publishing enabled
   - Number of replicas
   - Custom runtime settings (if any)
2. Click **Deploy model**.
3. Wait for the deployment to complete. The Status column will show:
   - **Creating**: Deployment is in progress
   - **✓ Deployed**: Model is ready to use

?> ⏳ Initial deployment can take 5-10 minutes depending on model size and cluster resources.

## Verify Model Deployment

After deploying your model, verify that all components are properly configured.

### Verify LLMInferenceService

1. Check the status of your LLMInferenceService:

```bash
oc get llminferenceservice llama-2-7b-chat -n your-project
```

Expected output:

```text
NAME               READY   URL                                            AGE
llama-2-7b-chat    True    https://llama-2-7b-chat-your-project.apps...   5m
```

?> ✅ The **READY** column should show **True**.

2. Verify that the inference pods are running:

```bash
oc get pods -n your-project -l inference.opendatahub.io/llm-inference-service=llama-2-7b-chat
```

Expected output:

```text
NAME                                READY   STATUS    RESTARTS   AGE
llama-2-7b-chat-predictor-0         2/2     Running   0          5m
```

### Verify MaaSModelRef

1. Verify that the MaaSModelRef was created:

```bash
oc get maasmodelref llama-2-7b-chat -n your-project
```

Expected output:

```text
NAME               AGE
llama-2-7b-chat    5m
```

2. View the MaaSModelRef details:

```bash
oc get maasmodelref llama-2-7b-chat -n your-project -o yaml
```

?> 📎 Look for the `status.conditions` section to verify the model is properly registered with MaaS.

### Test Model Endpoint (Without MaaS Governance)

1. Before configuring subscriptions, you can test the model endpoint directly:

```bash
# Get the model URL
MODEL_URL=$(oc get llminferenceservice llama-2-7b-chat -n your-project -o jsonpath='{.status.url}')

# Get your token
TOKEN=$(oc whoami -t)

# Test the model endpoint
curl -k -X POST "${MODEL_URL}/v1/chat/completions" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-2-7b-chat",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "What is OpenShift?"}
    ],
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

?> ✅ A successful response returns JSON with the model's completion.

!> ⚠️ **Note:** This direct endpoint test bypasses MaaS governance. Once subscriptions and authorization policies are configured, users will access models through the MaaS gateway with enforced token limits.

## List Published Models

1. List all MaaSModelRef resources across namespaces:

```bash
oc get maasmodelref --all-namespaces
```

Expected output:

```text
NAMESPACE       NAME               AGE
your-project    llama-2-7b-chat    10m
team-b          mistral-7b         5m
```

2. View detailed information about a specific model:

```bash
oc describe maasmodelref llama-2-7b-chat -n your-project
```

## Troubleshooting Model Deployment

### Model Deployment Stuck in Creating Status

**Issue**: Model deployment shows "Creating" status for extended time.

**Solution**:

1. Check pod status:

```bash
oc get pods -n your-project -l inference.opendatahub.io/llm-inference-service=llama-2-7b-chat
```

2. Check pod events:

```bash
oc describe pod <pod-name> -n your-project
```

Common issues:

- Insufficient GPU resources: Check node capacity
- Image pull errors: Verify network connectivity and image registry access
- Storage connection issues: Verify data connection secret is correct

### MaaSModelRef Not Created

**Issue**: Model deployed but MaaSModelRef is missing.

**Solution**:

1. Verify that you selected "Publish as MaaS" in the deployment wizard.
2. Check MaaS controller logs:

```bash
oc logs -n redhat-ods-applications -l control-plane=controller-manager,app.kubernetes.io/name=maas-controller
```

### Model Not Visible in Dashboard

**Issue**: Model deployed but not visible in MaaS models list.

**Solution**:

1. Verify namespace has the correct labels:

```bash
oc get namespace your-project --show-labels
```

2. Check dashboard configuration:

```bash
oc get odhdashboardconfig default-odh-dashboard-config -n redhat-ods-applications -o jsonpath='{.spec.dashboardConfig.modelsAsAService}'
```

## Next Steps

Now that your model is deployed and published to MaaS:

1. **Create subscriptions** to grant groups quota for this model with token limits
2. **Create authorization policies** to grant API gateway access
3. **Generate API keys** for users to access the model

**Congratulations!** You have successfully deployed a model with Models-as-a-Service. In the next section, you will configure subscriptions to control model access and token limits.
