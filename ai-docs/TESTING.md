# Testing Guide

## Test Framework

All tests use **Ginkgo/Gomega** (`github.com/onsi/ginkgo/v2`, `github.com/onsi/gomega`). Unit tests use **envtest** for a local API server.

## Test Suites

| Suite | Location | Command | Requirements |
|-------|----------|---------|--------------|
| Unit tests | `*_test.go` alongside source | `make unit` | None (envtest provides API server) |
| E2E tests | `pkg/e2e/` | `make e2e` | Deployed operator + `KUBECONFIG` |
| Snapshot tests | Throughout | `make verify-snapshots` | None |

## Running Tests

### Unit tests

```bash
# All unit tests (containerized by default)
make unit

# Run locally instead of in container
NO_DOCKER=1 make unit

# Run a specific test
GINKGO_ARGS="-v --focus='your test pattern'" make unit

# Full check suite (formatting, linting, security, + unit tests)
make test
```

Test output goes to `ARTIFACT_DIR` (default: `./_output`). Coverage report: `test-unit-coverage.out`.

### E2E tests

```bash
# Requires a cluster with the operator deployed
KUBECONFIG=/path/to/kubeconfig NAMESPACE=openshift-multiarch-tuning-operator make e2e
```

E2E tests are in `pkg/e2e/` with separate suites:
- `pkg/e2e/operator/` — operator lifecycle tests
- `pkg/e2e/podplacement/` — pod placement behavior tests
- `pkg/e2e/podplacementconfig/` — PodPlacementConfig behavior tests

### Snapshot verification

```bash
make verify-snapshots  # Checks test snapshots are up to date
```

Uses `hack/check-snapshots.sh` to ensure expected outputs match.

## Test Helpers

### Fluent builders (`pkg/testing/builder/`)

The project provides fluent builder functions for constructing Kubernetes objects in tests:

```go
// Example: build a pod with scheduling gate (pkg/testing/builder/pod.go)
pod := builder.NewPod().
    WithName("test-pod").
    WithNamespace("test-ns").
    WithSchedulingGates(utils.SchedulingGateName).
    WithContainersImages("registry.io/test:latest").
    Build()
```

Builders exist for Pods, Deployments, Namespaces, and other common objects.

### Framework utilities (`pkg/testing/framework/`)

Provides test lifecycle helpers:
- Cluster connection and client setup
- Namespace creation/cleanup
- Resource assertion helpers
- Wait/retry utilities for eventually-consistent checks

## Test Patterns

### Controller unit tests

Controller tests use envtest (`sigs.k8s.io/controller-runtime/pkg/envtest`) to run a local API server:

```go
var _ = BeforeSuite(func() {
    testEnv = &envtest.Environment{
        CRDDirectoryPaths: []string{filepath.Join("..", "..", "..", "config", "crd", "bases")},
    }
    cfg, err = testEnv.Start()
    // ...
})
```

### Webhook tests

Webhook tests create pods with various configurations and verify the webhook correctly adds or skips the scheduling gate based on namespace selectors, exclusion rules, and pod characteristics.

### Image inspection mocks

Tests mock the image inspection cache to control architecture responses without requiring registry access:

```go
// Tests set imageInspectionCache to a mock that returns controlled results
imageInspectionCache = &mockCache{
    inspectResult: sets.New("amd64", "arm64"),
}
```

## CI Integration

The CI pipeline runs via `hack/ci-test.sh`, which executes the full test suite including:
- `make test` (all checks + unit tests)
- E2E tests against a deployed cluster (when available)

Tests run in containers by default using `BUILD_IMAGE`. Set `NO_DOCKER=1` to run locally.

For generic OpenShift testing conventions, see [openshift/enhancements](https://github.com/openshift/enhancements) (`dev-guide/`).
