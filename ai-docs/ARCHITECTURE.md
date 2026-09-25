# Architecture

## Repository Layout

```text
cmd/
├── main.go                          # Single binary entrypoint; flag-driven mode selection
└── enoexec-daemon/main.go           # Separate eBPF daemon binary (deployed as DaemonSet)

api/
├── common/                          # Shared constants (LogVerbosity), labels, plugins interface
│   └── plugins/                     # Plugin types: NodeAffinityScoring, CEL ArchitectureRule
├── v1alpha1/                        # Alpha API version with conversion to v1beta1
└── v1beta1/                         # Storage version — ClusterPodPlacementConfig, PodPlacementConfig types

internal/controller/
├── operator/                        # Operator mode: reconciles CPPC, deploys operand resources
│   ├── objects.go                   # Resource builders (Deployment, Service, RBAC, webhook)
│   ├── podplacement_objects.go      # Pod placement operand resource definitions
│   └── enoexecevent_objects.go      # ENoExec operand resource definitions
├── podplacement/                    # Operand: pod reconciler, webhook, CEL evaluator
│   ├── pod_reconciler.go            # Watches gated Pending pods, inspects images, sets nodeAffinity
│   ├── scheduling_gate_mutating_webhook.go  # Adds scheduling gate to new pods
│   ├── pod_model.go                 # Core pod processing logic and image architecture detection
│   ├── cel_evaluator.go             # CEL expression compilation/caching/evaluation
│   ├── cel_integration.go           # Integrates PodPlacementConfig CEL rules into reconciler
│   ├── architecture_application.go  # In-place nodeAffinity mutation for architecture constraints
│   ├── architecture_removal.go      # Removes architecture constraints from nodeSelector/affinity
│   ├── global_pull_secret.go        # GlobalPullSecretSyncer runnable
│   └── metrics/                     # Prometheus metric definitions
├── podplacementconfig/              # PodPlacementConfig validation webhook
└── enoexecevent/
    ├── daemon/                      # eBPF-based exec format error monitor (separate binary)
    └── handler/                     # ENoExecEvent CR reconciler

pkg/
├── image/                           # Container image inspection (containers/image library, CGO)
│   ├── inspector.go                 # Registry manifest retrieval, architecture detection, caching
│   └── metrics/                     # Image inspection metrics
├── informers/clusterpodplacementconfig/  # CPPC singleton informer (runtime config access)
├── utils/                           # Constants, resourceapply dispatcher, namespace/image resolution
│   ├── const.go                     # Labels, finalizers, scheduling gate name, arch constants
│   ├── resource.go                  # ApplyResource — library-go resourceapply type switch
│   └── runtime.go                   # Namespace(), Image() from env vars
├── testing/                         # Test helpers: builders, framework, fake registry
│   ├── builder/                     # Fluent builders for K8s objects in tests
│   ├── framework/                   # Shared test utilities (envtest setup, cluster version)
│   └── image/fake/                  # Fake container registry for unit tests
├── models/                          # Shared data models
└── e2e/                             # E2E test suites (operator, podplacement, podplacementconfig)

config/                              # Kustomize overlays (default, crd, rbac, webhook, certmanager)
bundle/                              # OLM bundle (CSV, CRDs, RBAC manifests)
deploy/                              # Deployment templates
hack/                                # Build scripts, CI helpers
docs/                                # Repo docs: metrics, alerts, enhancements, release process
```

## Key Domain Concepts

The Multiarch Tuning Operator solves the problem of scheduling pods onto nodes with compatible CPU architectures in heterogeneous clusters. The core abstraction is the **scheduling gate**: a Kubernetes mechanism (KEP-3521) that prevents a pod from being scheduled until it is explicitly ungated.

**Primary workflow** — Pod Placement:
1. User creates a `ClusterPodPlacementConfig` singleton CR (name must be `"cluster"`)
2. Operator controller deploys operand components (controller + webhook Deployments, RBAC, MutatingWebhookConfiguration)
3. Webhook intercepts new pod creation and adds `multiarch.openshift.io/scheduling-gate`
4. Pod reconciler watches Pending pods with the gate, inspects container images to determine supported architectures
5. Reconciler adds `kubernetes.io/arch` nodeAffinity (required) and optionally preferred scheduling terms
6. Reconciler removes the scheduling gate, allowing the scheduler to place the pod

**PodPlacementConfig** (namespaced) extends this with CEL-based architecture rules that override image inspection results for matching pods, enabling fine-grained per-namespace control.

**ENoExecEvent** monitors exec format errors via eBPF on nodes. A DaemonSet captures ENOEXEC syscall failures and creates ENoExecEvent CRs. The handler controller processes them and publishes events on affected pods.

## Component/Controller Details

| Component | Framework | Mode Flag | Leader Election ID |
|-----------|-----------|-----------|-------------------|
| ClusterPodPlacementConfigReconciler | controller-runtime + library-go events | `--enable-operator` | `operator-208d7abd.multiarch.openshift.io` |
| PodReconciler | controller-runtime | `--enable-ppc-controllers` | `ppc-controllers-208d7abd.multiarch.openshift.io` |
| PodSchedulingGateMutatingWebHook | controller-runtime webhook | `--enable-ppc-webhook` | (shares webhook mode) |
| ENoExecEventHandler | controller-runtime | `--enable-enoexec-event-controllers` | `enoexecevent-controllers-208d7abd.multiarch.openshift.io` |
| GlobalPullSecretSyncer | manager runnable | (with ppc-controllers) | — |
| CPPCSyncer | manager runnable | `--enable-cppc-informer` | — |

**Startup sequence** (`cmd/main.go`):
1. `bindFlags()` — parse CLI flags, initialize atomic log level
2. `validateFlags()` — ensure exactly one mode flag is set
3. Build cache options (Pending pod field selector for ppc-controllers mode)
4. Create controller-runtime Manager with scheme, metrics, webhook server
5. Conditionally call `RunOperator`, `RunClusterPodPlacementConfigOperandControllers`, `RunClusterPodPlacementConfigOperandWebHook`, or `RunENoExecEventControllers`
6. `mgr.Start()` — blocks until signal

## Resource Management

The operator controller uses a **dual apply strategy**:

| Resource Type | Apply Method | Code Reference |
|---------------|-------------|----------------|
| ClusterRole, ClusterRoleBinding | `resourceapply.ApplyClusterRole/Binding` | `pkg/utils/resource.go:88-90` |
| Role, RoleBinding | `resourceapply.ApplyRole/Binding` | `pkg/utils/resource.go:82-84` |
| ServiceAccount | `resourceapply.ApplyServiceAccount` | `pkg/utils/resource.go:86` |
| MutatingWebhookConfiguration | `resourceapply.ApplyMutatingWebhookConfigurationImproved` | `pkg/utils/resource.go:79` |
| ServiceMonitor, PrometheusRule | `resourceapply` (dynamic unstructured) | `pkg/utils/resource.go:96+` |
| Deployment, Service | Custom apply via `resourcemerge` + Get/Create/Update | `pkg/utils/resource.go` (`applyDeployment`, `applyService`) |
| ClusterPodPlacementConfig (status, finalizers) | `controller-runtime` `r.Update()` | `internal/controller/operator/clusterpodplacementconfig_controller.go:201+` |

**DO NOT** mix these: RBAC/webhook resources go through `ApplyResource` (library-go), CR status and finalizers go through controller-runtime's `r.Update()`.

The pod reconciler uses controller-runtime `r.Update()` (full-object update, not status subresource) to apply pod affinity changes and remove scheduling gates (`internal/controller/podplacement/pod_reconciler.go:104`).

## Feature Gates

This operator does **not** define its own feature gates. It does not use `openshift/api` FeatureGate definitions or TechPreviewNoUpgrade gating. Feature enablement is entirely flag-driven at the binary level.

## Error Classification

| Error Type | Behavior | Code Reference |
|-----------|----------|----------------|
| `requeueAfterError` | Custom type for controlled requeue with specific duration | `internal/controller/operator/clusterpodplacementconfig_controller.go:76-84` |
| Image inspection failure | Retried up to max retries; if `fallbackArchitecture` is set, uses fallback | `internal/controller/podplacement/pod_reconciler.go` |
| CR not found (404) | Ignored (reconcile ends) | Standard controller-runtime pattern |
| Status update failure | Merged with primary errors via `mergeWithStatusErr` | `internal/controller/operator/clusterpodplacementconfig_controller.go:87-97` |

## OpenShift Integration Points

| Integration | How | Code Reference |
|-------------|-----|----------------|
| library-go resourceapply | RBAC, webhook, ServiceMonitor resource management | `pkg/utils/resource.go` |
| library-go events | Event recording on ClusterPodPlacementConfig changes | `cmd/main.go:228-238` |
| Serving certificates | `service.beta.openshift.io/serving-cert-secret-name` annotation on Services | `internal/controller/operator/objects.go` |
| Global pull secret | Syncs `openshift-config/pull-secret` for image inspection auth | `internal/controller/podplacement/global_pull_secret.go` |
| Image registry certs | Reads `image-registry-certificates` ConfigMap | `cmd/main.go:355` |
| SecurityContextConstraints | Pods annotated with `openshift.io/required-scc: restricted-v2` | `internal/controller/operator/objects.go:34` |
| PrometheusRule alerts | Alert rules for operand component health | `internal/controller/operator/objects.go` (via resource builders) |
| OLM | Bundle in `bundle/`, CSV in `bundle/manifests/` | Standard OLM lifecycle |

## Generated Code Inventory

| File Pattern | Generator | Regenerate |
|-------------|-----------|-----------|
| `api/*/zz_generated.deepcopy.go` | controller-gen | `make generate` |
| `config/crd/bases/*.yaml` | controller-gen | `make manifests` |
| `config/rbac/role.yaml` | controller-gen (RBAC markers) | `make manifests` |
| `bundle/` | operator-sdk | `make bundle VERSION=<ver>` |

**NEVER** hand-edit any `zz_generated*` file or `config/crd/bases/` YAML.

## API Behavioral Contracts

### ClusterPodPlacementConfig (cluster-scoped singleton)
- **Singleton**: Name must be `"cluster"` — enforced by convention via constant `common.SingletonResourceObjectName` (`api/common/const.go:3`); the validation webhook validates plugin config, not the resource name
- **API versions**: v1alpha1 (with conversion webhook) → v1beta1 (storage version)
- **Finalizers**: `finalizers.multiarch.openshift.io/pod-placement` (operand lifecycle), `finalizers.multiarch.openshift.io/no-pod-placement-config` (PPC object guard), `finalizers.multiarch.openshift.io/enoexec-events` (ENoExec cleanup)
- **Status conditions**: `Available`, `Progressing`, `Degraded`, `Deprovisioning`, `PodPlacementControllerNotRolledOut`, `PodPlacementWebhookNotRolledOut`, `MutatingWebhookConfigurationNotAvailable`
- **Deletion**: Ordered — removes scheduling gates from all gated pods before deleting operand Deployments
- **Spec fields**: `logVerbosity` (Normal/Debug/Trace/TraceAll), `namespaceSelector`, `plugins` (NodeAffinityScoring), `fallbackArchitecture` (amd64/arm64/ppc64le/s390x/empty)

### PodPlacementConfig (namespace-scoped)
- **Purpose**: Fine-grained, per-namespace architecture rules using CEL expressions
- **Priority**: `priority` field (0-255, default 0) — higher priority configs override lower ones
- **Spec fields**: `labelSelector` (pod matching), `plugins` (CEL ArchitectureRule), `priority`
- **CEL evaluation**: Compiled programs cached in LRU (1024 entries), evaluated per-pod

### Scheduling Gate Contract
- Gate name: `multiarch.openshift.io/scheduling-gate`
- Labels set on processed pods: `multiarch.openshift.io/node-affinity` (`set` or `overriden`), `multiarch.openshift.io/scheduling-gate` (`gated`/`removed`), architecture labels (`single-arch`/`multi-arch`/`no-supported-arch`/`fallback-arch`)

### Namespace Exclusions (hardcoded)
- Operator's own namespace (`NAMESPACE` env var via `utils.Namespace()`)
- `kube-*` prefixed namespaces
- Pods already having `spec.nodeName`, control-plane nodeSelector, or DaemonSet ownership

**WARNING**: `openshift-*` and `hypershift-*` namespaces are NOT hardcoded exclusions — they require explicit `namespaceSelector` configuration on the CR.

### Pod Reconciler Concurrency
- `MaxConcurrentReconciles = runtime.NumCPU() * 4` — optimized for I/O-bound image inspection
- Cache field selector: only watches `status.phase=Pending` pods
- Watches `PodPlacementConfig` to re-queue gated pods when PPC is created/updated/deleted

### Image Inspection
- Uses `containers/image` library (requires CGO + gpgme)
- Supports OCI and Docker v2 manifest formats
- Authenticates via synced global pull secret and registry certificates
- Results cached to reduce registry queries
- Supports multi-arch manifest lists — extracts supported architectures from all platforms

## Design References

### Scheduling Gate Approach (KEP-3521)
**Decision**: Use Kubernetes scheduling gates rather than node selectors or admission rejection.
**Rationale**: Gates allow the pod to exist in the API but prevent scheduling until architecture affinity is determined. This avoids race conditions, supports image inspection workflows, and integrates with the Kubernetes scheduler natively.
**Consequences**: Requires K8s 1.27+ (scheduling gates GA). All gated pods must be ungated before operator removal.

### Dual Apply Strategy
**Decision**: Use library-go `resourceapply` for RBAC/infrastructure resources and controller-runtime client for CR status updates.
**Rationale**: `resourceapply` provides idempotent, cache-backed apply semantics with proper conflict resolution for shared infrastructure resources. CR status requires standard controller-runtime patterns for optimistic concurrency.
**Consequences**: Two distinct update paths — never cross them. Operator controller tests must mock both client types.

### CEL-Based Architecture Rules
**Decision**: Use CEL expressions (via `cel-go`) for PodPlacementConfig architecture rules instead of static field matching.
**Rationale**: CEL provides expressive, Kubernetes-native evaluation that can match on any pod field. LRU caching (1024 compiled programs) amortizes compilation cost.
**Consequences**: Rules are evaluated per-pod; complex expressions can add latency. CEL environment is initialized once (`sync.Once`).

## Platform Documentation

For generic OpenShift development patterns, refer to the [openshift/enhancements](https://github.com/openshift/enhancements) repository:
- Development conventions: [`dev-guide/`](https://github.com/openshift/enhancements/tree/master/dev-guide)
- Enhancement process: [`guidelines/`](https://github.com/openshift/enhancements/tree/master/guidelines)
- Coding standards: [`CONVENTIONS.md`](https://github.com/openshift/enhancements/blob/master/CONVENTIONS.md)

## Known Issues & Operational Gotchas

> **Source**: Jira project MULTIARCH (component: Multiarch-Tuning-Operator) and Slack discussions from #forum-ocp-testplatform. Verified 2026-09-25.

### Active / Open Issues

| Jira | Summary | Impact |
|------|---------|--------|
| [MULTIARCH-6240](https://issues.redhat.com/browse/MULTIARCH-6240) | **Finalizer injection race condition** — The controller adds the `pod-placement` finalizer asynchronously during reconciliation. If a CPPC is deleted before the controller processes the create event, the object is deleted immediately with no cleanup, leaving orphaned operand resources (Deployments, RBAC, ServiceMonitor). **Proposed fix**: switch to a mutating admission webhook that injects the finalizer at creation time. The existing controller logic would remain as a fallback. | Affects rapid create/delete scenarios, GitOps reconciliation loops, and flaky E2E test "Should cleanup all finalizers" |
| [MULTIARCH-5569](https://issues.redhat.com/browse/MULTIARCH-5569) | **Network policies not implemented** — MTO does not currently deploy NetworkPolicy resources, leaving operand pods without network-level isolation. | Security hardening gap |
| [MULTIARCH-6087](https://issues.redhat.com/browse/MULTIARCH-6087) | **Deprecated events API** — MTO uses the legacy events API; migration to the new `events.k8s.io/v1` API is tracked. | Future deprecation risk |
| [MULTIARCH-6199](https://issues.redhat.com/browse/MULTIARCH-6199) | **UBI10 migration** — Container base images need migration from UBI9 to UBI10. | Build infrastructure modernization |
| [MULTIARCH-4984](https://issues.redhat.com/browse/MULTIARCH-4984) | **Image volumes handling** — Open question on how MTO should handle Kubernetes image volumes (a K8s 1.31+ feature). | Future feature gap |
| [MULTIARCH-6309](https://issues.redhat.com/browse/MULTIARCH-6309) | **ServicePortsMatch uses positional comparison** — Port comparison during resource apply uses index-based matching instead of name-keyed lookup, which can cause unnecessary resource updates. | Minor correctness issue |

### Recently Fixed Issues (Tribal Knowledge)

These bugs are fixed but document important behavioral patterns developers should understand:

| Jira | Summary | Fix Version | Root Cause & Lesson |
|------|---------|-------------|---------------------|
| [MULTIARCH-6269](https://issues.redhat.com/browse/MULTIARCH-6269) | CPPC deletion stuck due to informer cache race | v1.3.4 | The controller read CPPC via `r.Get()` (informer cache), which returned stale data without `DeletionTimestamp`. **Fix**: Use `r.APIReader.Get()` (direct API server read) at the top of `Reconcile` to always see the latest object state. **Lesson**: When deletion timing matters, prefer `APIReader` over the cached client. |
| [MULTIARCH-6239](https://issues.redhat.com/browse/MULTIARCH-6239) | PPC preferred affinity missed when informer cache is stale | — | Image inspection completed before the informer cache synced the PodPlacementConfig, causing preferred affinity terms to be skipped. **Lesson**: Same informer staleness pattern as MULTIARCH-6269. |
| [MULTIARCH-5800](https://issues.redhat.com/browse/MULTIARCH-5800) | Images with attestation manifests cause "unknown" architecture | v1.3 | OCI image indexes can contain attestation manifests with `platform.architecture: "unknown"`. MTO included these in the supported architecture set, resulting in `nodeAffinity` for architecture "unknown". **Fix**: Filter out attestation manifests during image inspection. **Lesson**: Always validate platform entries in manifest lists — not all entries represent real architectures. |
| [MULTIARCH-5764](https://issues.redhat.com/browse/MULTIARCH-5764) | Short image names fail inspection on OCP 4.21 | v1.3 | Kubelet on OCP 4.21 changed how short image names are resolved. MTO's image inspection did not account for this, causing failures when images used short names without a registry prefix. |
| [MULTIARCH-6270](https://issues.redhat.com/browse/MULTIARCH-6270) | enoexec-event-daemon DaemonSet fails in kustomize deployments | v1.3.4 | ServiceAccount pull secret race condition during startup. |
| [MULTIARCH-6271](https://issues.redhat.com/browse/MULTIARCH-6271) | OLM CSV lifecycle cycling on OCP 4.16 | v1.3.4 | OLM cycling prevented operator convergence and blocked CPPC finalizer processing. |
| [MULTIARCH-6236](https://issues.redhat.com/browse/MULTIARCH-6236) | CPPC deletion tears down operands before checking for PPC | — | Deletion order matters: operand resources were removed before checking if PodPlacementConfig objects still existed, causing orphaned PPC state. |
| [MULTIARCH-6207](https://issues.redhat.com/browse/MULTIARCH-6207) | eNoExecEvent daemon status update conflict race | — | Daemon and handler controller both update the same ENoExecEvent CR status, causing frequent conflict errors. |

### Common CI/Operational Issues (from Slack)

These patterns appear repeatedly in `#forum-ocp-testplatform` Slack discussions:

1. **Scheduling gate cycling**: Pods can appear to be gated, ungated, then re-gated in quick succession. This typically indicates the operator was restarted or the webhook was temporarily unavailable. Check operator pod logs for restart events.

2. **Image inspection failures with "manifest unknown"**: When the internal OpenShift image registry (`image-registry.svc:5000`) returns "manifest unknown", MTO cannot determine the image architecture. This usually indicates the image hasn't been fully imported yet. The `fallbackArchitecture` field on CPPC can mitigate this.

3. **Pods stuck in SchedulingGated state**: If the MTO controller pod is down or overloaded, pods remain gated indefinitely. The scheduling gate can be manually removed from individual pods as a workaround: `kubectl patch pod <name> --type=json -p '[{"op":"remove","path":"/spec/schedulingGates/0"}]'`

4. **CI multi-arch payload failures**: In OpenShift CI, architecture-specific image variants may be absent from the payload, causing MTO to set incorrect architecture constraints. Check that all container images in the pod spec have the expected architecture variants published.

5. **Short image name resolution**: On OCP 4.21+, ensure images use fully qualified names (including registry) to avoid inspection failures. This was fixed in MTO v1.3 ([MULTIARCH-5764](https://issues.redhat.com/browse/MULTIARCH-5764)).

## Design References — Upstream KEP Details

> **Source**: Verified from `kubernetes/enhancements` repository, 2026-09-25.

### KEP-3521: Pod Scheduling Readiness (Stable since K8s 1.30)

The scheduling gate mechanism (`.spec.schedulingGates`) is the foundation of MTO's approach. Key details from the upstream KEP:

- **State transition**: `schedulingGates` can only be set at pod creation (by client or mutating webhooks). After creation, gates can only be _removed_, never added. This is why MTO uses a mutating webhook to add the gate at admission time.
- **Restricted state**: A pod with `spec.nodeName` set cannot have scheduling gates (enforced by API server validation). This is why MTO skips pods with `spec.nodeName` already set.
- **Metric**: `scheduler_pending_pods{queue="gated"}` tracks the number of gated pods in the scheduler's queue.
- **Feature gate**: `PodSchedulingReadiness` — enabled by default since K8s 1.27 (beta), locked to enabled since K8s 1.30 (GA). Cannot be disabled in 1.30+.

### KEP-3838: Pod Mutable Scheduling Directives (Stable since K8s 1.30)

This KEP is what allows MTO to modify `nodeAffinity` on gated pods after creation:

- **Scope**: While a pod is gated (has any `schedulingGate`), its `nodeAffinity`, `nodeSelector`, and `tolerations` can be mutated. Once all gates are removed and the pod enters scheduling, these fields become immutable again.
- **Dependency**: Relies on KEP-3521's scheduling gates. Same feature gate (`PodSchedulingReadiness`).
- **Related**: KEP-2926 (Job Mutable Scheduling Directives) extends similar mutability to Job-owned pods.

## SME Review Recommended

- Exact ICSP/IDMS/ITMS interaction with image inspection (noted as TODO in `pkg/image/inspector.go:92`)
- eBPF tracepoint details and CRI runtime compatibility (`internal/controller/enoexecevent/daemon/internal/tracepoint/`)
- OLM upgrade strategy details (CSV `spec.replaces`/`skipRange` patterns)
