# GPU as a Service on Red Hat OpenShift AI 3.2

This guide provides a practical, step-by-step approach to implementing GPU as a Service on Red Hat OpenShift AI (RHOAI) 3.2. GPU as a Service transforms expensive, underutilized GPU infrastructure into an efficient, governed allocation engine that maximizes ROI and ensures fair resource distribution.

## What Is GPU as a Service?

GPU as a Service is a governance framework that enables efficient GPU resource allocation through two primary strategies:

1. **Time-Slicing:** Share a single physical GPU across multiple concurrent users, maximizing density and reducing idle time.
2. **Multi-Instance GPU (MIG):** Partition large GPUs (A100, H100) into hardware-isolated slices, providing guaranteed performance and memory isolation.

**Key Benefits:**
- **Maximize ROI:** Serve 4-5x more users on the same hardware through intelligent sharing
- **Automated Fairness:** Integrate with Kueue for fair-share scheduling across teams
- **Industrialized Scale:** Abstract GPU complexity behind user-friendly profiles
- **Governance:** Enforce limits on GPU consumption and prevent resource hoarding

## Prerequisites

Before implementing GPU as a Service, ensure you have:

- Access to a **Red Hat OpenShift AI 3.2** cluster
- **Cluster-admin privileges** (to install Operators and configure GPU resources)
- The **Node Feature Discovery (NFD)** Operator installed and active
- The **NVIDIA GPU Operator** installed and configured
- The `oc` CLI tool installed in your terminal
- **OpenShift AI Administrator** privileges

## Quick Start: Enabling GPU as a Service

### Strategy 1: Time-Slicing (Maximum Density)

Time-slicing allows multiple users to share a single physical GPU by time-sharing the compute resources. This is ideal for development, notebooks, and inference workloads.

**Step 1: Configure GPU Operator for Time-Slicing**

Create a ConfigMap to define time-slicing configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: time-slicing-config
  namespace: nvidia-gpu-operator
data:
  any: |-
    version: v1
    sharing:
      timeSlicing:
        resources:
        - name: nvidia.com/gpu
          replicas: 4  # Share 1 GPU across 4 users
```

**Step 2: Patch the ClusterPolicy**

```bash
oc patch clusterpolicy cluster-policy \
  -n nvidia-gpu-operator \
  --type merge \
  -p '{"spec": {"devicePlugin": {"config": {"name": "time-slicing-config"}}}}'
```

**Step 3: Verify Time-Slicing**

```bash
oc describe node <gpu-node-name> | grep nvidia.com/gpu
```

*Expected Output:* `nvidia.com/gpu: 4` (indicating 4 virtual GPUs available)

### Strategy 2: Multi-Instance GPU (MIG) (Guaranteed Isolation)

MIG partitions large GPUs into hardware-isolated slices, providing guaranteed performance and memory isolation. Ideal for production workloads requiring predictable performance.

**Step 1: Enable MIG on GPU Nodes**

```bash
# Label the node for MIG
oc label node <node-name> nvidia.com/mig.config=all-1g.5gb --overwrite
```

**Step 2: Verify MIG Partitions**

```bash
oc describe node <node-name> | grep nvidia.com/mig
```

*Expected Output:* `nvidia.com/mig-1g.5gb: 7` (indicating 7 MIG slices available)

## Core Configuration Elements

### 1. Hardware Profiles for GPU Allocation

Create Hardware Profiles that expose GPU resources to users through the RHOAI dashboard:

```yaml
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: nvidia-shared-gpu
  namespace: redhat-ods-applications
spec:
  displayName: "Shared GPU (Time-Sliced)"
  description: "Best for notebooks and development. Compute is shared with neighbors."
  enabled: true
  identifiers:
    - identifier: nvidia.com/gpu
      displayName: "NVIDIA GPU (Shared)"
      defaultCount: 1
      minCount: 1
      maxCount: 1
  resourceLimits:
    - name: memory
      default: "8Gi"
      max: "16Gi"  # Critical: Prevent OOM in time-slicing
```

### 2. MIG Profile Configuration

For MIG-based allocation:

```yaml
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: nvidia-mig-small
  namespace: redhat-ods-applications
spec:
  displayName: "Small GPU Slice (MIG 1g.5gb)"
  description: "Dedicated hardware slice. 5GB vRAM. Good for inference and small models."
  enabled: true
  identifiers:
    - identifier: nvidia.com/mig-1g.5gb
      displayName: "NVIDIA MIG 1g.5gb"
      defaultCount: 1
      minCount: 1
      maxCount: 1
```

### 3. Fair-Share Scheduling with Kueue

Enable dynamic GPU sharing across teams using Kueue:

```yaml
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: nvidia-gpu-fairshare
  namespace: redhat-ods-applications
spec:
  displayName: "Shared GPU (Queue)"
  description: "Submit jobs to the fair-share queue. Resources allocated by priority."
  enabled: true
  identifiers:
    - identifier: nvidia.com/gpu
      displayName: "NVIDIA GPU"
      defaultCount: 1
      minCount: 1
      maxCount: 4
  allocationStrategy:
    kind: LocalQueue
    localQueue:
      name: "default"
```

## Verification and Troubleshooting

### Verify GPU Capacity

```bash
# Check Time-Slicing
oc describe node <node-name> | grep "nvidia.com/gpu"

# Check MIG
oc describe node <node-name> | grep "nvidia.com/mig"
```

### Verify Dashboard Integration

1. Log in to **OpenShift AI Dashboard**
2. Navigate to **Settings** → **Hardware profiles**
3. Confirm your GPU profiles are listed and **Enabled**

### Common Issues

**Issue: Time-Slicing replicas not showing up**
- **Symptom:** `oc describe node` shows `nvidia.com/gpu: 1` instead of expected count
- **Fix:** Verify the ConfigMap name matches the ClusterPolicy reference
- **Fix:** Check GPU Operator logs: `oc logs -n nvidia-gpu-operator -l app=gpu-operator-feature-discovery`

**Issue: "Silent OOM" in Time-Slicing**
- **Symptom:** User kernels dying randomly or "Kernel Restarting" messages
- **Root Cause:** Multiple users exceeding total GPU memory
- **Fix:** Enforce memory limits at container level or move users to MIG profiles

**Issue: MIG node stuck in SchedulingDisabled**
- **Symptom:** Node stuck in `SchedulingDisabled` or `NotReady` for hours
- **Root Cause:** GPU Operator cannot drain node due to running pods
- **Fix:** Check pending evictions: `oc get pods --all-namespaces --field-selector spec.nodeName=<node-name>`

**Issue: Profile not visible in dashboard**
- **Fix:** Verify `enabled: true` in HardwareProfile CR
- **Fix:** Verify correct namespace (`redhat-ods-applications` for global profiles)
- **Fix:** Check that identifiers match what NFD reports

## Complete Workflow Example

### Automated Time-Slicing Setup

```bash
# 1. Create Time-Slicing ConfigMap
oc apply -f time-slicing-config.yaml

# 2. Patch ClusterPolicy
oc patch clusterpolicy cluster-policy \
  -n nvidia-gpu-operator \
  --type merge \
  -p '{"spec": {"devicePlugin": {"config": {"name": "time-slicing-config"}}}}'

# 3. Wait for GPU Operator to apply changes
oc wait --for=condition=ready pod -l app=nvidia-device-plugin -n nvidia-gpu-operator --timeout=5m

# 4. Verify capacity
oc describe node <gpu-node> | grep nvidia.com/gpu

# 5. Create Hardware Profile
oc apply -f hardware-profile-shared.yaml

# 6. Verify in dashboard
# Navigate to Settings -> Hardware Profiles
```

## Next Steps

1. **Review the full course content** in Chapter 1 for detailed explanations and advanced scenarios
2. **Audit current GPU utilization** to identify optimization opportunities
3. **Implement fair-share scheduling** for teams sharing GPU resources
4. **Enable GPU sharing strategies** (Time-Slicing or MIG) to maximize ROI

## Additional Resources

- Full course content: See Chapter 1 modules for comprehensive guides
- Automation Lab: Step-by-step hands-on exercises
- Operations Guide: Day 2 operations, troubleshooting, and governance

---

**Ready to build your GPU allocation engine?** Start with Time-Slicing for maximum density, then explore MIG for guaranteed isolation in production workloads.
