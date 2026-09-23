# Architecture

## Repository Layout

```text
api/
├── common/              # Shared constants (verbosity levels) and plugin types
│   └── plugins/         # Plugin definitions (NodeAffinityScoring, CelArchitecturePlacement, ExecFormatErrorMonitor)
├── v1alpha1/            # Alpha API — conversion to/from v1beta1 hub
└── v1beta1/             # Beta API — storage version (ClusterPodPlacementConfig, PodPlacementConfig, ENoExecEvent)

cmd/
├── main.go              # Entrypoint — four mutually exclusive modes via --enable-* flags
└── enoexec-daemon/      # Separate binary for eBPF-based exec format error monitoring on nodes

internal/controller/
├── operator/            # Operator mode: CPPC lifecycle, operand deployment, resource building
│   ├── podplacement_objects.go   # Build functions for webhook, controller deployments, RBAC
│   ├── enoexecevent_objects.go   # Build functions for enoexec DaemonSet, handler, alerts
│   └── objects.go                # Shared build helpers (Service, Deployment, ClusterRole, ServiceMonitor)
├── podplacement/        # Operand mode: pod reconciler, webhook, image inspection, CEL evaluator
│   ├── pod_model.go              # Core pod processing logic (700 lines) — shouldIgnorePod, affinity, gates
│   ├── cel_evaluator.go          # CEL expression compilation and caching (LRU, 1024 entries)
│   └── metrics/                  # Prometheus metrics for controller and webhook
├── podplacementconfig/  # PodPlacementConfig controller (namespaced, stub — TODO in reconciler)
└── enoexecevent/
    ├── daemon/          # eBPF tracepoint monitoring daemon (runs as DaemonSet on nodes)
    └── handler/         # ENoExecEvent CR handler (creates events on affected pods)

pkg/
├── image/               # Container image inspection — manifest fetching, auth, caching
├── informers/           # ClusterPodPlacementConfig singleton informer (runtime config access)
├── models/              # Shared pod model (gate management, label helpers)
├── testing/             # Test utilities — fluent builders (builder/), framework helpers (framework/)
└── utils/               # Constants, resource apply/delete, runtime helpers
    ├── const.go          # ALL label keys, annotation keys, scheduling gate name, component names
    └── resource.go       # ApplyResource/ApplyResources — library-go resourceapply dispatcher

config/                  # Kustomize overlays (CRDs, RBAC, webhook, default, manager)
bundle/                  # OLM bundle (CSV, CRDs, RBAC manifests)
hack/                    # Build/test scripts (CI, version bumping, snapshot checks)
deploy/                  # Deployment manifests (OCP-specific)
test/manifests/          # Test fixture manifests
docs/                    # Operational docs (metrics, OCP release process, alerts, enhancements)
```

## Key Domain Concepts

The Multiarch Tuning Operator (MTO) solves a single problem: in clusters with nodes of different CPU architectures (amd64, arm64, ppc64le, s390x), pods must only be scheduled on nodes whose architecture matches the container images they run.

**Primary workflow — Pod Placement:**
1. User creates a `ClusterPodPlacementConfig` singleton CR (name must be `"cluster"`)
2. Operator controller deploys the pod placement operand (controller + webhook deployments)
3. Webhook intercepts pod creation and adds the `multiarch.openshift.io/scheduling-gate` scheduling gate
4. Pod reconciler watches gated pods, inspects container images to determine supported architectures
5. Reconciler sets `nodeAffinity` for `kubernetes.io/arch` matching supported architectures
6. Reconciler removes the scheduling gate — pod enters the regular scheduling cycle

**Secondary workflow — ENoExec Monitoring:**
An eBPF-based daemon on each node detects exec format errors (wrong-architecture binaries) and creates `ENoExecEvent` CRs. A handler controller processes these and publishes Kubernetes events on affected pods.

**Three CRDs:**
- `ClusterPodPlacementConfig` — cluster-scoped singleton controlling the operand lifecycle and global config
- `PodPlacementConfig` — namespace-scoped, label-selected pod matching with CEL rules and priority-based ordering
- `ENoExecEvent` — cluster-scoped, created by the eBPF daemon when exec format errors are detected

## Component/Controller Details

All controllers use **controller-runtime**. The binary runs in one of four mutually exclusive modes:

| Mode | Flag | Leader Election ID | Controller |
|------|------|--------------------|------------|
| Operator | `--enable-operator` | `operator-208d7abd.multiarch.openshift.io` | `ClusterPodPlacementConfigReconciler` |
| PPC Controllers | `--enable-ppc-controllers` | `ppc-controllers-208d7abd.multiarch.openshift.io` | `PodReconciler` |
| PPC Webhook | `--enable-ppc-webhook` | (none — webhook only) | `PodSchedulingGateMutatingWebHook` |
| ENoExec Controllers | `--enable-enoexec-event-controllers` | `enoexecevent-controllers-208d7abd.multiarch.openshift.io` | `ENoExecEventReconciler` |

**Startup sequence** (`cmd/main.go`):
1. Parse flags via `bindFlags()` — validates exactly one `--enable-*` flag is set
2. Register schemes (v1alpha1, v1beta1, monitoring)
3. Build manager with mode-specific cache, webhook, and metrics configuration
4. Register controllers, runnables, and health checks for the selected mode
5. Start manager with leader election (except webhook mode)

**Concurrency**: PodReconciler uses `MaxConcurrentReconciles = NumCPU * 4` (`internal/controller/podplacement/pod_reconciler.go:344`) because image inspection is I/O-bound. The webhook uses an `ants` multi-pool (`ants.NewMultiPool(16, 16, ants.LeastTasks)` — 16 sub-pools of 16 goroutines, up to 256 total) for event publishing (`cmd/main.go:264`).

## Resource Management

The operator controller uses **library-go `resourceapply`** for operand resource management, not raw Create/Update:

| Resource Type | Apply Method | Code Reference |
|---------------|-------------|----------------|
| Deployment | `applyDeployment` (custom wrapper using library-go generation tracking) | `pkg/utils/resource.go:72-73` |
| DaemonSet | `applyDaemonSet` (custom wrapper) | `pkg/utils/resource.go:74-75` |
| Service | `applyService` (custom wrapper) | `pkg/utils/resource.go:76-77` |
| MutatingWebhookConfiguration | `resourceapply.ApplyMutatingWebhookConfigurationImproved` | `pkg/utils/resource.go:78-80` |
| Role, RoleBinding | `resourceapply.ApplyRole`, `resourceapply.ApplyRoleBinding` | `pkg/utils/resource.go:81-83` |
| ClusterRole, ClusterRoleBinding | `resourceapply.ApplyClusterRole`, `resourceapply.ApplyClusterRoleBinding` | `pkg/utils/resource.go:86-89` |
| ServiceAccount | `resourceapply.ApplyServiceAccount` | `pkg/utils/resource.go:84-86` |
| ServiceMonitor, PrometheusRule | `resourceapply.ApplyServiceMonitor/ApplyPrometheusRule` (via unstructured) | `pkg/utils/resource.go:91-106` |

**Pod reconciler** and **ENoExec handler** use direct `r.Update()` calls (controller-runtime client) to patch pod/CR status and remove scheduling gates.

**Operator uses finalizers** for ordered teardown:
- `finalizers.multiarch.openshift.io/pod-placement` — ensures pods are ungated before operand removal
- `finalizers.multiarch.openshift.io/no-pod-placement-config` — tracks PodPlacementConfig dependency
- `finalizers.multiarch.openshift.io/enoexec-events` — enoexec cleanup

All operand resources have `ownerReferences` set via `ctrl.SetControllerReference`, enabling garbage collection when the CPPC CR is deleted.

## Error Classification

| Error Type | Effect | Code Reference |
|------------|--------|----------------|
| `requeueAfterError` | Requeue after configurable duration (e.g., 5s) | `internal/controller/operator/clusterpodplacementconfig_controller.go:76-79` |
| Image inspection failure | Retried up to `MaxRetryCount` (5), then gate removed with error labels | `internal/controller/podplacement/pod_model.go:47` |
| Status update failure | Returns error to requeue | operator controller pattern throughout |
| Webhook `failurePolicy: Ignore` | Pod proceeds without gating if webhook is unavailable | `internal/controller/operator/podplacement_objects.go:43` |

## Generated Code Inventory

| File/Pattern | Generator | Regenerate Command |
|-------------|-----------|-------------------|
| `api/*/zz_generated.deepcopy.go` | controller-gen | `make generate` |
| `config/crd/bases/*.yaml` | controller-gen | `make manifests` |
| `config/rbac/role.yaml` | controller-gen (kubebuilder markers) | `make manifests` |
| `bundle/manifests/*.yaml` | operator-sdk | `make bundle VERSION=<ver>` |

**NEVER hand-edit** any `zz_generated*` file or CRD YAML under `config/crd/bases/`.

## API Behavioral Contracts

**Singleton constraint**: `ClusterPodPlacementConfig` must be named `"cluster"`. The validation webhook rejects any other name (`api/v1beta1/clusterpodplacementconfig_webhook.go`).

**API versions**: v1alpha1 has a conversion webhook to v1beta1 (hub). v1beta1 is the storage version.

**Namespace exclusions** (hardcoded in `shouldIgnorePod`, `internal/controller/podplacement/pod_model.go:471`):
- Operator's own namespace (`utils.Namespace()`)
- `kube-*` prefixed namespaces
- Pods with `spec.nodeName` already set
- Pods with control-plane nodeSelector (`node-role.kubernetes.io/master` or `node-role.kubernetes.io/control-plane`)
- DaemonSet-owned pods

**DO NOT** assume `openshift-*` or `hypershift-*` namespaces are excluded — they are not hardcoded. They require explicit `namespaceSelector` configuration on the CR.

**Fallback architecture**: When image inspection fails and `spec.fallbackArchitecture` is set (one of `amd64`, `arm64`, `ppc64le`, `s390x`), the pod is scheduled to nodes of that architecture instead of failing.

**PodPlacementConfig priority**: Namespaced PodPlacementConfig resources are applied in priority order (0-255). Higher priority configs take precedence for preferred affinity weights.

**Plugin system** (`api/common/plugins/`):
- `NodeAffinityScoringPluginName` — adds preferred (soft) nodeAffinity based on cluster node distribution
- `CelArchitecturePlacementPluginName` — applies CEL-based architecture rules
- `ExecFormatErrorMonitorPluginName` — enables the eBPF-based enoexec monitoring subsystem

**Scheduling gate label tracking**: The operator labels pods with `multiarch.openshift.io/scheduling-gate: gated` when gated and `removed` when the gate is lifted. Additional labels indicate `single-arch`, `multi-arch`, `no-supported-arch`, and `fallback-arch` for debugging.

## OpenShift Integration Points

| Integration | Mechanism | Code Reference |
|------------|-----------|----------------|
| OLM lifecycle | CSV in `bundle/manifests/`, `spec.replaces` chain | `bundle/manifests/multiarch-tuning-operator.clusterserviceversion.yaml` |
| CA bundle injection | Annotation `service.beta.openshift.io/inject-cabundle: true` on webhook config | `internal/controller/operator/podplacement_objects.go:29` |
| Monitoring | ServiceMonitor + PrometheusRule (conditional on CRD availability) | `internal/controller/operator/clusterpodplacementconfig_controller.go:824-834` |
| SCC | Detects `hostmount-anyuid` or `hostmount-anyuid-v2` for controller SecurityContext | `internal/controller/operator/clusterpodplacementconfig_controller.go:894` |
| Global pull secret | Syncs OpenShift's global pull secret for image inspection | `internal/controller/podplacement/global_pull_secret.go` |

## Plugins System

Plugins are defined in `api/common/plugins/` and configured via the `plugins` field on ClusterPodPlacementConfig (global) and PodPlacementConfig (namespaced).

**NodeAffinityScoring** (`plugins/nodeaffinityscoring_plugin.go`): Adds preferred nodeAffinity terms with architecture-specific weights. Weights can be configured per-architecture (1-100).

**CelArchitecturePlacement** (`api/common/plugins/celarchitectureplacement_plugin.go`): Uses CEL (Common Expression Language) expressions evaluated at scheduling time to determine architecture constraints. Compiled programs are cached in an LRU cache (1024 entries) in `internal/controller/podplacement/cel_evaluator.go`.

**ExecFormatErrorMonitor**: Enables the eBPF-based daemon that detects exec format errors and the handler that processes them.

## Design References

**Scheduling gate pattern**: The operator leverages KEP-3521 (Pod Scheduling Readiness) to temporarily prevent pod scheduling while image inspection determines architecture compatibility. This is a non-blocking pattern — the webhook's `failurePolicy: Ignore` ensures pods are not permanently blocked if the operator is down. The gate is removed even on inspection failure (with appropriate labels for debugging).

**Library-go resourceapply for operand management**: The operator controller uses library-go's `resourceapply` functions rather than raw Create/Update. This provides generation-based change detection — resources are only applied when their specification changes, reducing unnecessary API calls. See `pkg/utils/resource.go` for the type-switch dispatcher.

**Ordered deletion with finalizers**: The operator uses a finalizer-based ordered deletion sequence to ensure pods are ungated before operand removal. This prevents pods from being permanently blocked if the operator is deleted while pods are still gated.

## Platform Documentation

For generic OpenShift development patterns, testing conventions, and coding standards, see the [openshift/enhancements](https://github.com/openshift/enhancements) repository:
- `dev-guide/` — development conventions
- `guidelines/` — enhancement process
- `CONVENTIONS.md` — coding standards

## Image Inspection

The image inspector (`pkg/image/`) is the central component for determining container image architecture support:

- **Registry interaction**: Fetches image manifests via the `containers/image` library (requires CGO + gpgme)
- **Multi-arch support**: Handles OCI image indexes and Docker v2.2 manifest lists to extract per-platform entries
- **Authentication**: Uses pod-level pull secrets + the global pull secret (synced from `openshift-config/pull-secret`)
- **Caching**: `image.FacadeSingleton()` provides a process-wide cache to avoid redundant registry queries
- **Metrics**: Tracks inspection time (`mto_ppo_ctrl_time_to_inspect_image_seconds`) and failures (`mto_ppo_ctrl_failed_image_inspection_total`)

Architecture set computation (`internal/controller/podplacement/pod_model.go`):
1. For each container image, inspect the manifest to get supported architectures
2. Intersect all container architecture sets — the pod runs only on architectures all containers support
3. If intersection is empty, label the pod `multiarch.openshift.io/no-supported-arch` and remove the gate
4. If non-empty, set required nodeAffinity matching the intersection and remove the gate

## SME Review Recommended

- Exact behavior of `PodPlacementConfig` reconciler (currently a stub with `TODO` — implementation may be in-flight)
- CEL plugin expression format and available variables/functions
- ENoExec eBPF tracepoint specifics and kernel compatibility requirements
- Downstream Konflux/Tekton pipeline configuration
