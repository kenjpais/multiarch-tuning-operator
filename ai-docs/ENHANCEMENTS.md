# Enhancement Proposals & Design Documents

## Local Design Documents (docs/enhancements/)

| ID | Title | Link |
|----|-------|------|
| MTO-0001 | Multiarch Manager Operator | [docs/enhancements/MTO-0001.md](../docs/enhancements/MTO-0001.md) |
| MTO-0002 | Namespace-scoped PodPlacementConfig | [docs/enhancements/MTO-0002-local-pod-placement.md](../docs/enhancements/MTO-0002-local-pod-placement.md) |
| MTO-0003 | Support for non-OKD Kubernetes/CRI-O Clusters | [docs/enhancements/MTO-0003-support-non-ocp-clusters.md](../docs/enhancements/MTO-0003-support-non-ocp-clusters.md) |
| MTO-0004 | eBPF-based ENOEXEC Monitoring | [docs/enhancements/MTO-0004-enoexec-monitoring.md](../docs/enhancements/MTO-0004-enoexec-monitoring.md) |
| MTO-0005 | CEL Architecture Rules Plugin | [docs/enhancements/MTO-0005-architecture-rules-plugin.md](../docs/enhancements/MTO-0005-architecture-rules-plugin.md) |

## OpenShift Enhancement Proposals

| Title | Status | Link |
|-------|--------|------|
| Multiarch Manager Operator | Implemented | [openshift/enhancements: multi-arch/multiarch-manager-operator.md](https://github.com/openshift/enhancements/blob/master/enhancements/multi-arch/multiarch-manager-operator.md) |

## Related Upstream KEPs

| KEP | Title | Relevance |
|-----|-------|-----------|
| KEP-3521 | Pod Scheduling Readiness | Core mechanism — scheduling gates used to hold pods until architecture affinity is determined |
| KEP-3838 | Pod Mutable Scheduling Directives | Enables modifying pod scheduling directives (nodeAffinity) after creation while gated |

- [KEP-3521: Pod Scheduling Readiness](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/3521-pod-scheduling-readiness)
- [KEP-3838: Pod Mutable Scheduling Directives](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/3838-pod-mutable-scheduling-directives)

## Additional Documentation

| Document | Purpose | Link |
|----------|---------|------|
| Metrics & Monitoring | Prometheus metrics reference and Grafana dashboard | [docs/metrics.md](../docs/metrics.md) |
| Alert Runbooks | Per-component alert runbooks | [docs/alerts/](../docs/alerts/) |
| Non-OKD Support | Running on non-OpenShift clusters | [docs/support-non-okd.md](../docs/support-non-okd.md) |
| OCP Release Process | Release procedures for OpenShift | [docs/ocp-release.md](../docs/ocp-release.md) |
| ENoExec Monitoring Architecture | SVG architecture diagram | [docs/enhancements/enoexec-monitoring-architecture.svg](../docs/enhancements/enoexec-monitoring-architecture.svg) |
