# Multiarch Tuning Operator (MTO)

**Repository**: `openshift/multiarch-tuning-operator` | **Default branch**: `main` | **Go**: 1.26 | **Framework**: controller-runtime + Kubebuilder

Architecture-aware pod scheduling for multi-architecture OpenShift/Kubernetes clusters. Adds node affinity based on container image architectures via scheduling gates ([KEP-3521](https://github.com/kubernetes/enhancements/blob/afad6f270c7ac2ae853f4d1b72c379a6c3c7c042/keps/sig-scheduling/3521-pod-scheduling-readiness/README.md)).

## Critical Warnings

1. **NEVER assume uniform apply methods** — operator controller uses library-go `resourceapply` via `utils.ApplyResources()` for operand resources; pod controller uses `r.Update` to patch pods after gate removal
2. **NEVER hand-edit `zz_generated.deepcopy.go`** — run `make generate` after API type changes
3. **NEVER omit `namespaceSelector`** on ClusterPodPlacementConfig — without it, the webhook gates pods in `openshift-*` namespaces, blocking critical workloads
4. **NEVER build without CGO** — image inspection requires `gpgme` (`gpgme-devel` / `libgpgme-dev`)
5. **NEVER set multiple `--enable-*` flags** — binary runs in exactly one mode per process

## Architecture at a Glance

Four mutually exclusive binary modes: Operator (`--enable-operator`) manages CR lifecycle and deploys operands; PPC Controllers (`--enable-ppc-controllers`) reconcile gated pods; PPC Webhook (`--enable-ppc-webhook`) gates new pods; ENoExec Controllers (`--enable-enoexec-event-controllers`) handle exec format errors via eBPF.

## Documentation

| Need | Start here |
|------|-----------|
| Architecture, API contracts, integrations | [ai-docs/ARCHITECTURE.md](ai-docs/ARCHITECTURE.md) |
| Build, deploy, common tasks | [ai-docs/DEVELOPMENT.md](ai-docs/DEVELOPMENT.md) |
| Test suites, patterns, running tests | [ai-docs/TESTING.md](ai-docs/TESTING.md) |
| Enhancement proposals | [ai-docs/ENHANCEMENTS.md](ai-docs/ENHANCEMENTS.md) |
| Metrics reference | [docs/metrics.md](docs/metrics.md) |
| OCP release process | [docs/ocp-release.md](docs/ocp-release.md) |
| Alert runbooks | [docs/alerts/](docs/alerts/) |
| Non-OKD cluster setup | [docs/support-non-okd.md](docs/support-non-okd.md) |

## Key Files

| File | Purpose |
|------|---------|
| `cmd/main.go` | Entrypoint, mode flags, manager setup |
| `api/v1beta1/clusterpodplacementconfig_types.go` | ClusterPodPlacementConfig CRD (hub) |
| `api/v1beta1/podplacementconfig_types.go` | PodPlacementConfig CRD (namespaced) |
| `internal/controller/operator/` | Operator mode controller |
| `internal/controller/podplacement/pod_reconciler.go` | Pod reconciliation logic |
| `internal/controller/podplacement/pod_model.go` | Pod processing, image inspection, affinity |
| `internal/controller/podplacement/scheduling_gate_mutating_webhook.go` | Webhook adding scheduling gates |
| `pkg/image/` | Container image architecture inspection |
| `pkg/utils/const.go` | All operator constants and label keys |

## External References

- [OpenShift Enhancement](https://github.com/openshift/enhancements/blob/6cebc13f0672c601ebfae669ea4fc8ca632721b5/enhancements/multi-arch/multiarch-manager-operator.md)
- [Platform conventions](https://github.com/openshift/enhancements) (`dev-guide/`, `guidelines/`, `CONVENTIONS.md`)
