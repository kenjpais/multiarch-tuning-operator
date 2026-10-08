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

| Title | Status | Link | Tracking |
|-------|--------|------|----------|
| Multiarch Manager Operator | Implemented (shipping since OCP 4.15) | [openshift/enhancements: multi-arch/multiarch-manager-operator.md](https://github.com/openshift/enhancements/blob/master/enhancements/multi-arch/multiarch-manager-operator.md) | [MIXEDARCH-215](https://issues.redhat.com/browse/MIXEDARCH-215) |

> **Verified 2026-09-25**: The enhancement document exists in `openshift/enhancements` at the referenced path. Authors: @aleskandro and @Prashanth684. Reviewers include @bparees (overall design), @deads2k (apiserver/scheduler), @joelanford (OLM), @dmage (image inspection), @ingvagabund (scheduling). The enhancement does not carry a formal `status:` field in its YAML frontmatter (unlike upstream KEPs), but MTO is a shipping product through OCP 4.15+ and has reached v1.3.x releases.

## Related Upstream KEPs

| KEP | Title | Upstream Status | Stage | GA Milestone | Relevance |
|-----|-------|----------------|-------|-------------|-----------|
| KEP-3521 | Pod Scheduling Readiness | **Implemented** | **Stable** | **v1.30** (Alpha v1.26, Beta v1.27) | Core mechanism — scheduling gates used to hold pods until architecture affinity is determined |
| KEP-3838 | Pod Mutable Scheduling Directives | **Implementable** | **Stable** | **v1.30** (Beta v1.27) | Enables modifying pod scheduling directives (nodeAffinity) after creation while gated |

- [KEP-3521: Pod Scheduling Readiness](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/3521-pod-scheduling-readiness) — authored by @Huang-Wei (sig-scheduling), approved by @ahg-g, @smarterclayton. Feature gate: `PodSchedulingReadiness` (kube-apiserver, kube-scheduler). GA metric: `scheduler_pending_pods{queue="gated"}`.
- [KEP-3838: Pod Mutable Scheduling Directives](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/3838-pod-mutable-scheduling-directives) — authored by @ahg-g (sig-scheduling), approved by @Huang-Wei. Shares the `PodSchedulingReadiness` feature gate. Related to KEP-2926 (Job Mutable Scheduling Directives).

> **Verified 2026-09-25**: Both KEP `kep.yaml` files were read from `kubernetes/enhancements` master branch. KEP-3521 status=`implemented` stage=`stable` latest-milestone=`v1.30`. KEP-3838 status=`implementable` stage=`stable` latest-milestone=`v1.30`. Both KEPs are GA since Kubernetes 1.30, meaning the scheduling gates MTO depends on are stable API and the `PodSchedulingReadiness` feature gate is locked to `true` (cannot be disabled) in K8s 1.30+. This is critical: MTO requires K8s 1.27+ (when scheduling gates went beta and were enabled by default).

## Additional Documentation

| Document | Purpose | Link |
|----------|---------|------|
| Metrics & Monitoring | Prometheus metrics reference and Grafana dashboard | [docs/metrics.md](../docs/metrics.md) |
| Alert Runbooks | Per-component alert runbooks | [docs/alerts/](../docs/alerts/) |
| Non-OKD Support | Running on non-OpenShift clusters | [docs/support-non-okd.md](../docs/support-non-okd.md) |
| OCP Release Process | Release procedures for OpenShift | [docs/ocp-release.md](../docs/ocp-release.md) |
| ENoExec Monitoring Architecture | SVG architecture diagram | [docs/enhancements/enoexec-monitoring-architecture.svg](../docs/enhancements/enoexec-monitoring-architecture.svg) |
