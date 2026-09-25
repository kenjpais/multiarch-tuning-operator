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

## Lint and Quality

```bash
make lint       # golangci-lint
make gosec      # SAST security scanning
make vet        # go vet
make goimports  # goimports formatting check
make fmt        # gofmt formatting
```

All checks run automatically as part of `make test`.

## SME Review Recommended

- CI/CD pipeline configuration details for downstream OpenShift releases (Konflux-specific)
- Multi-architecture build matrix and `qemu-user-static` setup for `docker-buildx`
