# Code Review Instructions

## Priority Areas

1. **Resource apply method correctness**: RBAC/webhook resources must use `pkg/utils/resource.go` `ApplyResource()` (library-go `resourceapply`). CR status/finalizer updates must use controller-runtime `r.Update()`. Never cross these paths.
2. **Namespace exclusion logic**: Only `kube-*` and the operator namespace are hardcoded exclusions in `shouldIgnorePod`. Verify changes don't accidentally exclude or include namespaces.
3. **Scheduling gate lifecycle**: Gate must be removed after nodeAffinity is set. Orphaned gates block pod scheduling permanently.
4. **Image inspection error handling**: Verify retry logic and `fallbackArchitecture` behavior on inspection failures.
5. **CEL expression safety**: CEL programs are cached (LRU 1024). Verify new expressions compile correctly and cache invalidation is handled.

## Do not report

- Style issues enforced by CI (`make fmt`, `make goimports`, `make lint`, `make vet`)
- Generated file modifications (`**/zz_generated*`, `config/crd/bases/**`, `config/rbac/role.yaml`)
- Vendored dependency code (`vendor/**`)
- Test helper boilerplate (`pkg/testing/**`)
- Bundle manifest formatting (`bundle/**`)

## Path-specific rules

### `api/v1beta1/` — CRD type definitions
- Verify kubebuilder markers match intended validation
- Check that new fields have proper `+optional` or `+kubebuilder:validation:Required` markers
- Ensure DeepCopy is regenerated (`make generate`)

### `internal/controller/operator/` — Operator reconciler
- Verify resource builders in `objects.go` produce correct RBAC and deployment specs
- Check finalizer add/remove ordering (pod ungating must precede operand deletion)
- Verify status condition updates cover all paths

### `internal/controller/podplacement/` — Pod reconciler and webhook
- Verify nodeAffinity mutation is correct (in-place update, not replacement of existing terms)
- Check `MaxConcurrentReconciles` is appropriate for I/O-bound workloads
- Verify webhook path registration matches MutatingWebhookConfiguration

### `pkg/utils/resource.go` — Resource apply dispatcher
- Each resource type must route to the correct `resourceapply.*` function
- New resource types need a case in the type switch
- `resourceCache` must be passed for idempotent applies

### `pkg/image/` — Image inspection
- Verify registry auth handling (pull secrets, certificates)
- Check cache invalidation on pull secret changes
- CGO/gpgme dependency awareness

### `cmd/main.go` — Binary entrypoint
- Only one execution mode flag may be set at a time (validated by `validateFlags()`)
- Verify new flags are registered in `bindFlags()` and validated
- Leader election IDs are deterministic; changes affect rolling upgrades

### `api/v1alpha1/` — Alpha API with conversion
- Changes must maintain conversion compatibility with v1beta1
- Conversion logic in `clusterPodPlacementConfig_conversion.go`

### `internal/controller/enoexecevent/` — eBPF monitoring
- eBPF tracepoint code requires careful review for kernel compatibility
- ENoExecEvent CR lifecycle must handle node failures gracefully

## Platform conventions

- Follow [dev-guide/api-conventions.md](https://github.com/openshift/enhancements/blob/master/dev-guide/api-conventions.md) for API changes
- Follow [CONVENTIONS.md](https://github.com/openshift/enhancements/blob/master/CONVENTIONS.md) for coding standards
