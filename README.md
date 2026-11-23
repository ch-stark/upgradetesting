# OpenShift upgrade policies: concepts and manual examples

Plain YAML for manual testing from a Red Hat Advanced Cluster Management hub. No Go templates, Python runner, Ansible dependency, new CRD, or custom controller.

This repository contains two separate example paths and a customer design proposal:

| Path | Purpose | Changes a real upgrade target? |
|---|---|---|
| `overlays/` | Demonstrate policy phases using synthetic ConfigMap data on the local hub | No |
| `real-upgrade/` | Manually test one managed test cluster using ClusterCurator and informational health policies | Only after editing and explicitly enabling the disabled trigger |
| [Customer cohort design](docs/customer-cohort.md) | Proposed AutoShift-first journey for one Git-approved cohort of up to ten clusters | Documentation only; no additional upgrade executor is installed |

**If the hub is your only cluster, use the demo. A real upgrade needs a separate managed test cluster. Do not use the hub as the upgrade target.**

The proposed customer path reuses AutoShift's existing CVO upgrade flow, with local Policy Dependencies and exact targets approved in Git. The ClusterCurator example is a separate alternative, not a second writer to enable alongside that flow. AutoShift integration and ten-cluster capacity are not established by the example manifests. See the [customer design and acceptance criteria](docs/customer-cohort.md).

## Important corrections to the initial draft

- An Argo CD sync window does not create, delete, or expire a ConfigMap automatically.
- A hub lock is a delivery/intent signal, not an executor-side deadline. A disconnected spoke can later act on already-delivered intent.
- Closing the window does not cancel an active ClusterCurator Job or CVO upgrade.
- Root `remediationAction: enforce` overrides child remediation. Health checks are therefore in separate `inform` root policies, never inside the enforcing trigger policy.
- Informational `mustnothave` checks detect unwanted failure conditions. They must not be changed to enforce, which can request deletion of matching platform objects.
- Lower severity does not suppress native ACM noncompliance. Completion can remain NonCompliant while an upgrade progresses normally.

See [design boundaries](docs/design.md) and [manual acceptance checklist](docs/manual-checklist.md).

## Why only some Policies have cluster results

A root Policy remains listed in Governance even when it is disabled. A PlacementBinding can still reference it, but `spec.disabled: true` prevents propagation for evaluation. Its configured remediation mode can still read `enforce`; that does not mean it is currently enforcing.

An attended upgrade profile can leave informational baseline, safety, and completion checks active after the run, while retiring consumed channel/target/request Policies. Earlier synthetic scenarios should be kept separate from real health checks. A disabled or deferred check is **not a passed check**.

In Governance, a green check with `1` means one compliant cluster, not one violation. A dash is not a compliance pass; disabled Policies, empty selection, or missing evaluation can produce no current result. If a Policy should be active, inspect its disabled flag, PlacementDecisions, add-on health, and replicated-policy details. `Source: Local` describes the Policy's origin, not its cluster selection.

```bash
HUB_CONTEXT='your-hub-context'
oc --context="$HUB_CONTEXT" get policies -n policies \
  -o custom-columns=NAME:.metadata.name,DISABLED:.spec.disabled,MODE:.spec.remediationAction
oc --context="$HUB_CONTEXT" get placementdecisions -n policies -o yaml
```

One root Policy can evaluate on multiple clusters. A ten-cluster cohort does not require ten copies of every root Policy. The synthetic overlays in this repository intentionally enable all their demo checks; they do not automatically implement phase retirement. See [policy lifecycle and Governance interpretation](docs/policy-lifecycle.md) before changing activation in a real profile.

## Policy Dependency and namespaces

Root Policies, Placements, and PlacementBindings live in the existing **`policies`** namespace. Synthetic observations and the manual approval ConfigMap live in the disposable **`upgradetesting`** namespace. ACM still creates its replicated policies in managed-cluster namespaces; do not move those replicas.

This uses the native `spec.dependencies` and template `extraDependencies` patterns from [policydependencydemo](https://github.com/ch-stark/policydependencydemo/tree/d7ecb5bb11adbea22cabba7688c5da00aa1f54ec), written directly as static YAML. No Policy Generator installation is required. See [dependency design and manual tests](docs/policy-dependencies.md).

## 1. Start with the hub-only demo

Keep all edits in Git. Review/commit a scenario change before delivering it. Use your existing Argo CD Application, or apply the committed YAML manually for this lab. Manual `oc apply` is not controlled by Argo sync windows.

Set your actual hub context:

```bash
HUB_CONTEXT='your-hub-context'
oc --context="$HUB_CONTEXT" get managedclusters -l local-cluster=true
oc --context="$HUB_CONTEXT" get managedclusterset global
oc --context="$HUB_CONTEXT" get namespace policies
oc --context="$HUB_CONTEXT" get managedclustersetbinding global -n policies
oc --context="$HUB_CONTEXT" get namespace upgradetesting --ignore-not-found
```

The registered local hub must be available with the policy add-on enabled. The shared `policies` namespace must already be bound to the `global` cluster set, including the local hub and any real test target. Have the hub administrator establish that prerequisite if missing. The demo creates only its own `upgradetesting` namespace; it does not manage the shared namespace or binding. If `upgradetesting` already contains unrelated work, choose an isolated namespace consistently in Git first.

Deliver the first scenario:

```bash
oc --context="$HUB_CONTEXT" apply -k overlays/01-window-closed
oc --context="$HUB_CONTEXT" get placementdecisions -n policies
oc --context="$HUB_CONTEXT" get policies -n policies
```

Wait for policy propagation/evaluation. Inspect details if no results appear:

```bash
oc --context="$HUB_CONTEXT" describe policy upgrade-02-window-admission -n policies
```

Apply one scenario at a time, from its reviewed Git source:

```bash
oc --context="$HUB_CONTEXT" apply -k overlays/02-ready-window-open
oc --context="$HUB_CONTEXT" apply -k overlays/03-upgrading
oc --context="$HUB_CONTEXT" apply -k overlays/04-upgrade-failed
oc --context="$HUB_CONTEXT" apply -k overlays/05-window-closed-in-flight
oc --context="$HUB_CONTEXT" apply -k overlays/06-completed-window-closed
```

Additional scenarios are `07-stale-observation` and `08-precheck-blocked`.

### Expected demo results

`C` = Compliant, `NC` = NonCompliant. An unmet dependency leaves the admission template Pending; inspect the per-cluster details rather than relying on a single summary field.

| Scenario | Precheck | Window admission | Progress safety | Completion | Strict steady state |
|---|---|---|---|---|---|
| Window closed | C | NC | C | NC | NC |
| Ready, window open | C | C | C | NC | NC |
| Upgrading | C | C | C | NC | NC |
| Upgrade failed | C | C | NC | NC | NC |
| Window closed in flight | C | NC | C | NC | NC |
| Completed, window closed | C | NC | C | C | C |
| Stale observation | NC | Pending | NC | NC | NC |
| Precheck blocked | NC | Pending | C | NC | NC |

The demo's precheck represents a synthetic baseline result. Admission depends on that root Policy being Compliant. Completion's second template depends on its first ConfigurationPolicy being Compliant: target convergence first, settled health second. Progress safety stays independent, so a new failure remains visible even when another phase is blocked. These ConfigMap values are not measurements of actual cluster health. No policy enforces any change in this demo.

Every template has `ignorePending: false`. A blocked template is not treated as a passed check. Native ACM can summarize a Pending template as root-policy NonCompliant; read the per-cluster template messages.

The admission policy observes whether the lock is active. It does not execute an upgrade, reserve capacity, or provide late-spoke protection. All policies and templates are explicitly informational.

### Demo cleanup

If Argo owns these resources, remove/disable that source through Git first to prevent recreation. Confirm the namespace contains only the demo, then:

```bash
oc --context="$HUB_CONTEXT" delete -k overlays/01-window-closed
```

This deletes the five named demo root Policies, their Placement/Binding, and the dedicated demo namespace. It leaves the shared `policies` namespace and `global` binding intact. Do not run demo cleanup while using the real-upgrade approval ConfigMap in `upgradetesting`. It does not delete a managed-cluster namespace or curator.

## 2. Optional real upgrade: one managed test cluster

The files in `real-upgrade/` are static examples. They are not referenced by the demo. Their `.yaml.example` extension and disabled trigger prevent accidental activation through the demo or a directory apply.

Before testing:

1. Use a non-hub, standard self-managed OpenShift test cluster.
2. Choose an exact **recommended forward patch in its current minor release**. EUS and minor-version sequencing are outside this first manual test.
3. Confirm ClusterCurator and ManagedClusterView are installed and the hub/view agents have the required permissions.
4. Inspect any existing curator. Do not overwrite an active lifecycle operation or a curator controlled by another automation. The upgrade must not contain Ansible hooks, intermediate steps, or custom Job overrides.
5. Pause competing upgrade writers through their owning Git configuration. This concept has no fleet-wide concurrency lock.

### 2a. Prepare files in Git

Copy the `.yaml.example` files to `.yaml` files in a separate test Git path. Do not bulk-apply the directory.

Replace every `REPLACE_MANAGED_CLUSTER_NAME` and `REPLACE_TARGET_VERSION`. The cluster and target must match across the approval, admission, trigger, completion, and view files. No target/channel is chosen for you. Keep both approval values `false` initially.

```bash
oc --context="$HUB_CONTEXT" get managedcluster YOUR_TEST_CLUSTER --show-labels
oc --context="$HUB_CONTEXT" get clustercurator YOUR_TEST_CLUSTER -n YOUR_TEST_CLUSTER -o yaml
```

The health Placement assumes the ManagedCluster has an existing `name` label equal to its name. Verify it. If absent, select an existing unique label in Git instead. Do not change AutoShift-managed labels imperatively. Verify the resulting PlacementDecision names exactly the intended test cluster.

Leave `spec.disabled: true` in the trigger during preparation.

### 2b. Deploy read-only observation and checks

After reviewing/committing the prepared files, deliver them individually:

```bash
oc --context="$HUB_CONTEXT" get namespace policies
oc --context="$HUB_CONTEXT" get managedclustersetbinding global -n policies
oc --context="$HUB_CONTEXT" apply -f base/namespace.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/01-health-policies.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/02-spoke-placement.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/04-observation-views.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/03-trigger.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/05-hub-admission.yaml
```

The trigger must still have `disabled: true`. Applying it here creates the hub Placement used by admission without enabling the upgrade. The approval values must still be `false`.

The view resources are created on the hub in the selected managed-cluster namespace. They request read-only observations of remote ClusterVersion, ClusterOperators, and MachineConfigPools. No direct spoke login is required.

```bash
oc --context="$HUB_CONTEXT" get placementdecisions -n policies -o yaml
oc --context="$HUB_CONTEXT" get policies -n policies
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-version -n YOUR_TEST_CLUSTER -o yaml
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-operators -n YOUR_TEST_CLUSTER -o yaml
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-pools -n YOUR_TEST_CLUSTER -o yaml
```

Inspect the view conditions and results. Missing, stale, unknown, empty, or failed observations are not permission to proceed. The absence checks alone can miss missing/Unknown resources; inspect actual inventory and conditions manually.

Confirm baseline availability, no degradation/failure, no active upgrade, required pool convergence, and that the target appears in the remote ClusterVersion's `status.availableUpdates`. Perform the application's own readiness/backup checks as applicable. This example does not implement operator compatibility planning or application hooks.

### 2c. Explicitly activate one upgrade

The trigger is evaluated on the hub and creates/updates one curator in the explicitly named managed-cluster namespace. Its Placement is **not** the health policy's spoke Placement.

The trigger has a native Policy Dependency on `policies/live-upgrade-admission`. Both admission and trigger run on the local hub. Admission requires:

1. The approval ConfigMap to name this exact cluster/target and have `prechecksReviewed: 'true'` and `windowApproved: 'true'`.
2. A reported `Compliant` entry for this exact cluster in the hub root `policies/live-precheck` Policy's `status.status` list.
3. The ManagedCluster to report Joined and Available.

Admission's second template uses `extraDependencies` on the approval-check ConfigurationPolicy. This is an explicit hub-side observation bridge, not a native dependency across two execution clusters. It reads asynchronously reported status; it does not prove freshness or perform an atomic health-and-start operation. **You still review actual remote evidence manually before approval.**

After that review, change both approval values to `'true'` in the prepared `05-hub-admission.yaml`, review/commit, and deliver that file. Verify the local hub's admission result is Compliant and inspect its template messages. If it is Pending, NonCompliant, or Unknown, do not enable the trigger.

Then change only the prepared trigger's `spec.disabled` to `false`, review/commit it, and deliver:

```bash
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/03-trigger.yaml
```

This can start a real upgrade. The trigger contains only the curator assertion; the failure detectors are in separate informational policies. Verify no binding override changes their remediation. The trigger explicitly sets `pruneObjectBehavior: None`: loss of a dependency must not delete the curator. Loss of admission cannot undo an intent or stop an upgrade that has already started.

### 2d. Verify actual completion from the hub

```bash
oc --context="$HUB_CONTEXT" get clustercurator YOUR_TEST_CLUSTER -n YOUR_TEST_CLUSTER -o yaml
oc --context="$HUB_CONTEXT" get jobs -n YOUR_TEST_CLUSTER
oc --context="$HUB_CONTEXT" get policies -n policies
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-version -n YOUR_TEST_CLUSTER -o yaml
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-operators -n YOUR_TEST_CLUSTER -o yaml
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-pools -n YOUR_TEST_CLUSTER -o yaml
```

Do not equate curator existence, successful policy enforcement, or Argo Synced with completion. Verify:

- The latest ClusterVersion history entry, not just any historical entry, is the intended version with state `Completed`.
- ClusterVersion desired version is correct; `Available=True`, `Progressing=False`, and `Failing=False`.
- Required ClusterOperators are available and not degraded, and their rollout has settled.
- Required MachineConfigPools are updated, not updating/degraded/paused, and ready/updated counts equal machine counts.
- Curator/Job outcomes and independent application checks do not report an unresolved failure.

Completion noncompliance during a normal rollout is expected. Degraded/unavailable/failing observations still require investigation. Severity only prioritizes findings; it does not hide them.

### 2e. Stop further reconciliation after the test

Set the trigger back to disabled through Git and deliver the change. Revoke both approval values through Git as well. Neither action cancels an active upgrade or rolls back the platform. Do not delete/recreate a curator to retry or clean up while execution is unresolved. Do not delete the managed-cluster namespace or shared `policies` namespace. Retain the results for review.

## Argo CD

`argocd/` contains partial examples for an existing test project and Application. Review names, paths, permissions, and the installed version's window behavior. Do not replace existing project permissions with the snippets.

Only the demo overlay belongs in the first Argo Application. Keep the real trigger outside automated discovery until its exact target and manual process are reviewed.

The demo contains resources in both `policies` and `upgradetesting`, plus the demo Namespace resource. The existing AppProject and Argo service account must permit those destinations/resource kinds. Explicit resource namespaces are intentional; do not add a Kustomize namespace override that moves every resource into one namespace.

An open window allows delivery. It does not expire the lock, guarantee a CVO start time, or stop an already-running upgrade. A lock-closing Git change can itself wait behind a closed sync window. Strict deadlines require executor-side enforcement and are not claimed here.

## Validation status

All scenarios can be rendered locally with `oc kustomize overlays/SCENARIO`. YAML/render inspection is not live admission, policy propagation, or upgrade validation. Qualify the actual ACM/controller behavior, including the hub-side status bridge and access to root Policies, in the intended environment. This public repository does not establish ten-way parallel upgrade capacity or strict executor-side start deadlines.

Keep credentials, kubeconfigs, cluster endpoints, recovery files, and environment-specific test evidence outside public Git. Use placeholders in shared examples. Private execution records are not part of this repository's public validation claims.

If you deployed the previous revision, moving a resource's namespace does not remove its old copy. Follow the [migration instructions](docs/policy-dependencies.md#migrating-an-earlier-checkout) before applying this revision.
