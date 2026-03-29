---
name: openshift-ai-hardware-profile
description: Enable and configure Hardware Profiles in OpenShift AI 2.25+ to optimize GPU/CPU resource allocation for model serving. Use this skill when configuring GPU node selectors, tolerations, accelerator profiles, or hardware-specific resource pools for KServe inference. Based on https://access.redhat.com/solutions/7133582
---

## Symptoms

You need to configure hardware-specific resource allocation for model serving in OpenShift AI 2.25+. Hardware Profiles allow mapping inference services to specific GPU types, CPU architectures, or node pools with fine-grained resource controls.

## Check Current Configuration

Verify the OpenShift AI version and existing accelerator profiles.

```bash
oc get csv -n redhat-ods-operator | grep opendatahub
```

List existing accelerator profiles in the cluster.

```bash
oc get acceleratorprofiles -n redhat-ods-applications
```

Check the DSCInitialization status for hardware profile support.

```bash
oc get dscinitialization default-dsci -o yaml | grep -A5 "hardwareProfiles"
```

## Enable Hardware Profiles

Hardware Profiles are managed through the DataScienceCluster (DSC) custom resource. Edit the DSC to enable the feature.

```bash
oc edit datasciencecluster default-dsc
```

Add or update the `dashboard` component section:

```yaml
spec:
  components:
    dashboard:
      managementState: Managed
      hardwareProfiles:
        enabled: true
```

Save and verify the change rolls out.

```bash
oc get pods -n redhat-ods-applications -w
```

## Create a Hardware Profile

Define a Hardware Profile for a specific GPU type or node pool.

```bash
cat <<EOF | oc apply -f -
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: nvidia-a100
  namespace: redhat-ods-applications
spec:
  displayName: "NVIDIA A100"
  identifiers:
    - resourceType: nvidia.com/gpu
      identifier: "A100"
  nodeSelector:
    nvidia.com/gpu.product: NVIDIA-A100
  tolerations:
    - key: nvidia.com/gpu
      operator: Exists
      effect: NoSchedule
EOF
```

For CPU-optimized workloads, create a profile without GPU selectors.

```bash
cat <<EOF | oc apply -f -
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: cpu-optimized
  namespace: redhat-ods-applications
spec:
  displayName: "CPU Optimized"
  nodeSelector:
    node.kubernetes.io/instance-type: m6i.2xlarge
  tolerations: []
EOF
```

## Verify Hardware Profile

List all configured hardware profiles.

```bash
oc get hardwareprofiles -n redhat-ods-applications
```

Check the detailed configuration of a specific profile.

```bash
oc get hardwareprofile nvidia-a100 -n redhat-ods-applications -o yaml
```

## Use Hardware Profile in InferenceService

Reference the Hardware Profile in a KServe InferenceService to deploy to specific hardware.

```bash
cat <<EOF | oc apply -f -
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: my-model
  namespace: my-project
  annotations:
    serving.kserve.io/deploymentMode: RawDeployment
    opendatahub.io/hardwareProfile: nvidia-a100
spec:
  predictor:
    model:
      modelFormat:
        name: huggingface
      storageUri: s3://my-bucket/models/llm
      resources:
        limits:
          nvidia.com/gpu: 1
EOF
```

Verify the pod is scheduled on the correct nodes.

```bash
oc get pod -n my-project -l serving.kserve.io/inferenceservice=my-model -o wide
```

## Troubleshooting

If hardware profiles don't appear in the dashboard:

Check the dashboard pod logs for configuration errors.

```bash
oc logs -n redhat-ods-applications deployment/rhods-dashboard | grep -i "hardware"
```

Verify the node labels match the profile's nodeSelector.

```bash
oc get nodes -l nvidia.com/gpu.product=NVIDIA-A100
```

Check the InferenceService events if scheduling fails.

```bash
oc describe inferenceservice my-model -n my-project
```

## Related

- Product: Red Hat OpenShift AI 2.25+
- Component: Dashboard, KServe
- Tags: hardware, profile, rhoai-self-managed, GPU, accelerator
- KB Article: https://access.redhat.com/solutions/7133582
