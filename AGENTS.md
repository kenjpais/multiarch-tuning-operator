# Multiarch Tuning Operator (MTO)

**Repository**: `github.com/openshift/multiarch-tuning-operator`
**Purpose**: Architecture-aware pod scheduling for multi-arch OpenShift/Kubernetes clusters. Adds `kubernetes.io/arch` nodeAffinity to pods via scheduling gates and image inspection.

## Critical Warnings

1. **NEVER** set more than one execution mode flag at a time (`--enable-operator`, `--enable-ppc-controllers`, `--enable-ppc-webhook`, `--enable-enoexec-event-controllers`)
2. **NEVER** hand-edit `zz_generated.deepcopy.go` files — run `make generate`
3. **NEVER** assume `openshift-*` namespaces are excluded from pod placement -- only `kube-*` and the operator namespace are hardcoded; others require `namespaceSelector` on the CR
4. **NEVER** build without CGO — the `containers/image` library requires `gpgme-devel` (RHEL) or `libgpgme-dev` (Debian)
5. **NEVER** mix `resourceapply` (RBAC, webhooks) with `controller-runtime` Update (CR status, finalizers) — see ARCHITECTURE.md §Resource Management

## Architecture at a Glance

Single binary, four mutually exclusive modes: Operator → deploys operands (controller + webhook deployments). Pod Placement Controller → reconciles gated pods. Pod Placement Webhook → adds scheduling gates. ENoExecEvent Controllers → eBPF exec-format-error monitoring. The operator uses `library-go/resourceapply` for RBAC/webhook resources and `controller-runtime` client for CR status/finalizer updates. The pod reconciler uses `MaxConcurrentReconciles = NumCPU * 4` (I/O-bound image inspection). CEL-based architecture rules are evaluated via `PodPlacementConfig` (namespaced).

## Documentation

| Need | Start here |
|------|-----------|
| Internals, integrations, behavioral contracts | [ai-docs/ARCHITECTURE.md](ai-docs/ARCHITECTURE.md) |
| Build, common tasks, env vars | [ai-docs/DEVELOPMENT.md](ai-docs/DEVELOPMENT.md) |
| Test suites and patterns | [ai-docs/TESTING.md](ai-docs/TESTING.md) |
| Enhancement proposals / design docs | [ai-docs/ENHANCEMENTS.md](ai-docs/ENHANCEMENTS.md) |
| Metrics and monitoring | [docs/metrics.md](docs/metrics.md) |
| Alert runbooks | [docs/alerts/](docs/alerts/) |
| Non-OKD cluster support | [docs/support-non-okd.md](docs/support-non-okd.md) |
| OCP release process | [docs/ocp-release.md](docs/ocp-release.md) |

## Key Files

| File | Purpose |
|------|---------|
| `cmd/main.go` | Binary entrypoint, flag parsing, mode selection |
| `internal/controller/operator/clusterpodplacementconfig_controller.go` | Operator reconciler — deploys/manages operands |
| `internal/controller/podplacement/pod_reconciler.go` | Pod reconciler — image inspection, nodeAffinity |
| `internal/controller/podplacement/scheduling_gate_mutating_webhook.go` | Mutating webhook — adds scheduling gate |
| `internal/controller/podplacement/cel_evaluator.go` | CEL expression engine for architecture rules |
| `pkg/utils/resource.go` | `ApplyResource` — library-go resourceapply dispatcher |
| `api/v1beta1/clusterpodplacementconfig_types.go` | ClusterPodPlacementConfig CRD types |
| `api/v1beta1/podplacementconfig_types.go` | PodPlacementConfig (namespaced) CRD types |

## External References

- [OpenShift Enhancement](https://github.com/openshift/enhancements/blob/master/enhancements/multi-arch/multiarch-manager-operator.md)
- [KEP-3521: Pod Scheduling Readiness](https://github.com/kubernetes/enhancements/tree/master/keps/sig-scheduling/3521-pod-scheduling-readiness)
- Platform docs: [openshift/enhancements dev-guide/](https://github.com/openshift/enhancements/tree/master/dev-guide), [CONVENTIONS.md](https://github.com/openshift/enhancements/blob/master/CONVENTIONS.md)
