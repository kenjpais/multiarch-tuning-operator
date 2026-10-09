# Multiarch Tuning Operator

**Purpose**: Architecture-aware pod placement in multi-architecture Kubernetes clusters

**Repository**: https://github.com/openshift/multiarch-tuning-operator

## What This Operator Does

The Multiarch Tuning Operator automatically schedules workloads on compatible CPU architectures by:

1. **Inspecting container images** to determine supported architectures (amd64, arm64, ppc64le, s390x)
2. **Setting nodeAffinity** constraints based on image architectures
3. **Preventing exec format errors** from incompatible pod-to-node placement

**Result**: Workloads run on the right architecture without developer intervention.

## Quick Start

```bash
# Deploy operator
make deploy IMG=quay.io/yourorg/multiarch-tuning-operator:latest

# Enable pod placement
kubectl create -f - <<EOF
apiVersion: multiarch.openshift.io/v1beta1
kind: ClusterPodPlacementConfig
metadata:
  name: cluster
spec:
  logVerbosity: Normal
  namespaceSelector:
    matchExpressions:
      - key: multiarch.openshift.io/exclude-pod-placement
        operator: DoesNotExist
EOF

# Create a pod → automatically gated, architectures detected, nodeAffinity set
kubectl run nginx --image=nginx:latest
```

## Core Components

### [Operator Controller](agentic/design-docs/components/operator-controller.md)
**Manages** ClusterPodPlacementConfig CR lifecycle  
**Deploys** pod placement controller and webhook operands  
**Mode**: `--enable-operator`

### [Pod Placement Controller](agentic/design-docs/components/pod-placement-controller.md)
**Watches** pods with scheduling gates  
**Inspects** container images from registries  
**Sets** nodeAffinity for architecture constraints  
**Mode**: `--enable-ppc-controllers`

### [Pod Placement Webhook](agentic/design-docs/components/pod-placement-webhook.md)
**Intercepts** pod creation via mutating admission  
**Adds** scheduling gates to hold pods before scheduling  
**Mode**: `--enable-ppc-webhook`

### [Image Inspector](agentic/design-docs/components/image-inspector.md)
**Fetches** image manifests from container registries  
**Parses** OCI/Docker manifest lists  
**Returns** set of supported architectures  
**Location**: `pkg/image/inspector.go`

## Key Concepts

### [Scheduling Gates](agentic/domain/concepts/scheduling-gates.md)
Hold pods before scheduling to allow architecture analysis

### [Image Inspection](agentic/domain/concepts/image-inspection.md)
Determine supported architectures by examining registry manifests

### [Node Affinity](agentic/domain/concepts/node-affinity.md)
Kubernetes scheduler constraints based on architecture labels

### [ClusterPodPlacementConfig API](agentic/domain/concepts/clusterpodplacementconfig-api.md)
Singleton CR controlling pod placement operand behavior

## Workflows

### [Pod Placement Flow](agentic/domain/workflows/pod-placement.md)
**End-to-end**: Pod creation → gating → inspection → nodeAffinity → ungating → scheduling

## Development

### [Development Setup](agentic/DEVELOPMENT.md)
**Prerequisites**, build commands, local vs containerized testing

### [Testing Strategy](agentic/TESTING.md)
**Unit tests** (Ginkgo + envtest), **E2E tests** (deployed operator)

### [Design Philosophy](agentic/DESIGN.md)
**Core beliefs**: Zero-touch multi-arch support, safety through gating, operator-of-operators pattern

## Operations

### [Reliability & Observability](agentic/RELIABILITY.md)
**SLOs**, Prometheus metrics, alerting, failure modes

### [Security Model](agentic/SECURITY.md)
**Threat model**, RBAC, pull secret handling, admission control

## Architecture Decisions

**Webhook + Controller Duality**: Webhook adds gates fast (< 500ms), controller processes async (supports slow image inspection)

**Image Inspection Over Heuristics**: Inspect actual manifests instead of labels/conventions for ground truth

**Operator-of-Operators**: Operator deploys operands rather than reconciling directly (enables independent scaling)

## Binary Execution Modes

The operator runs in four mutually exclusive modes:

1. **`--enable-operator`**: Manages ClusterPodPlacementConfig CR (operator controller)
2. **`--enable-ppc-controllers`**: Reconciles gated pods (pod placement controller)
3. **`--enable-ppc-webhook`**: Adds scheduling gates (mutating webhook)
4. **`--enable-enoexec-event-controllers`**: Monitors exec format errors (eBPF daemon + handler)

## Excluded Namespaces

System namespaces **always excluded** (cannot be overridden):
- `openshift-*`
- `kube-*`
- `hypershift-*`

Additional exclusions via namespace label: `multiarch.openshift.io/exclude-pod-placement`

## Metrics

All components expose Prometheus metrics at `:8080/metrics`:
- `mto_ppo_ctrl_time_to_process_gated_pod_seconds` - Pod ungating latency
- `mto_ppo_ctrl_time_to_inspect_image_seconds` - Image inspection time
- `mto_ppo_wh_response_time_seconds` - Webhook response time

See [docs/metrics.md](docs/metrics.md) for Grafana dashboards.

## Related Documentation

- [OpenShift Enhancement Proposal](https://github.com/openshift/enhancements/blob/master/enhancements/multi-arch/multiarch-manager-operator.md)
- [KEP-3521: Pod Scheduling Readiness](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/3521-pod-scheduling-readiness)
- [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## Contributing

See [DEVELOPMENT.md](agentic/DEVELOPMENT.md) for build instructions and test commands.

## License

Apache License 2.0 - Copyright 2023 Red Hat, Inc.
