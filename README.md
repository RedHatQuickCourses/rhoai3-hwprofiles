# Hardware Profiles in Red Hat OpenShift AI 3: Execution and Creation Guide

This guide provides a practical, step-by-step approach to creating and executing hardware profiles in Red Hat OpenShift AI (RHOAI) 3. Hardware profiles transform complex Kubernetes resource management into a simple, governed menu for data scientists, enabling efficient allocation of compute resources like GPUs, CPUs, and memory.

## What Are Hardware Profiles?

Hardware profiles are Custom Resources (CRs) in RHOAI that serve as an abstraction layer between physical infrastructure and end-users. Instead of manually configuring Kubernetes pod specs with taints, tolerations, and resource limits, users simply select a profile from a dropdown menu (e.g., "Small CPU", "NVIDIA A100 GPU", "Large Memory").

**Key Benefits:**
- **Governance:** Enforce limits on compute resource consumption
- **Simplicity:** Hide Kubernetes complexity behind user-friendly names
- **Efficiency:** Ensure workloads are scheduled on correct hardware automatically
- **ROI Maximization:** Enable GPU sharing strategies (Time-Slicing, MIG) to serve more users

## Prerequisites

Before creating hardware profiles, ensure you have:

- Access to a **Red Hat OpenShift AI 3.2** cluster
- **Cluster-admin privileges** (to install Operators and apply node labels)
- The **Node Feature Discovery (NFD)** Operator installed and active
- The **NVIDIA GPU Operator** installed (if creating GPU profiles)
- The `oc` CLI tool installed in your terminal
- **OpenShift AI Administrator** privileges

## Quick Start: Creating Your First Hardware Profile

### Method 1: Using the OpenShift AI Dashboard (UI)

1. Log in to the **OpenShift AI Dashboard**
2. Navigate to **Settings** → **Hardware profiles**
3. Click **Create hardware profile**
4. Fill in the configuration fields:
   - **Display Name:** User-facing name (e.g., "NVIDIA A100 - Large")
   - **Description:** Guidance for users (e.g., "Use this for Large Language Model training")
   - **Identifiers:** Add resource identifiers (e.g., `nvidia.com/gpu`)
   - **Resource Limits:** Set CPU and Memory defaults and maximums
   - **Tolerations/Node Selectors:** Configure if targeting specific nodes
5. Click **Create** to make the profile available

### Method 2: Using YAML (GitOps/Code-Based)

Create a YAML file defining the HardwareProfile Custom Resource:

```yaml
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: nvidia-a100-isolated
  namespace: redhat-ods-applications  # Global scope
spec:
  displayName: "Exclusive - NVIDIA A100"
  description: "Guaranteed access to isolated A100 nodes."
  enabled: true
  identifiers:
    - identifier: nvidia.com/gpu
      displayName: "NVIDIA A100 GPU"
      defaultCount: 1
      minCount: 1
      maxCount: 4
  resourceLimits:
    - name: cpu
      default: "4"
      max: "8"
    - name: memory
      default: "16Gi"
      max: "32Gi"
  tolerations:
    - key: "nvidia.com/gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
```

Apply the profile:

```bash
oc apply -f profile-isolation.yaml
```

## Core Configuration Elements

### 1. Identifiers (Target Hardware)

Identifiers tell Kubernetes *what* hardware to request. Common examples:

- `nvidia.com/gpu` - Generic NVIDIA GPU
- `nvidia.com/mig-1g.5gb` - Specific MIG partition
- `intel.com/gaudi` - Intel Gaudi accelerator

**Discovery:** Use Node Feature Discovery (NFD) to identify available hardware:

```bash
oc get nodes --show-labels | grep nvidia
```

### 2. Resource Limits (Sizing)

Define CPU and Memory constraints to prevent resource hoarding:

- **Default:** Pre-filled value users see
- **Minimum (Request):** Guaranteed allocation
- **Maximum (Limit):** Hard ceiling to prevent starvation

**Sizing Strategy Example:**

| Profile Name | Use Case | CPU (Req/Lim) | Memory (Req/Lim) | Accelerator |
|--------------|----------|---------------|------------------|-------------|
| Standard | Data exploration | 2 / 4 Cores | 8GB / 16GB | None |
| Training | Deep learning | 8 / 16 Cores | 32GB / 64GB | 1x NVIDIA GPU |
| Large | LLM Fine-tuning | 24 / 48 Cores | 128GB / 256GB | 2x NVIDIA GPU |

### 3. Taints and Tolerations (Isolation)

Use this "lock and key" mechanism to reserve expensive hardware:

**Step 1: Apply Taint to Node (The Lock)**
```bash
oc adm taint nodes <node-name> nvidia.com/gpu=true:NoSchedule
```

**Step 2: Add Toleration to Profile (The Key)**
```yaml
tolerations:
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### 4. Node Selectors (Placement)

Pin workloads to specific node labels:

```yaml
nodeSelector:
  gpu-type: "A100-80GB"
```

**Apply Custom Labels:**
```bash
oc label node <node-name> gpu-type=A100-80GB
```

## Advanced Strategies

### Strategy 1: Fair-Share Scheduling with Kueue

Enable dynamic resource sharing across teams using Kueue:

```yaml
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: nvidia-a100-fairshare
  namespace: redhat-ods-applications
spec:
  displayName: "Shared - NVIDIA A100 (Queue)"
  description: "Submit jobs to the fair-share queue. Resources allocated by priority."
  enabled: true
  identifiers:
    - identifier: nvidia.com/gpu
      displayName: "NVIDIA A100 GPU"
      defaultCount: 1
      minCount: 1
      maxCount: 4
  allocationStrategy:
    kind: LocalQueue
    localQueue:
      name: "default"  # Must match a LocalQueue in the user's namespace
```

**Important:** You *cannot* combine `tolerations` or `nodeSelectors` with `LocalQueue` strategy. Choose one approach.

### Strategy 2: GPU Sharing with Time-Slicing

Maximize density by sharing a single GPU across multiple users:

1. Configure GPU Operator for time-slicing (4 replicas per GPU)
2. Create profile targeting `nvidia.com/gpu` identifier
3. Set strict memory limits to prevent OOM crashes

```yaml
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: nvidia-shared-gpu
spec:
  displayName: "Shared GPU (Time-Sliced)"
  description: "Best for notebooks and development. Compute is shared with neighbors."
  identifiers:
    - identifier: nvidia.com/gpu
      count: 1
  resourceLimits:
    - name: memory
      default: "8Gi"
      max: "16Gi"  # Critical: Prevent one user from consuming all VRAM
```

### Strategy 3: Multi-Instance GPU (MIG)

Use hardware-isolated GPU partitions for guaranteed performance:

1. Enable MIG on A100/H100 GPUs via GPU Operator
2. Identify MIG resource name (e.g., `nvidia.com/mig-1g.5gb`)
3. Create profile targeting the specific MIG identifier

```yaml
apiVersion: dashboard.opendatahub.io/v1alpha1
kind: HardwareProfile
metadata:
  name: nvidia-mig-small
spec:
  displayName: "Small GPU Slice (MIG 1g.5gb)"
  description: "Dedicated hardware slice. 5GB vRAM. Good for inference and small models."
  identifiers:
    - identifier: nvidia.com/mig-1g.5gb  # Specific MIG slice name
      count: 1
```

## Governance: Global vs. Project-Scoped Profiles

### Global Profiles (Public Utility)

Visible to **all users** in the OpenShift AI dashboard. Use for generic resources.

- **Namespace:** `redhat-ods-applications`
- **Use Case:** General development, interactive notebooks, training

### Project-Scoped Profiles (Private Reserve)

Visible *only* to users within a specific Data Science Project. Use for specialized hardware.

- **Namespace:** User's project namespace (e.g., `finance-models-prod`)
- **Use Case:** Sensitive workloads, reserved capacity, compliance requirements

## Verification and Troubleshooting

### Verify Profile Creation

```bash
oc get hardwareprofiles -n redhat-ods-applications
```

### Check Pod Configuration

Inspect a running workbench pod to verify profile settings are applied:

```bash
oc get pod <workbench-pod-name> -o yaml
```

Look for:
- **Resource Limits:** Under `spec.containers.resources`
- **Tolerations:** Under `spec.tolerations`
- **Queue Label:** `kueue.x-k8s.io/queue-name` (if using Kueue)

### Common Issues

**Issue: Workbench stuck in "Pending" state**
- **Kueue Quota:** Check if project has exhausted quota
- **Physical Capacity:** Verify nodes exist with matching labels: `oc get nodes -l <selector-key>=<selector-value>`

**Issue: "Unschedulable" - Taint mismatch**
- **Symptom:** `0/5 nodes are available: 5 node(s) had taint {nvidia.com/gpu: true}, that the pod didn't tolerate`
- **Fix:** Add matching toleration to Hardware Profile YAML

**Issue: Profile not visible in dashboard**
- Verify `enabled: true` in CR spec
- Verify correct namespace (`redhat-ods-applications` for global)
- Check that identifiers match what NFD reports

## Enabling Hardware Profiles Feature

If the feature is not visible in the dashboard:

1. Access **OpenShift Console** (Administrator view)
2. Find the `OdhDashboardConfig` custom resource
3. Set `disableHardwareProfiles: false`
4. Refresh the OpenShift AI dashboard

## Complete Workflow Example

### Automated Discovery and Isolation

```bash
# 1. Discover accelerator nodes
oc get nodes -l nvidia.com/gpu.present=true -o jsonpath='{.items[*].metadata.name}'

# 2. Apply taint (lock)
oc adm taint nodes <node-name> nvidia.com/gpu=true:NoSchedule --overwrite

# 3. Verify taint
oc describe node <node-name> | grep Taints

# 4. Create and apply profile
oc apply -f profile-isolation.yaml

# 5. Verify profile
oc get hardwareprofiles -n redhat-ods-applications
```

## Next Steps

1. **Review the full course content** in Chapter 1 for detailed explanations and advanced scenarios
2. **Audit current utilization** to identify optimization opportunities
3. **Implement fair-share scheduling** for teams sharing resources
4. **Enable GPU sharing strategies** (Time-Slicing or MIG) to maximize ROI

## Additional Resources

- Full course content: See Chapter 1 modules for comprehensive guides
- Automation Lab: Step-by-step hands-on exercises
- Operations Guide: Day 2 operations, troubleshooting, and governance

---

**Ready to build your allocation engine?** Start with the dashboard UI for your first profile, then move to YAML-based definitions for version control and GitOps workflows.
