# Development Guide

## Prerequisites

- **Go**: 1.26.7 (see `go.mod`)
- **CGO**: Required — `containers/image` library needs `gpgme-devel` (RHEL/Fedora) or `libgpgme-dev` (Debian/Ubuntu)
- **Container runtime**: Podman (preferred) or Docker — set `FORCE_DOCKER=1` to use Docker
- **Default branch**: `main`
- **Vendoring**: Uses Go vendor (`GOFLAGS=-mod=vendor`)

## Build Commands

```bash
# Local build (requires gpgme-devel)
make build

# Run all checks + unit tests (fmt, vet, goimports, gosec, lint, unit)
make test

# Unit tests only
make unit

# Single-architecture image build
make docker-build IMG=<registry>/multiarch-tuning-operator:tag

# Multi-architecture image build (requires qemu-user-static)
make docker-buildx IMG=<registry>/multiarch-tuning-operator:tag
```

### Containerized vs Local Execution

By default, `make test`, `make unit`, and `make build` run inside a container using `BUILD_IMAGE`. To run locally:

```bash
# Option 1: Environment variable
NO_DOCKER=1 make test

# Option 2: .env file (see dotenv.example)
echo "NO_DOCKER=1" > .env
make test
```

## Common Tasks

### Add a new API field

1. Edit type definitions in `api/v1beta1/clusterpodplacementconfig_types.go` or `api/v1beta1/podplacementconfig_types.go`
2. Add kubebuilder markers for validation
3. Run `make generate` to regenerate `zz_generated.deepcopy.go`
4. Run `make manifests` to regenerate CRDs in `config/crd/bases/` and RBAC in `config/rbac/`
5. If the field affects v1alpha1, update conversion in `api/v1alpha1/clusterPodPlacementConfig_conversion.go`
6. Update bundle: `make bundle VERSION=<version>`
7. Run `make verify-diff` to ensure all generated files are committed

### Add a new operand resource

1. Define the resource builder function in `internal/controller/operator/objects.go` (or `podplacement_objects.go` / `enoexecevent_objects.go`)
2. Add it to the reconciliation loop in `clusterpodplacementconfig_controller.go`
3. Add the resource type case to `pkg/utils/resource.go` `ApplyResource()` if using library-go resourceapply
4. If it needs deletion on CPPC removal, add to the `toDelete` list with a `ToDeleteRef`

### Add a new metric

1. Define the metric in `internal/controller/podplacement/metrics/` or the appropriate controller package
2. Register it in the metrics package init
3. Document in `docs/metrics.md`
4. If alerting is needed, add PrometheusRule in the operator's resource builders

### Run a single test

```bash
# Focus on specific test by pattern
GINKGO_ARGS="-v --focus='your test pattern'" make unit

# Run tests for a specific package (local execution)
NO_DOCKER=1 go test ./internal/controller/podplacement/... -v
```

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `NO_DOCKER` | unset | Run builds/tests locally instead of containerized |
| `FORCE_DOCKER` | unset | Use Docker instead of Podman |
| `BUILD_IMAGE` | `registry.access.redhat.com/ubi9/go-toolset:1.26.7` | Builder image for containerized builds |
| `RUNTIME_IMAGE` | `registry.access.redhat.com/ubi9/ubi-minimal:latest` | Runtime base image |
| `NAMESPACE` | `default` (typically set to `openshift-multiarch-tuning-operator` in deployment) | Operator namespace (read at runtime via `utils.Namespace()`) |
| `IMAGE` | (from Makefile) | Operator image reference (read at runtime via `utils.Image()`) |
| `ARTIFACT_DIR` | `./_output` | Test artifact output directory |
| `KUBECONFIG` | — | Required for E2E tests |

## Code Generation

```bash
make generate    # DeepCopy implementations (zz_generated.deepcopy.go)
make manifests   # CRDs, RBAC, webhook configs
make bundle VERSION=<ver>  # OLM bundle (CSV, CRDs, manifests)
make vendor      # Update vendored dependencies
make verify-diff # Verify no uncommitted generated changes
```

## Deployment

```bash
# Install CRDs
make install

# Deploy operator
make deploy IMG=<registry>/multiarch-tuning-operator:tag

# Create singleton CR to enable pod placement
kubectl apply -f - <<EOF
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

# Undeploy
make undeploy

# Uninstall CRDs
make uninstall
```

## Bundle / Catalog Operations

```bash
make bundle VERSION=<version>
make bundle-verify                    # Verify deterministic generation
make bundle-build BUNDLE_IMG=<img>    # Build bundle image
make bundle-push BUNDLE_IMG=<img>     # Push bundle image
make catalog-build CATALOG_IMG=<img>  # Build catalog image
make catalog-push CATALOG_IMG=<img>   # Push catalog image
```

## Common Mistakes

1. **Building without gpgme**: `make build` fails with linker errors if `gpgme-devel` is not installed. The `containers/image` library (used for registry manifest inspection) requires CGO and gpgme.
2. **Forgetting `make generate` after API changes**: DeepCopy methods will be stale, causing runtime panics or compilation errors. Always run `make generate` then `make manifests`.
3. **Editing generated files**: Files matching `zz_generated*` and `config/crd/bases/*.yaml` are overwritten by `make generate` / `make manifests`. Changes will be lost.
4. **Running E2E without deployed operator**: E2E tests require the operator running in a cluster. Set `KUBECONFIG` and `NAMESPACE` before running `make e2e`.
5. **Assuming namespace exclusions**: Only `kube-*` and the operator namespace are hardcoded. Without a `namespaceSelector`, the webhook gates pods in `openshift-*` namespaces, which can block critical workloads.
6. **Using `r.Get()` when deletion timing matters**: The informer cache can return stale objects missing `DeletionTimestamp`. Use `r.APIReader.Get()` for direct API server reads when delete-order correctness is required. This was the root cause of [MULTIARCH-6269](https://issues.redhat.com/browse/MULTIARCH-6269) and [MULTIARCH-6239](https://issues.redhat.com/browse/MULTIARCH-6239).
7. **Not filtering attestation manifests during image inspection**: OCI image indexes may contain attestation manifests with `platform.architecture: "unknown"`. Always validate that platform entries represent real architectures before including them in nodeAffinity. Fixed in [MULTIARCH-5800](https://issues.redhat.com/browse/MULTIARCH-5800).
8. **ServicePortsMatch positional comparison**: When adding or reordering Service ports in operand resource builders, be aware that the port comparison in `resourcemerge` uses positional (index-based) matching rather than name-keyed lookup ([MULTIARCH-6309](https://issues.redhat.com/browse/MULTIARCH-6309)). This can cause unnecessary resource updates.

## Debugging Tips

> **Source**: Patterns from Jira bug reports and #forum-ocp-testplatform Slack threads.

### Pods stuck in SchedulingGated state
If pods remain gated, check:
1. MTO controller pod is running: `oc get pods -n openshift-multiarch-tuning-operator`
2. Controller logs for image inspection errors: `oc logs -n openshift-multiarch-tuning-operator deploy/multiarch-tuning-operator-controller`
3. Manual gate removal (emergency workaround): `kubectl patch pod <name> --type=json -p '[{"op":"remove","path":"/spec/schedulingGates/0"}]'`

### CPPC deletion stuck with finalizer
If a ClusterPodPlacementConfig has `DeletionTimestamp` but is not being cleaned up:
1. Check controller logs for reconcile errors
2. This may indicate the informer cache race ([MULTIARCH-6269](https://issues.redhat.com/browse/MULTIARCH-6269)) — fixed in v1.3.4
3. Manual finalizer removal (last resort): `kubectl patch cppc cluster --type=json -p '[{"op":"remove","path":"/metadata/finalizers"}]'`

### Image inspection returns "unknown" architecture
If pods get nodeAffinity for architecture "unknown":
1. Inspect the image manifest: `skopeo inspect --raw docker://<image>`
2. Look for attestation manifests with `platform.architecture: "unknown"` — fixed in v1.3 ([MULTIARCH-5800](https://issues.redhat.com/browse/MULTIARCH-5800))
3. Ensure you're running MTO v1.3+ which filters out attestation manifests

### E2E test "Should cleanup all finalizers" is flaky
This test creates and immediately deletes a CPPC, which races with the controller's finalizer injection. The root cause is tracked in [MULTIARCH-6240](https://issues.redhat.com/browse/MULTIARCH-6240) (open — switch to mutating webhook for finalizer injection).

## Lint and Quality

```bash
make lint       # golangci-lint
make gosec      # SAST security scanning
make vet        # go vet
make goimports  # goimports formatting check
make fmt        # gofmt formatting
```

All checks run automatically as part of `make test`.

## Active Development Areas

> **Source**: Jira project MULTIARCH, 2026-09-25.

| Area | Jira | Status |
|------|------|--------|
| Network policies for MTO | [MULTIARCH-5569](https://issues.redhat.com/browse/MULTIARCH-5569) | In Progress |
| Per-branch catalog build in Prow | [MULTIARCH-5537](https://issues.redhat.com/browse/MULTIARCH-5537) | In Progress |
| Code coverage investigation | [MULTIARCH-6003](https://issues.redhat.com/browse/MULTIARCH-6003) | In Progress |
| Migrate deprecated events API | [MULTIARCH-6087](https://issues.redhat.com/browse/MULTIARCH-6087) | To Do |
| Migrate to UBI10 | [MULTIARCH-6199](https://issues.redhat.com/browse/MULTIARCH-6199) | To Do |
| Finalizer via mutating webhook | [MULTIARCH-6240](https://issues.redhat.com/browse/MULTIARCH-6240) | To Do |
| PPC plugins field optional | [MULTIARCH-6340](https://issues.redhat.com/browse/MULTIARCH-6340) | To Do |
| Image volumes handling | [MULTIARCH-4984](https://issues.redhat.com/browse/MULTIARCH-4984) | To Do |
| Auto-create default CPPC with PPC | [MULTIARCH-5322](https://issues.redhat.com/browse/MULTIARCH-5322) | To Do |

## SME Review Recommended

- CI/CD pipeline configuration details for downstream OpenShift releases (Konflux-specific)
- Multi-architecture build matrix and `qemu-user-static` setup for `docker-buildx`
