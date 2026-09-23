# Development Guide

## Prerequisites

- **Go** 1.26.7 (`go.mod` specifies `go 1.26.7`, no `toolchain` directive)
- **CGO dependencies**: `gpgme-devel` (Fedora/RHEL/CS9) or `libgpgme-dev` (Debian/Ubuntu) — required for container image inspection
- **Container runtime**: Podman (preferred) or Docker
- **operator-sdk**: For bundle generation
- **Default branch**: `main`

## Build

```bash
# Local build (requires CGO + gpgme)
make build

# Single-arch container image
make docker-build IMG=<registry>/multiarch-tuning-operator:tag

# Multi-arch container image (requires qemu-user-static)
make docker-buildx IMG=<registry>/multiarch-tuning-operator:tag

# Prevent buildx instance deletion between builds
touch .persistent-buildx
```

By default, builds run inside a container using `BUILD_IMAGE`. To build locally:

```bash
NO_DOCKER=1 make build
```

## Environment Configuration

Create a `.env` file at the repo root (see `dotenv.example`):

| Variable | Purpose |
|----------|---------|
| `NO_DOCKER=1` | Run builds and tests locally instead of in container |
| `FORCE_DOCKER=1` | Force Docker instead of Podman |
| `BUILD_IMAGE` | Override builder image |
| `RUNTIME_IMAGE` | Override runtime base image |

## Common Tasks

### After API type changes

```bash
make generate    # Regenerate zz_generated.deepcopy.go
make manifests   # Regenerate CRDs, RBAC, webhook config
```

### After dependency changes

```bash
make vendor      # Update vendored dependencies (GOFLAGS=-mod=vendor)
make verify-diff # Verify no uncommitted changes
```

### Run all quality checks + tests

```bash
make test  # Runs: manifests, generate, fmt, vet, goimports, gosec, lint, unit tests
```

### Deploy to cluster

```bash
make install                                            # Install CRDs
make deploy IMG=<registry>/multiarch-tuning-operator:tag # Deploy operator
make undeploy                                           # Remove operator
make uninstall                                          # Remove CRDs
```

### Enable pod placement operand

```bash
kubectl create -f - <<EOF
apiVersion: multiarch.openshift.io/v1beta1
kind: ClusterPodPlacementConfig
metadata:
  name: cluster
spec:
  logVerbosityLevel: Normal
  namespaceSelector:
    matchExpressions:
      - key: multiarch.openshift.io/exclude-pod-placement
        operator: DoesNotExist
EOF
```

### OLM Bundle

```bash
make bundle VERSION=<version>                # Generate bundle manifests
make bundle-verify                           # Verify bundle is deterministic
make bundle-build BUNDLE_IMG=<registry>/multiarch-tuning-operator-bundle:<version>
make bundle-push BUNDLE_IMG=<registry>/multiarch-tuning-operator-bundle:<version>
make catalog-build CATALOG_IMG=<registry>/multiarch-tuning-operator-catalog:<version>
make catalog-push CATALOG_IMG=<registry>/multiarch-tuning-operator-catalog:<version>
```

### Version bumping

```bash
hack/bump-version.sh <new-version>
```

## Key Makefile Targets

| Target | Description |
|--------|-------------|
| `build` | Compile the operator binary |
| `test` | Full check suite (fmt, vet, lint, gosec, goimports, unit tests) |
| `unit` | Unit tests only |
| `e2e` | E2E tests (requires deployed operator) |
| `lint` | golangci-lint (config: `.golangci.yaml`) |
| `gosec` | SAST security scanning |
| `fmt` | `gofmt` formatting check |
| `vet` | `go vet` static analysis |
| `goimports` | Import ordering check |
| `manifests` | Regenerate CRDs + RBAC from kubebuilder markers |
| `generate` | Regenerate deepcopy implementations |
| `vendor` | Update vendor directory |
| `verify-diff` | Ensure working tree has no uncommitted changes |
| `verify-snapshots` | Check test snapshots are up to date |
| `install` / `uninstall` | Install/remove CRDs from cluster |
| `deploy` / `undeploy` | Deploy/remove operator from cluster |

## Common Mistakes

1. **Building without gpgme**: The `containers/image` library requires CGO and gpgme. Without `gpgme-devel`, `make build` fails with missing header errors.

2. **Setting multiple `--enable-*` flags**: The binary validates exactly one mode flag is set and exits with an error otherwise (`cmd/main.go`).

3. **Forgetting `make manifests` after API changes**: CRD YAML and RBAC are generated from kubebuilder markers. Hand-editing `config/crd/bases/` files will be overwritten.

4. **Missing `namespaceSelector`**: Without a properly configured `namespaceSelector` on ClusterPodPlacementConfig, the webhook will gate pods in `openshift-*` namespaces, potentially blocking critical workloads.

5. **Not running `make vendor`**: The project uses `GOFLAGS=-mod=vendor`. After changing `go.mod`, always run `make vendor` to sync the vendor directory.

6. **Adding RBAC markers to operand code**: Operand RBAC is defined programmatically in `internal/controller/operator/podplacement_objects.go`. Kubebuilder markers in operand code would incorrectly add permissions to the operator's ClusterRole.

## Release Process

See [docs/ocp-release.md](../docs/ocp-release.md) for the full OCP release process including:
- ProdSec SAST considerations
- Golang and Kubernetes API version pinning
- Branch creation for new development streams
- FBC (File-Based Catalog) fragment management

## Monitoring

See [docs/metrics.md](../docs/metrics.md) for the full metrics reference. Key metrics:
- `mto_ppo_ctrl_time_to_process_gated_pod_seconds` — end-to-end pod processing time
- `mto_ppo_ctrl_failed_image_inspection_total` — image inspection failures
- `mto_ppo_wh_pods_gated_total` — pods gated by the webhook

Alert runbooks are in [docs/alerts/](../docs/alerts/).

For generic OpenShift development conventions, see [openshift/enhancements](https://github.com/openshift/enhancements) (`dev-guide/`, `CONVENTIONS.md`).
