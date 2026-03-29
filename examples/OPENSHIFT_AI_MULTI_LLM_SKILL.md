---
name: openshift-ai-multi-llm
description: Diagnose and resolve issues serving multiple LLMs concurrently in OpenShift AI 2.25+, including intermittent inference endpoint failures, model scheduling conflicts, KServe resource contention, or out-of-memory errors when deploying multiple large language models. Use when users mention "multiple LLMs", "concurrent models", "inference endpoint problems", or model-serving issues on OpenShift AI with KServe. Based on https://access.redhat.com/solutions/7133917
---

## Symptom

Intermittent inference endpoint failures when serving multiple large language models concurrently on OpenShift AI 2.25. Models may fail to deploy, respond with timeouts, or report resource unavailable errors when multiple models are loaded simultaneously.

## Diagnose

List running InferenceServices to check deployment status:

```bash
oc get inferenceservices -A
```

Check KServe model pods for resource issues (OOMKilled, Pending, CrashLoopBackOff):

```bash
oc get pods -n <model-namespace> -l component=predictor
```

View resource allocation across model pods — insufficient CPU/memory is a common cause when running multiple LLMs:

```bash
oc adm top pods -n <model-namespace>
```

Check for node resource pressure on GPU workers:

```bash
oc describe nodes -l nvidia.com/gpu.present=true | grep -A 5 "Allocated resources"
```

Review KServe controller logs for scheduling errors:

```bash
oc logs -n redhat-ods-applications deployment/kserve-controller-manager | grep -i "insufficient\|failed\|timeout"
```

## Root Cause

Running multiple LLMs concurrently requires significant GPU memory and cluster resources. Without proper resource limits and node selectors, pods may contend for GPU memory, trigger OOM kills, or fail to schedule on nodes with available capacity. KServe InferenceServices may also lack the `runtime` configuration needed for efficient multi-model serving.

## Resolve

### 1. Configure dedicated ServingRuntimes for each model

Using separate ServingRuntimes per model type prevents resource contention and allows independent scaling:

```yaml
apiVersion: serving.kserve.io/v1alpha1
kind: ServingRuntime
metadata:
  name: <model-name>-runtime
spec:
  containers:
    - name: kserve-container
      image: quay.io/opendatahub/vllm:latest
      resources:
        limits:
          nvidia.com/gpu: "1"
          memory: "32Gi"
        requests:
          nvidia.com/gpu: "1"
          memory: "32Gi"
```

Apply with `oc apply -f <filename>`.

### 2. Set resource limits per InferenceService

Explicit resource requests prevent pods from being scheduled on overcommitted nodes:

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: <model-name>
spec:
  predictor:
    model:
      modelFormat:
        name: <framework>
      resources:
        limits:
          nvidia.com/gpu: "1"
          memory: "32Gi"
        requests:
          nvidia.com/gpu: "1"
          memory: "32Gi"
    nodeSelector:
      nvidia.com/gpu.present: "true"
```

### 3. Enable proper GPU memory management

For vLLM-based serving (recommended for multi-LLM), configure GPU memory utilization to leave headroom:

```yaml
spec:
  predictor:
    model:
      args:
        - --gpu-memory-utilization=0.85
        - --max-model-len=4096
        - --tensor-parallel-size=1
```

The `gpu-memory-utilization` of 0.85 leaves 15% buffer for KV cache fragmentation.

### 4. Scale with model mesh for concurrent inference

For high-concurrency scenarios, deploy the OpenShift AI ModelMesh component which can serve multiple models from the same runtime pool:

```bash
oc get servingruntimes -n redhat-ods-applications
```

Ensure ModelMesh is enabled in your OpenShift AI cluster configuration.

### 5. Verify model endpoints

After deployment, verify each model responds correctly:

```bash
oc get routes -n <model-namespace>

# Test inference
curl -v -k https://<route>/v1/models
curl -v -k https://<route>/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "<model-name>", "prompt": "test", "max_tokens": 10}'
```

## Workaround

If immediate fixes aren’t possible, stagger model deployments by minute intervals to avoid resource contention during startup:

```bash
for model in model-a model-b model-c; do
  oc apply -f ${model}-isvc.yaml
  sleep 60
done
```

Monitor with:

```bash
watch -n 5 'oc get pods -n <namespace> -o wide'
```

## Related

- OpenShift AI 2.25 documentation: https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.25
- KServe InferenceService API: https://kserve.github.io/website/latest/
- Multi-model serving with ModelMesh: https://access.redhat.com/documentation/en-us/red_hat_openshift_ai_self-managed/2.25/html/serving_models/index
- Subscribe to relevant errata at https://access.redhat.com/errata/ for OpenShift AI updates
- Product: Red Hat OpenShift AI 2.25+
- Tags: rhoai, kserve, llm, multi-model, inference, gpu
