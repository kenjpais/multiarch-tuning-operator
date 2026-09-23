# Review Instructions

## Scope

This operator provides architecture-aware pod scheduling for multi-arch clusters. Reviews should focus on correctness of scheduling logic, API contract preservation, and operand lifecycle safety.

## Critical Review Areas

- **Pod model logic** (`internal/controller/podplacement/pod_model.go`): Verify `shouldIgnorePod` exclusion rules, architecture set computation, and affinity generation
- **Webhook behavior** (`internal/controller/podplacement/scheduling_gate_mutating_webhook.go`): Verify namespace selector handling and scheduling gate addition
- **Operator reconciliation** (`internal/controller/operator/`): Verify finalizer ordering, resource apply logic, and status condition updates
- **API types** (`api/v1beta1/`): Verify validation webhooks, conversion logic, and backward compatibility

## Do not report

These are generated, vendored, or auto-managed — do not flag style or structure issues:

- `vendor/**` — vendored dependencies
- `**/zz_generated*.go` — controller-gen generated code
- `config/crd/bases/**` — generated CRD YAML
- `config/rbac/role.yaml` — generated RBAC
- `bundle/manifests/**` — generated OLM bundle
- `go.sum` — dependency checksums
- `hack/**` — build/CI scripts (review for security only)

## Platform conventions

Follow [openshift/enhancements](https://github.com/openshift/enhancements) conventions:
- API changes must follow `dev-guide/api-conventions.md`
- Status conditions follow the operator pattern (Available, Progressing, Degraded)
- RBAC markers must not be placed on operand code (see `internal/controller/podplacement/pod_reconciler.go:56-58`)

## Path-specific rules

| Path | Focus |
|------|-------|
| `api/**` | Backward compatibility, validation webhook completeness, CRD marker correctness |
| `internal/controller/podplacement/pod_model.go` | Exclusion logic correctness, architecture intersection, gate lifecycle |
| `internal/controller/operator/` | Finalizer ordering, resource apply method correctness, status condition transitions |
| `pkg/image/` | Registry auth handling, manifest parsing, cache invalidation |
| `pkg/utils/const.go` | Label/annotation key consistency, no duplicate definitions |
| `internal/controller/podplacement/cel_*.go` | CEL expression safety, cache bounds, evaluation correctness |
| `cmd/main.go` | Flag validation, mode exclusivity, startup sequence |

## High-churn areas (severity: elevated)

- `internal/controller/operator/clusterpodplacementconfig_controller.go` — operator reconciliation core
- `internal/controller/podplacement/pod_model.go` — pod processing logic
- `internal/controller/podplacement/cel_evaluator.go` — CEL plugin engine
- `bundle/manifests/multiarch-tuning-operator.clusterserviceversion.yaml` — OLM metadata

## Verification checklist for PRs

- [ ] `make test` passes (includes fmt, vet, lint, gosec, unit tests)
- [ ] `make manifests && make generate && make verify-diff` shows no changes
- [ ] API changes include conversion webhook updates if modifying v1alpha1↔v1beta1
- [ ] New controller patterns include unit tests with envtest
- [ ] Namespace exclusion changes preserve `shouldIgnorePod` contract
