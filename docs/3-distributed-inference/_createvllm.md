```bash
cat << 'EOF' | oc apply -n <USER_NAME> -f -
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama-vllm-single
  annotations:
    openshift.io/display-name: Llama 3.1 8B FP8 Single
    security.opendatahub.io/enable-auth: "false"
    serving.kserve.io/deploymentMode: RawDeployment
    opendatahub.io/hardware-profile-name: gpu-profile
    opendatahub.io/hardware-profile-namespace: redhat-ods-applications
spec:
  predictor:
    automountServiceAccountToken: false
    maxReplicas: 1
    minReplicas: 1
    deploymentStrategy:
      type: Recreate
    model:
      modelFormat:
        name: vLLM
      name: ''
      resources:
        limits:
          cpu: '4'
          memory: 8Gi
          nvidia.com/gpu: '1'
        requests:
          cpu: '4'
          memory: 8Gi
          nvidia.com/gpu: '1'
      runtime: rhaiis-cuda
      storageUri: 'oci://registry.redhat.io/rhelai1/modelcar-llama-3-1-8b-instruct-fp8-dynamic:1.5'
EOF
```

Verify the inference service was properly created:

```bash
oc get InferenceService -n <USER_NAME>
```

* **Sample `oc get InferenceService` output** (add `-o wide` for the prediction URL; column names can vary by CLI/operator version):

```text
NAME                URL                                                             READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION   AGE
llama-vllm-single   http://llama-vllm-single-predictor.acidonpe.svc.cluster.local   True                                                                  18m
```

?> It takes a few minutes for the model to load and the state to change to "Ready". Leave this running in your terminal and continue reading the concepts below. We’ll come back to test the deployment at the end of this module.