# Graph Report - .  (2026-04-24)

## Corpus Check
- 177 files · ~45,450 words
- Verdict: corpus is large enough that graph structure adds value.

## Summary
- 1063 nodes · 1930 edges · 52 communities detected
- Extraction: 59% EXTRACTED · 41% INFERRED · 0% AMBIGUOUS · INFERRED: 785 edges (avg confidence: 0.8)
- Token cost: 73,220 input · 5,248 output

## Community Hubs (Navigation)
- [[_COMMUNITY_RBAC Builders|RBAC Builders]]
- [[_COMMUNITY_ConfigMap Builders|ConfigMap Builders]]
- [[_COMMUNITY_Pod Affinity Builders|Pod Affinity Builders]]
- [[_COMMUNITY_Logging & Verbosity|Logging & Verbosity]]
- [[_COMMUNITY_API Type Definitions|API Type Definitions]]
- [[_COMMUNITY_RBAC Subject Builders|RBAC Subject Builders]]
- [[_COMMUNITY_Authentication|Authentication]]
- [[_COMMUNITY_Pod Placement Core|Pod Placement Core]]
- [[_COMMUNITY_ENoExecEvent API|ENoExecEvent API]]
- [[_COMMUNITY_ENoExec Daemon & eBPF|ENoExec Daemon & eBPF]]
- [[_COMMUNITY_Plugin System|Plugin System]]
- [[_COMMUNITY_E2E Testing|E2E Testing]]
- [[_COMMUNITY_Test Registry|Test Registry]]
- [[_COMMUNITY_Main Controllers|Main Controllers]]
- [[_COMMUNITY_Design Rationale|Design Rationale]]
- [[_COMMUNITY_Image Caching|Image Caching]]
- [[_COMMUNITY_Pod Builders|Pod Builders]]
- [[_COMMUNITY_ENoExecEvent Model|ENoExecEvent Model]]
- [[_COMMUNITY_ENoExec Architecture|ENoExec Architecture]]
- [[_COMMUNITY_CPPC Builders|CPPC Builders]]
- [[_COMMUNITY_Deployment Builders|Deployment Builders]]
- [[_COMMUNITY_Service Builders|Service Builders]]
- [[_COMMUNITY_Volume Builders|Volume Builders]]
- [[_COMMUNITY_Container Builders|Container Builders]]
- [[_COMMUNITY_Secret Builders|Secret Builders]]
- [[_COMMUNITY_Node Builders|Node Builders]]
- [[_COMMUNITY_Job Builders|Job Builders]]
- [[_COMMUNITY_StatefulSet Builders|StatefulSet Builders]]
- [[_COMMUNITY_DaemonSet Builders|DaemonSet Builders]]
- [[_COMMUNITY_CronJob Builders|CronJob Builders]]
- [[_COMMUNITY_Webhook Configuration|Webhook Configuration]]
- [[_COMMUNITY_Metrics & Monitoring|Metrics & Monitoring]]
- [[_COMMUNITY_Image Inspection|Image Inspection]]
- [[_COMMUNITY_Registry Client|Registry Client]]
- [[_COMMUNITY_Namespace Selector|Namespace Selector]]
- [[_COMMUNITY_Leader Election|Leader Election]]
- [[_COMMUNITY_Finalizers|Finalizers]]
- [[_COMMUNITY_Status Conditions|Status Conditions]]
- [[_COMMUNITY_Reconciler Patterns|Reconciler Patterns]]
- [[_COMMUNITY_Event Publishing|Event Publishing]]
- [[_COMMUNITY_Community 40|Community 40]]
- [[_COMMUNITY_Community 41|Community 41]]
- [[_COMMUNITY_Community 42|Community 42]]
- [[_COMMUNITY_Community 43|Community 43]]
- [[_COMMUNITY_Community 44|Community 44]]
- [[_COMMUNITY_Community 45|Community 45]]
- [[_COMMUNITY_Community 46|Community 46]]
- [[_COMMUNITY_Community 47|Community 47]]
- [[_COMMUNITY_Community 48|Community 48]]
- [[_COMMUNITY_Community 67|Community 67]]
- [[_COMMUNITY_Community 68|Community 68]]
- [[_COMMUNITY_Community 69|Community 69]]

## God Nodes (most connected - your core abstractions)
1. `NewPod()` - 39 edges
2. `Deploy()` - 39 edges
3. `Namespace()` - 38 edges
4. `Pod` - 23 edges
5. `PodBuilder` - 23 edges
6. `PodSpecBuilder` - 19 edges
7. `runManager()` - 18 edges
8. `TestPod_SetPreferredArchNodeAffinityPPC()` - 17 edges
9. `main()` - 16 edges
10. `TestPod_SetPreferredArchNodeAffinityWithCPPC()` - 16 edges

## Surprising Connections (you probably didn't know these)
- `CommonBeforeSuite()` --uses--> `E2E Test Constants`  [INFERRED]
  /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/pkg/e2e/suites_lifecycle.go → pkg/e2e/const.go
- `TestPod_shouldIgnorePod()` --calls--> `NewOwnerReferenceBuilder()`  [INFERRED]
  /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/internal/controller/podplacement/pod_model_test.go → /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/pkg/testing/builder/owner_reference.go
- `newPod()` --calls--> `NewPod()`  [INFERRED]
  /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/internal/controller/podplacement/pod_model.go → /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/pkg/testing/builder/pod.go
- `ensureDeletion()` --calls--> `Namespace()`  [INFERRED]
  /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/internal/controller/enoexecevent/handler/enoexecevent_controller_test.go → /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/pkg/utils/runtime.go
- `ensureErrorLabel()` --calls--> `Namespace()`  [INFERRED]
  /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/internal/controller/enoexecevent/handler/enoexecevent_controller_test.go → /Users/kpais/kpais-workspace/claude-tmp/multiarch-tuning-operator/pkg/utils/runtime.go

## Hyperedges (group relationships)
- **** — scheduling_gates_concept_pattern, scheduling_gates_operator_usage, pod_placement_webhook_admission_flow, pod_placement_controller_reconciliation_flow [INFERRED]
- **** — design_zero_touch_philosophy, design_safety_through_gating_rationale, design_image_inspection_rationale, design_fail_open_rationale [INFERRED]
- **** — reliability_slo_webhook_availability, reliability_slo_ungating_latency, reliability_prometheus_metrics [INFERRED]
- **E2E Test Suite Initialization** — suites_lifecycle_commoninit, suites_lifecycle_commonbeforesuite, e2e_test_operator_init, e2e_test_operator_suite [EXTRACTED 1.00]
- **Pod Placement Affinity Test Cases** — pod_placement_test_non_conflicting_affinity, pod_placement_test_preferred_affinity_append, pod_placement_test_user_conflict [EXTRACTED 1.00]
- **E2E Test Workflow Pattern** — e2e_test_pattern_ephemeral_namespace, e2e_test_pattern_deployment_creation, e2e_test_pattern_verification [EXTRACTED 1.00]
- **ENoExec Monitoring Flow** — enoexec_monitoring_architecture_syscall_execve, enoexec_monitoring_architecture_enoexec_daemon, enoexec_monitoring_architecture_enoexecevent_crd, enoexec_monitoring_architecture_enoexec_handler, enoexec_monitoring_architecture_pod_events [EXTRACTED 1.00]
- **ENoExec RBAC Components** — enoexec_monitoring_architecture_rbac_daemon, enoexec_monitoring_architecture_rbac_handler, enoexec_monitoring_architecture_enoexec_daemon, enoexec_monitoring_architecture_enoexec_handler [EXTRACTED 1.00]
- **Infrastructure Services for ENoExec** — enoexec_monitoring_architecture_etcd, enoexec_monitoring_architecture_prometheus, enoexec_monitoring_architecture_node [EXTRACTED 1.00]

## Communities

### Community 0 - "RBAC Builders"
Cohesion: 0.03
Nodes (54): ClusterRoleBindingBuilder, ClusterRoleBuilder, ImageContentSourcePolicyBuilder, ImageDigestMirrorsBuilder, ImageDigestMirrorSetBuilder, ImageTagMirrorSetBuilder, MutatingWebhookConfigurationBuilder, RepositoryDigestMirrorsBuilder (+46 more)

### Community 1 - "ConfigMap Builders"
Cohesion: 0.03
Nodes (20): ConfigMapBuilder, ContainerBuilder, ContainerEnvBuilder, DaemonSetBuilder, DeploymentConfigBuilder, PodSpecBuilder, ServiceBuilder, ServicePortBuilder (+12 more)

### Community 2 - "Pod Affinity Builders"
Cohesion: 0.07
Nodes (45): PodPlacementConfigBuilder, PreferredSchedulingTermBuilder, PreferredSchedulingTermsBuilder, NewClusterPodPlacementConfig(), TestClusterPodPlacementConfigStatus_Build(), InitPodPlacementControllerMetrics(), FacadeSingleton(), TestEnsureArchitectureLabels() (+37 more)

### Community 3 - "Logging & Verbosity"
Cohesion: 0.07
Nodes (36): isDeploymentAvailable(), isDeploymentUpToDate(), LogVerbosityLevel, buildClusterRoleENoExecEventsController(), buildClusterRoleENoExecEventsDaemonSet(), buildDaemonSetENoExecEvent(), buildDeploymentENoExecEventHandler(), buildExecFormatErrorAvailabilityAlertRule() (+28 more)

### Community 4 - "API Type Definitions"
Cohesion: 0.05
Nodes (18): conditionFromBool(), init(), notFromBool(), Test_conditionFromBool(), Test_notFromBool(), Test_trimAndCapitalize(), trimAndCapitalize(), ClusterPodPlacementConfig (+10 more)

### Community 5 - "RBAC Subject Builders"
Cohesion: 0.06
Nodes (36): SubjectBuilder, CPPCSyncer, GetClusterPodPlacementConfig(), NewCPPCSyncer(), AllSupportedArchitecturesSet(), ExecFormatErrorEventMessage(), registryInspector, isBundleImage() (+28 more)

### Community 6 - "Authentication"
Cohesion: 0.05
Nodes (31): matchAndExpandGlob(), Test_authCfg_expandGlobs(), Test_matchAndExpandGlob(), NodeSelectorRequirementBuilder, NodeSelectorTermBuilder, equivalentNodeAffinityMatcher, equivalentPreferredNodeAffinityMatcher, ParseSchemelessURL() (+23 more)

### Community 7 - "Pod Placement Core"
Cohesion: 0.08
Nodes (8): Pod, newPod(), containerImage, Pod, PodReconciler, PodSchedulingGateMutatingWebHook, ArchLabelValue(), HistogramObserve()

### Community 8 - "ENoExecEvent API"
Cohesion: 0.05
Nodes (23): ENoExecEventBuilder, createENEEAndUpdateStatus(), createPodAndUpdateStatus(), defaultENoExecFormatError(), deletePod(), ensureDeletion(), ensureErrorLabel(), ensureEvent() (+15 more)

### Community 9 - "ENoExec Daemon & eBPF"
Cohesion: 0.09
Nodes (19): RunDaemon(), runWorker(), NewK8sENOExecEventStorage(), registerScheme(), K8sENOExecEventStorage, getPodContainerUUIDFor(), getPodNameFromUUID(), NewTracepoint() (+11 more)

### Community 10 - "Plugin System"
Cohesion: 0.07
Nodes (11): BasePlugin, ExecFormatErrorMonitor, IBasePlugin, LocalPlugins, NodeAffinityScoring, NodeAffinityScoringPlatformTerm, Plugins, TestBasePlugin_IsEnabled() (+3 more)

### Community 11 - "E2E Testing"
Cohesion: 0.07
Nodes (28): E2E Test Constants, Operator Test Suite Init, Operator E2E Test Suite, E2E Test Pattern - Deployment Creation, E2E Test Pattern - Ephemeral Namespace, E2E Test Pattern - Pod Verification, E2E Image Inspection Workflow, E2E Node Affinity Computation Workflow (+20 more)

### Community 12 - "Test Registry"
Cohesion: 0.12
Nodes (17): buildRegistryTLSConfig(), getClusterProxy(), MockImage, NewRegistry(), RegistryConfig, registryTLSConfig, RunRegistry(), setupRegistry() (+9 more)

### Community 13 - "Main Controllers"
Cohesion: 0.13
Nodes (19): InitCommonMetrics(), NewReconciler(), bindFlags(), btoi(), init(), initContext(), main(), must() (+11 more)

### Community 14 - "Design Rationale"
Cohesion: 0.08
Nodes (27): Fail Open Design Rationale, High Controller Concurrency Rationale, Image Inspection Over Heuristics Rationale, Operator-of-Operators Pattern, Safety Through Gating Design Rationale, Zero-Touch Multi-Arch Philosophy, CGO Dependencies (gpgme), Image Inspection Caching Strategy (+19 more)

### Community 15 - "Image Caching"
Cohesion: 0.11
Nodes (10): computeFNV128Hash(), newCacheProxy(), newImageFacade(), cacheProxy, Facade, registryInspector, cacheProxy, Facade (+2 more)

### Community 16 - "Pod Builders"
Cohesion: 0.11
Nodes (1): PodBuilder

### Community 17 - "ENoExecEvent Model"
Cohesion: 0.17
Nodes (1): ENoExecEvent

### Community 18 - "ENoExec Architecture"
Cohesion: 0.18
Nodes (13): ENoExec Monitoring Architecture Diagram, enoexec_cnt Container, enoexec-event-daemon (DaemonSet), ENOEXEC Error, enoexec-event-handler (Controller), EnoexecEvent CRD, etcd (Kubernetes API Storage), Kubernetes Node (+5 more)

### Community 19 - "CPPC Builders"
Cohesion: 0.18
Nodes (1): ClusterPodPlacementConfigBuilder

### Community 20 - "Deployment Builders"
Cohesion: 0.22
Nodes (1): StatefulSetBuilder

### Community 21 - "Service Builders"
Cohesion: 0.28
Nodes (2): ImageTagMirrorsBuilder, NewImageTagMirrors()

### Community 22 - "Volume Builders"
Cohesion: 0.22
Nodes (1): ContainerStatusBuilder

### Community 23 - "Container Builders"
Cohesion: 0.25
Nodes (1): DeploymentBuilder

### Community 24 - "Secret Builders"
Cohesion: 0.25
Nodes (1): SecurityContextBuilder

### Community 25 - "Node Builders"
Cohesion: 0.25
Nodes (1): Jobbuilder

### Community 26 - "Job Builders"
Cohesion: 0.29
Nodes (2): accessController, challenge

### Community 27 - "StatefulSet Builders"
Cohesion: 0.29
Nodes (1): BuildBuilder

### Community 28 - "DaemonSet Builders"
Cohesion: 0.29
Nodes (1): NodeBuilder

### Community 29 - "CronJob Builders"
Cohesion: 0.47
Nodes (1): ClusterPodPlacementConfigValidator

### Community 30 - "Webhook Configuration"
Cohesion: 0.33
Nodes (2): OwnerReferenceBuilder, NewOwnerReferenceBuilder()

### Community 31 - "Metrics & Monitoring"
Cohesion: 0.33
Nodes (2): NodeAffinityBuilder, NodeAffinityTerm

### Community 32 - "Image Inspection"
Cohesion: 0.4
Nodes (3): IStorage, IWStorage, IWStorageBase

### Community 33 - "Registry Client"
Cohesion: 0.5
Nodes (1): PodPlacementConfigReconciler

### Community 34 - "Namespace Selector"
Cohesion: 0.83
Nodes (3): patchDeploymentStatus(), setDeploymentReady(), validateReconcile()

### Community 35 - "Leader Election"
Cohesion: 0.83
Nodes (3): VerifyCOAreUpdated(), VerifyCOAreUpdating(), WaitForCOComplete()

### Community 36 - "Finalizers"
Cohesion: 0.83
Nodes (3): VerifyMCPAreUpdated(), VerifyMCPsAreUpdating(), WaitForMCPComplete()

### Community 37 - "Status Conditions"
Cohesion: 0.5
Nodes (4): Coverage Metric (40 points), Line Budget Trade-off Rationale, Navigation Metric (10 points), Quality Assessment Process

### Community 38 - "Reconciler Patterns"
Cohesion: 0.67
Nodes (2): ICache, IRegistryInspector

### Community 39 - "Event Publishing"
Cohesion: 1.0
Nodes (2): GetNodesWithLabel(), GetRandomNodeName()

### Community 40 - "Community 40"
Cohesion: 0.67
Nodes (3): envtest Integration Testing, Ginkgo/Gomega Test Framework, Test Pyramid Strategy

### Community 41 - "Community 41"
Cohesion: 0.67
Nodes (3): Prometheus Metrics System, Pod Ungating Latency SLO (P95 < 10s), Webhook Availability SLO (99.9%)

### Community 42 - "Community 42"
Cohesion: 0.67
Nodes (3): ClusterPodPlacementConfig Singleton Pattern, Fallback Architecture Configuration, Namespace Selector Configuration

### Community 43 - "Community 43"
Cohesion: 0.67
Nodes (3): Operator CPPC With Fallback Architecture Test, Operator CPPC With Plugins Test, Operator Failed Image Inspection Test

### Community 44 - "Community 44"
Cohesion: 1.0
Nodes (1): Plugin

### Community 45 - "Community 45"
Cohesion: 1.0
Nodes (2): Pod Admission Flow, Asynchronous Event Publication

### Community 46 - "Community 46"
Cohesion: 1.0
Nodes (2): Operator V1Alpha1 Deployment Test, Operator V1Beta1 Deployment Test

### Community 47 - "Community 47"
Cohesion: 1.0
Nodes (2): Operator Namespace Selector Opt-In Test, Operator Namespace Selector Opt-Out Test

### Community 48 - "Community 48"
Cohesion: 1.0
Nodes (2): Operator Local PodPlacementConfig Test, Operator PPC Priority Test

### Community 67 - "Community 67"
Cohesion: 1.0
Nodes (1): Binary Execution Modes

### Community 68 - "Community 68"
Cohesion: 1.0
Nodes (1): Documentation Generation Phases

### Community 69 - "Community 69"
Cohesion: 1.0
Nodes (1): Pod Placement Config E2E Test Suite

## Knowledge Gaps
- **62 isolated node(s):** `containerImage`, `sharedData`, `IWStorage`, `IStorage`, `Plugin` (+57 more)
  These have ≤1 connection - possible missing edges or undocumented components.
- **Thin community `Pod Builders`** (22 nodes): `PodBuilder`, `.Build()`, `.WithAffinity()`, `.WithAnnotations()`, `.WithContainer()`, `.WithContainerImagePullAlways()`, `.WithContainersImages()`, `.WithContainerStatuses()`, `.WithGenerateName()`, `.WithImagePullSecrets()`, `.WithInitContainersImages()`, `.WithLabels()`, `.WithName()`, `.WithNamespace()`, `.WithNodeAffinity()`, `.WithNodeName()`, `.WithNodeSelectors()`, `.WithOwnerReference()`, `.WithOwnerReferences()`, `.WithRequiredDuringSchedulingIgnoredDuringExecution()`, `.WithSchedulingGates()`, `pod.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `ENoExecEvent Model`** (13 nodes): `NewENoExecEvent()`, `ENoExecEvent`, `.Ctx()`, `.ENoExecEventObject()`, `.EnsureLabel()`, `.EnsureNoLabel()`, `.HasErrorLabel()`, `.IsMarkedAsError()`, `.MarkAsError()`, `.PublishEvent()`, `.PublishEventOnPod()`, `.Recorder()`, `enoexecevent_model.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `CPPC Builders`** (11 nodes): `ClusterPodPlacementConfigBuilder`, `.Build()`, `.WithExecFormatErrorMonitor()`, `.WithFallbackArchitecture()`, `.WithLogVerbosity()`, `.WithName()`, `.WithNamespaceSelector()`, `.WithNodeAffinityScoring()`, `.WithNodeAffinityScoringTerm()`, `.WithPlugins()`, `clusterpodplacementconfig.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Deployment Builders`** (9 nodes): `StatefulSetBuilder`, `.Build()`, `.WithName()`, `.WithNamespace()`, `.WithPodSpec()`, `.WithReplicas()`, `.WithSelectorAndPodLabels()`, `statefulset.go`, `NewStatefulSet()`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Service Builders`** (9 nodes): `ImageTagMirrorsBuilder`, `.Build()`, `.WithMirrorAllowContactingSource()`, `.WithMirrorNeverContactSource()`, `.WithMirrors()`, `.WithMirrorSourcePolicy()`, `.WithSource()`, `NewImageTagMirrors()`, `image_tag_mirror.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Volume Builders`** (9 nodes): `ContainerStatusBuilder`, `.Build()`, `.WithID()`, `.WithName()`, `.WithReady()`, `.WithRestartCount()`, `.WithState()`, `NewContainerStatus()`, `container_status.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Container Builders`** (8 nodes): `DeploymentBuilder`, `.Build()`, `.WithName()`, `.WithNamespace()`, `.WithPodSpec()`, `.WithReplicas()`, `.WithSelectorAndPodLabels()`, `deployment.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Secret Builders`** (8 nodes): `SecurityContextBuilder`, `.Build()`, `.WithPrivileged()`, `.WithRunAsGroup()`, `.WithRunAsUSer()`, `.WithSeccompProfileType()`, `security_context.go`, `NewSecurityContext()`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Node Builders`** (8 nodes): `Jobbuilder`, `.Build()`, `.WithName()`, `.WithNamespace()`, `.WithPodLabels()`, `.WithPodSpec()`, `NewJob()`, `job.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Job Builders`** (7 nodes): `init()`, `newAccessController()`, `accessController`, `.Authorized()`, `challenge`, `.SetHeaders()`, `access.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `StatefulSet Builders`** (7 nodes): `NewBuild()`, `BuildBuilder`, `.Build()`, `.WithDockerImage()`, `.WithName()`, `.WithNamespace()`, `build.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `DaemonSet Builders`** (7 nodes): `NodeBuilder`, `.Build()`, `.WithAnnotation()`, `.WithLabel()`, `.WithTaint()`, `NewNodeBuilder()`, `node.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `CronJob Builders`** (6 nodes): `clusterpodplacementconfig_webhook.go`, `ClusterPodPlacementConfigValidator`, `.validate()`, `.ValidateCreate()`, `.ValidateDelete()`, `.ValidateUpdate()`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Webhook Configuration`** (6 nodes): `OwnerReferenceBuilder`, `.Build()`, `.WithController()`, `.WithKind()`, `NewOwnerReferenceBuilder()`, `owner_reference.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Metrics & Monitoring`** (6 nodes): `NodeAffinityBuilder`, `.Build()`, `.WithPreferredNodeAffinity()`, `NodeAffinityTerm`, `NewNodeAffinityBuilder()`, `node_affinity.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Registry Client`** (4 nodes): `podplacementconfig_controller.go`, `PodPlacementConfigReconciler`, `.Reconcile()`, `.SetupWithManager()`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Reconciler Patterns`** (3 nodes): `ICache`, `IRegistryInspector`, `interfaces.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Event Publishing`** (3 nodes): `GetNodesWithLabel()`, `GetRandomNodeName()`, `node.go`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Community 44`** (2 nodes): `const.go`, `Plugin`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Community 45`** (2 nodes): `Pod Admission Flow`, `Asynchronous Event Publication`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Community 46`** (2 nodes): `Operator V1Alpha1 Deployment Test`, `Operator V1Beta1 Deployment Test`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Community 47`** (2 nodes): `Operator Namespace Selector Opt-In Test`, `Operator Namespace Selector Opt-Out Test`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Community 48`** (2 nodes): `Operator Local PodPlacementConfig Test`, `Operator PPC Priority Test`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Community 67`** (1 nodes): `Binary Execution Modes`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Community 68`** (1 nodes): `Documentation Generation Phases`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.
- **Thin community `Community 69`** (1 nodes): `Pod Placement Config E2E Test Suite`
  Too small to be a meaningful cluster - may be noise or needs more connections extracted.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **Why does `Deploy()` connect `ConfigMap Builders` to `RBAC Builders`, `Pod Affinity Builders`, `Logging & Verbosity`, `Test Registry`?**
  _High betweenness centrality (0.126) - this node is a cross-community bridge._
- **Why does `Namespace()` connect `Logging & Verbosity` to `RBAC Builders`, `Pod Affinity Builders`, `Namespace Selector`, `RBAC Subject Builders`, `Pod Placement Core`, `ENoExecEvent API`, `Main Controllers`?**
  _High betweenness centrality (0.092) - this node is a cross-community bridge._
- **Why does `TestPod_shouldIgnorePod()` connect `Pod Affinity Builders` to `RBAC Builders`, `Logging & Verbosity`, `RBAC Subject Builders`, `Pod Placement Core`, `Pod Builders`, `Webhook Configuration`?**
  _High betweenness centrality (0.054) - this node is a cross-community bridge._
- **Are the 37 inferred relationships involving `NewPod()` (e.g. with `TestPod_GetPodImagePullSecrets()` and `TestPod_HasSchedulingGate()`) actually correct?**
  _`NewPod()` has 37 INFERRED edges - model-reasoned connections that need verification._
- **Are the 37 inferred relationships involving `Deploy()` (e.g. with `.Build()` and `.WithSelector()`) actually correct?**
  _`Deploy()` has 37 INFERRED edges - model-reasoned connections that need verification._
- **Are the 36 inferred relationships involving `Namespace()` (e.g. with `main()` and `RunOperator()`) actually correct?**
  _`Namespace()` has 36 INFERRED edges - model-reasoned connections that need verification._
- **What connects `containerImage`, `sharedData`, `IWStorage` to the rest of the system?**
  _62 weakly-connected nodes found - possible documentation gaps or missing edges._