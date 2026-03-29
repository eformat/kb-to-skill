---
name: openshift-ai-gpu-select
description: Configure model deployment on a specific GPU in Red Hat OpenShift AI multi-GPU environments. Use this skill when a user needs to pin an AI/ML model to a particular GPU device, select a specific GPU by ID or UUID in OpenShift AI, or troubleshoot GPU assignment for model serving on multi-GPU nodes. Also use when users mention nvidia-smi, CUDA_VISIBLE_DEVICES, or GPU scheduling in the context of OpenShift AI or RHOAI. Based on https://access.redhat.com/solutions/7126515
---

## Symptoms

In a multi-GPU node, models deploy to the default GPU rather than a specific one. The user needs to target a particular GPU — for example, selecting GPU 1 (`NVIDIA GeForce GTX 1050 Ti`) instead of GPU 0 (`NVIDIA L4`):

```
sh-4.4# nvidia-smi -L
GPU 0: NVIDIA L4 (UUID: GPU-e0729c5e-88c4-e122-d21a-5128f785b863)
GPU 1: NVIDIA GeForce GTX 1050 Ti (UUID: GPU-ba6ad321-6d04-704b-88b1-8f542689f05c)
```

## Diagnose

List available GPUs on the node to identify device IDs and UUIDs.

```bash
oc debug node/<node-name> -- chroot /host nvidia-smi -L
```

Confirm which GPU resources the node advertises to Kubernetes.

```bash
oc describe node <node-name> | grep -A5 "nvidia.com/gpu"
```

Check current GPU allocation across pods on the node.

```bash
oc get pods -A -o json | jq -r '.items[] | select(.spec.nodeName=="<node-name>") | select(.spec.containers[].resources.limits["nvidia.com/gpu"] != null) | .metadata.namespace + "/" + .metadata.name'
```

## Resolve

### Option 1: CUDA_VISIBLE_DEVICES in the InferenceService

Set the `CUDA_VISIBLE_DEVICES` environment variable in the serving runtime or InferenceService to restrict which GPU the model sees. The model will only use the specified device(s).

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: <model-name>
  namespace: <project>
spec:
  predictor:
    model:
      modelFormat:
        name: <format>
      runtime: <runtime-name>
      resources:
        limits:
          nvidia.com/gpu: "1"
    containers:
      - name: kserve-container
        env:
          - name: CUDA_VISIBLE_DEVICES
            value: "1"
```

Replace `"1"` with the target GPU device ID from `nvidia-smi -L`.

### Option 2: GPU UUID targeting via NVIDIA_VISIBLE_DEVICES

For more precise control, use the GPU UUID instead of the device index — device indices can shift across reboots.

```yaml
env:
  - name: NVIDIA_VISIBLE_DEVICES
    value: "GPU-ba6ad321-6d04-704b-88b1-8f542689f05c"
```

### Option 3: Node-level GPU partitioning with MIG or Time-Slicing

If you need to share a single GPU across multiple models, configure NVIDIA MIG (Multi-Instance GPU) or time-slicing through the NVIDIA GPU Operator. This is configured at the cluster level, not per-model.

```bash
oc get clusterpolicy gpu-cluster-policy -o yaml | grep -A10 "mig\|timeSlicing"
```

## Verify

Confirm the model pod landed on the correct GPU.

```bash
oc exec <model-pod> -- nvidia-smi
```

The output should show only the targeted GPU, not all GPUs on the node.

## Related

- KB article: https://access.redhat.com/solutions/7126515
- Product: Red Hat OpenShift AI
- Components: openshift, GPU scheduling, model serving
