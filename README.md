# OpenShift upgrade testing: first concept

Plain YAML for manual testing from a Red Hat Advanced Cluster Management hub. No Go templates, Python runner, Ansible dependency, new CRD, or custom controller.

There are two separate paths:

| Path | Purpose | Changes a real upgrade target? |
|---|---|---|
| `overlays/` | Demonstrate policy phases using synthetic ConfigMap data on the local hub | No |
| `real-upgrade/` | Manually test one managed test cluster using ClusterCurator and informational health policies | Only after editing and explicitly enabling the disabled trigger |

**If the hub is your only cluster, use the demo. A real upgrade needs a separate managed test cluster. Do not use the hub as the upgrade target.**

## Important corrections to the initial draft

- An Argo CD sync window does not create, delete, or expire a ConfigMap automatically.
- A hub lock is a delivery/intent signal, not an executor-side deadline. A disconnected spoke can later act on already-delivered intent.
- Closing the window does not cancel an active ClusterCurator Job or CVO upgrade.
- Root `remediationAction: enforce` overrides child remediation. Health checks are therefore in separate `inform` root policies, never inside the enforcing trigger policy.
- Informational `mustnothave` checks detect unwanted failure conditions. They must not be changed to enforce, which can request deletion of matching platform objects.
- Lower severity does not suppress native ACM noncompliance. Completion can remain NonCompliant while an upgrade progresses normally.

See [design boundaries](docs/design.md) and [manual acceptance checklist](docs/manual-checklist.md).

## 1. Start with the hub-only demo

Keep all edits in Git. Review/commit a scenario change before delivering it. Use your existing Argo CD Application, or apply the committed YAML manually for this lab. Manual `oc apply` is not controlled by Argo sync windows.

Set your actual hub context:

```bash
HUB_CONTEXT='your-hub-context'
oc --context="$HUB_CONTEXT" get managedclusters -l local-cluster=true
oc --context="$HUB_CONTEXT" get managedclusterset global
oc --context="$HUB_CONTEXT" get namespace upgradetesting --ignore-not-found
```

The registered local hub must be available with the policy add-on enabled. The demo creates only its own `upgradetesting` namespace. If that namespace already contains unrelated work, stop and choose an isolated namespace consistently in Git first.

Deliver the first scenario:

```bash
oc --context="$HUB_CONTEXT" apply -k overlays/01-window-closed
oc --context="$HUB_CONTEXT" get placementdecisions -n upgradetesting
oc --context="$HUB_CONTEXT" get policies -n upgradetesting
```

Wait for policy propagation/evaluation. Inspect details if no results appear:

```bash
oc --context="$HUB_CONTEXT" describe policy upgrade-02-window-admission -n upgradetesting
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

The demo's precheck represents a synthetic baseline result. Progress safety stays independent, so a new failure remains visible even after admission. These ConfigMap values are not measurements of actual cluster health. No policy enforces any change in this demo.

The admission policy observes whether the lock is active. It does not execute an upgrade, reserve capacity, or provide late-spoke protection. All policies and templates are explicitly informational.

### Demo cleanup

If Argo owns these resources, remove/disable that source through Git first to prevent recreation. Confirm the namespace contains only the demo, then:

```bash
oc --context="$HUB_CONTEXT" delete -k overlays/01-window-closed
```

This deletes the dedicated demo namespace. It does not delete any managed-cluster namespace or real upgrade object.

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

Replace every `REPLACE_MANAGED_CLUSTER_NAME` and `REPLACE_TARGET_VERSION`. The target must match in the trigger and completion policy. No target/channel is chosen for you.

```bash
oc --context="$HUB_CONTEXT" get managedcluster YOUR_TEST_CLUSTER --show-labels
oc --context="$HUB_CONTEXT" get clustercurator YOUR_TEST_CLUSTER -n YOUR_TEST_CLUSTER -o yaml
```

The health Placement assumes the ManagedCluster has an existing `name` label equal to its name. Verify it. If absent, select an existing unique label in Git instead. Do not change AutoShift-managed labels imperatively. Verify the resulting PlacementDecision names exactly the intended test cluster.

Leave `spec.disabled: true` in the trigger during preparation.

### 2b. Deploy read-only observation and checks

After reviewing/committing the prepared files, deliver them individually:

```bash
oc --context="$HUB_CONTEXT" apply -f real-upgrade/00-namespace.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/01-health-policies.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/02-spoke-placement.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/04-observation-views.yaml
```

The view resources are created on the hub in the selected managed-cluster namespace. They request read-only observations of remote ClusterVersion, ClusterOperators, and MachineConfigPools. No direct spoke login is required.

```bash
oc --context="$HUB_CONTEXT" get placementdecisions -n upgradetesting-live -o yaml
oc --context="$HUB_CONTEXT" get policies -n upgradetesting-live
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-version -n YOUR_TEST_CLUSTER -o yaml
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-operators -n YOUR_TEST_CLUSTER -o yaml
oc --context="$HUB_CONTEXT" get managedclusterview upgradetesting-pools -n YOUR_TEST_CLUSTER -o yaml
```

Inspect the view conditions and results. Missing, stale, unknown, empty, or failed observations are not permission to proceed. The absence checks alone can miss missing/Unknown resources; inspect actual inventory and conditions manually.

Confirm baseline availability, no degradation/failure, no active upgrade, required pool convergence, and that the target appears in the remote ClusterVersion's `status.availableUpdates`. Perform the application's own readiness/backup checks as applicable. This example does not implement operator compatibility planning or application hooks.

### 2c. Explicitly activate one upgrade

The trigger is evaluated on the hub and creates/updates one curator in the explicitly named managed-cluster namespace. Its Placement is **not** the health policy's spoke Placement.

There is deliberately no claimed automatic cross-cluster dependency gate between the hub trigger and the spoke precheck. **You check the spoke result manually before activation.**

After confirming the correct target and prechecks, change only the prepared trigger's `spec.disabled` to `false`, review/commit it, and deliver:

```bash
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/03-trigger.yaml
```

This can start a real upgrade. The trigger contains only the curator assertion; the failure detectors are in separate informational policies. Verify no binding override changes their remediation.

### 2d. Verify actual completion from the hub

```bash
oc --context="$HUB_CONTEXT" get clustercurator YOUR_TEST_CLUSTER -n YOUR_TEST_CLUSTER -o yaml
oc --context="$HUB_CONTEXT" get jobs -n YOUR_TEST_CLUSTER
oc --context="$HUB_CONTEXT" get policies -n upgradetesting-live
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

Set the trigger back to disabled through Git and deliver the change. This does not cancel an active upgrade or roll back the platform. Do not delete/recreate a curator to retry or clean up while execution is unresolved. Do not delete the managed-cluster namespace. Retain the results for review.

## Argo CD

`argocd/` contains partial examples for an existing test project and Application. Review names, paths, permissions, and the installed version's window behavior. Do not replace existing project permissions with the snippets.

Only the demo overlay belongs in the first Argo Application. Keep the real trigger outside automated discovery until its exact target and manual process are reviewed.

An open window allows delivery. It does not expire the lock, guarantee a CVO start time, or stop an already-running upgrade. A lock-closing Git change can itself wait behind a closed sync window. Strict deadlines require executor-side enforcement and are not claimed here.

## Validation status

All scenarios can be rendered locally with `oc kustomize overlays/SCENARIO`. YAML/render inspection is not live admission, policy propagation, or upgrade validation. The actual ACM/controller behavior will be checked manually on the test hub.
