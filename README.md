# OpenShift upgrade policies: label-selected manual examples

Plain YAML for a Git-reviewed OpenShift upgrade workflow with ACM Policy Dependencies. No Go templates, Python runner, Ansible dependency, new CRD, or custom controller is added.

**Eligibility is opt-in:** a ManagedCluster must have the exact label `shouldupgrade: "true"`. Missing, `"false"`, and differently cased values are ignored. ACM Placement performs the fan-out across matching clusters; no ManagedCluster name is hardcoded and no custom inventory loop is required.

| Path | Purpose | Real platform changes? |
|---|---|---|
| `overlays/` | Synthetic ConfigMap scenarios on an opted-in local hub | No; all demo Policies are informational |
| `real-upgrade/` | Native CVO upgrade on eligible managed spokes, with local dependency gates | Only after replacing target placeholders and explicitly enabling the disabled channel/request Policies |
| [Customer cohort design](docs/customer-cohort.md) | AutoShift-first design for a reviewed cohort of up to ten | Design and acceptance criteria; not a capacity result |

The real-upgrade path now uses ClusterVersion directly. The earlier hub-side ClusterCurator trigger and named-cluster approval/view examples have been removed. **Existing deployments must follow the [migration instructions](docs/label-selection.md#migrate-the-earlier-example) before enabling the new trigger.** Use only one upgrade writer per cluster.

## Set membership in Git

Merge this fragment into each intended ManagedCluster's existing Git-managed definition, or configure its label owner to preserve the same bare key:

```yaml
metadata:
  labels:
    shouldupgrade: "true"
```

This is a metadata fragment, not a complete resource. It deliberately contains no cluster name. `shouldupgrade` and `autoshift.io/shouldupgrade` are different keys; the examples require the former. Do not patch AutoShift-owned labels imperatively or assume its prefixed values automatically create a bare label.

Both paths require this opt-in. The synthetic demo additionally requires `local-cluster: "true"` because its input ConfigMaps exist on the hub. The real path excludes clusters marked `local-cluster: "true"`, so opting the hub into the synthetic demo cannot select it for a real upgrade.

| Labels | Synthetic demo | Real upgrade |
|---|---|---|
| Missing `shouldupgrade`, or value other than `"true"` | Ignored | Ignored |
| `shouldupgrade: "true"`, local hub | Eligible | Excluded by hub-role guard |
| `shouldupgrade: "true"`, managed spoke | Ignored | Eligible within bound cluster sets and Placement availability rules |

There is no `numberOfClusters: 1` or first-match selection. All matching spokes can proceed independently. For the proposed customer cohort, review that at most ten members are opted in before activation. **This label is not a runtime concurrency cap:** if more are opted in, Placement can select more. Keep membership frozen through the run.

Keep `shouldupgrade: "true"` until postchecks are finished. Removing it withdraws both request and health-policy placement after reconciliation; it does not cancel an accepted CVO upgrade or guarantee instantaneous expiry of previously delivered intent. Close consumed admission by disabling the channel/target/request Policies through their owning Git source, while retaining membership and independent safety checks.

## Prerequisites

Use an already authenticated, explicit hub context. No credentials belong in this repository:

```bash
HUB_CONTEXT='your-hub-context'
oc --context="$HUB_CONTEXT" get namespace policies
oc --context="$HUB_CONTEXT" get managedclustersetbinding global -n policies
oc --context="$HUB_CONTEXT" get managedclusters -l shouldupgrade=true --show-labels
```

Root Policies, Placements and PlacementBindings live in `policies`. The existing namespace must be bound to an appropriate ManagedClusterSet. The selected clusters need usable policy add-ons. The examples do not own the shared namespace/binding or change cluster-set membership.

## 1. Synthetic hub-only demo

Opt the registered local hub into the demo through its label owner in Git. The real-upgrade Placement still excludes that hub. The demo creates only its own `upgradetesting` data namespace; check for unrelated work before using that name.

Review/commit changes before delivering them through your GitOps process or manually for this lab. Manual `oc apply` is not controlled by Argo sync windows.

```bash
oc --context="$HUB_CONTEXT" apply -k overlays/01-window-closed
oc --context="$HUB_CONTEXT" get placementdecisions -n policies -o yaml
oc --context="$HUB_CONTEXT" get policies -n policies
```

If nothing is selected, check `shouldupgrade` on the local ManagedCluster and the cluster-set binding first. Apply one scenario at a time from its reviewed source:

```bash
oc --context="$HUB_CONTEXT" apply -k overlays/02-ready-window-open
oc --context="$HUB_CONTEXT" apply -k overlays/03-upgrading
oc --context="$HUB_CONTEXT" apply -k overlays/04-upgrade-failed
oc --context="$HUB_CONTEXT" apply -k overlays/05-window-closed-in-flight
oc --context="$HUB_CONTEXT" apply -k overlays/06-completed-window-closed
```

Additional scenarios are `07-stale-observation` and `08-precheck-blocked`. They set synthetic inputs; they do not measure a real cluster.

`C` means Compliant; `NC` means NonCompliant. An unmet dependency leaves a template Pending; inspect per-cluster details rather than inferring success from a summary.

| Scenario | Precheck | Admission | Safety | Completion | Strict steady state |
|---|---|---|---|---|---|
| Window closed | C | NC | C | NC | NC |
| Ready, window open | C | C | C | NC | NC |
| Upgrading | C | C | C | NC | NC |
| Upgrade failed | C | C | NC | NC | NC |
| Window closed in flight | C | NC | C | NC | NC |
| Completed, window closed | C | NC | C | C | C |
| Stale observation | NC | Pending | NC | NC | NC |
| Precheck blocked | NC | Pending | C | NC | NC |

All templates use `ignorePending: false`. Admission depends on the precheck Policy; completion uses a ConfigurationPolicy extraDependency. Safety remains independent. See [dependency details](docs/policy-dependencies.md).

For cleanup, stop any Argo source that would recreate the demo, verify its ownership, then remove only this demo:

```bash
oc --context="$HUB_CONTEXT" delete -k overlays/01-window-closed
```

This removes its Policies/Placement/Binding and dedicated data namespace. It leaves the shared `policies` namespace, cluster-set binding, and real-upgrade resources alone. Do not remove a membership label still needed by an active real run.

## 2. Label-selected real upgrade

The `.yaml.example` files are excluded from the demo. Copy them into a separate reviewed Git path as `.yaml` files and deliver individually, not by bulk-applying the repository.

| File | Role |
|---|---|
| `01-health-policies.yaml.example` | Informational precheck, independent safety, and completion |
| `02-spoke-placement.yaml.example` | One label-selected Placement and binding for all real upgrade Policies |
| `03-trigger.yaml.example` | Disabled-by-default native CVO request, gated on local precheck/channel/target Policies |
| `04-channel-and-target.yaml.example` | Disabled-by-default channel change and informational exact-target validation |

The Placement requires `shouldupgrade: "true"`, excludes the hub role, and has no cluster-name predicate or numeric selection limit. Each selected member evaluates its own ClusterVersion and health; a blocked member does not make another member wait for fleet-wide compliance.

`ClusterVersion` named `version` is the platform singleton inside each member, not a hardcoded ManagedCluster name.

### Prepare and inspect

Use standard self-managed, non-hub OpenShift test clusters. Start with a supported, recommended forward patch within the current minor release. Check installed versions, existing upgrade ownership, recovery/application prerequisites, critical alerts, pool convergence and actual required resource inventory. Resolve or retire competing ClusterCurator/upgrade automation through its supported owner before enabling this path.

Prepare `REPLACE_APPROVED_CHANNEL` in the channel/target file. Keep both `live-upgrade-channel` and `live-upgrade-trigger` disabled initially. Prepare the exact version and recommended payload for the intended cohort when known; all target placeholders must be resolved before request activation. These static examples use a common approved target/image. Do not assume a single image is valid for incompatible architectures or update paths.

```bash
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/01-health-policies.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/02-spoke-placement.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/03-trigger.yaml
oc --context="$HUB_CONTEXT" apply -f PATH_TO_PREPARED/04-channel-and-target.yaml
oc --context="$HUB_CONTEXT" get placementdecisions -n policies \
  -l cluster.open-cluster-management.io/placement=live-upgrade-selected -o yaml
oc --context="$HUB_CONTEXT" get policy live-precheck -n policies -o yaml
```

Verify the decisions match the reviewed opted-in spokes and no old binding bypasses the label selector. An expected cluster with a false/missing label must not be selected. An empty selection does not demonstrate passed checks.

### Channel, target, then request

After the selected members' prechecks and the required independent readiness review, enable only `live-upgrade-channel` through Git and deliver the channel/target file. Keep the request disabled while recommendations refresh. If an existing writer or active upgrade is unresolved, do not proceed.

Resolve the latest eligible recommendation within the approved channel/boundary once. Pin `REPLACE_TARGET_VERSION` and `REPLACE_APPROVED_RELEASE_IMAGE` consistently in the health, target, and request files, review/commit, and deliver them with the request still disabled. Empty, unsupported, conditional, stale, or unknown recommendations are not permission to force a payload. Where available, use `oc adm upgrade recommend --version=EXACT_VERSION` or an equivalent authorized monitoring review. A direct ClusterVersion write does not prove CLI-side guards ran.

Inspect per-cluster results for `live-precheck`, `live-upgrade-channel`, and `live-upgrade-target`. They must be Compliant on a member before its request can act. Then explicitly enable `live-upgrade-trigger` through Git and deliver it. The request writes only that member's `ClusterVersion.spec.desiredUpdate`, with an exact version/image and `force: false`.

Target validation can become unsatisfied after the release is consumed, and strict prechecks can become unsatisfied during normal progress. These are admission states, not automatic proof of a failed upgrade. Preserve independent safety. See [phase lifecycle](docs/policy-lifecycle.md).

### Completion and handover

Read each selected cluster's actual completion and health evidence through your authorized console, direct context, or existing observation tooling. The Policy result is supporting evidence; a static history-list match does not prove that the latest entry completed. Verify latest CVO history, exact target/image, Available/Progressing/Failing conditions, operator health, actual pool counts/configuration, nodes, and application outcome.

After every member has an accepted/start or explicitly blocked outcome, retire consumed channel/target/request Policies through Git without removing the membership label. All enforcing templates use `pruneObjectBehavior: None`; retiring them does not delete ClusterVersion, cancel work, or roll the platform back. Keep safety and required postchecks selected until outcomes are recorded. Remove opt-in only after that work is resolved.

There is no automatic retry, queue, target resolver, phase controller, or strict executor-side deadline in these examples. A ten-member selection is not ten-way execution proof; see [customer qualification](docs/customer-cohort.md).

## Governance, remediation and delivery boundaries

A root Policy stays listed when disabled. A green check with `1` means one compliant target, not one violation; a dash is not a passed check. The remediation column shows configured mode, and `Source: Local` describes origin rather than cluster selection. One shared root Policy can evaluate on many selected members. See [Governance interpretation](docs/policy-lifecycle.md).

Keep health checks under informational roots. Root/binding remediation overrides must never turn a `mustnothave` observation into deletion enforcement. Missing required resources, unknown critical conditions and stale data remain unresolved; lower severity and `ignorePending: true` do not establish readiness.

`argocd/` contains partial delivery examples. Sync windows permit delivery; they do not create another target each week, expire a lock/label, guarantee when CVO starts, or cancel already-delivered intent. Keep both namespace permissions and explicit target selection under review. See [design boundaries](docs/design.md).

## Validation and privacy

Render the synthetic scenarios with `oc kustomize overlays/SCENARIO`. Static parsing and rendering do not validate real admission, update execution, concurrency, or start deadlines. Rehearse label opt-in, false/missing-label exclusion, failure isolation, removal behavior, and the intended ten-way concurrency in the target environment.

Keep credentials, kubeconfigs, cluster endpoints, recovery files and private test evidence outside public Git. This repository publishes generic patterns, not private execution records.
