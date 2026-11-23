# `shouldupgrade` selection and migration

## Contract

Merge the following into the existing Git-managed definition or label owner of each intended ManagedCluster:

```yaml
metadata:
  labels:
    shouldupgrade: "true"
```

There is no member name in this fragment. Use the bare, case-sensitive key and string value shown. `autoshift.io/shouldupgrade`, a missing key, and values such as `"false"` or `"True"` do not satisfy the selector. Git or another framework may own the object; configure that owner rather than relying on an imperative label that it will undo.

The real Placement selects:

```yaml
labelSelector:
  matchLabels:
    shouldupgrade: "true"
  matchExpressions:
    - key: local-cluster
      operator: NotIn
      values: ["true"]
```

The extra role guard excludes the local hub used for synthetic tests. It is not a hardcoded ManagedCluster name. The synthetic Placement requires the same opt-in plus the local hub role, because its ConfigMaps are hub-local.

Placement selects all matching members in the ManagedClusterSets bound to `policies`, subject to its normal availability rules. `numberOfClusters` is omitted. ACM performs fan-out; each selected cluster has its own dependency evaluation and CVO. [Placement semantics](https://open-cluster-management.io/docs/concepts/content-placement/placement/)

## Review selection

Using an authenticated explicit hub context, inspect:

```bash
HUB_CONTEXT='your-hub-context'
oc --context="$HUB_CONTEXT" get managedclusters -l shouldupgrade=true --show-labels
oc --context="$HUB_CONTEXT" get placementdecisions -n policies \
  -l cluster.open-cluster-management.io/placement=live-upgrade-selected -o yaml
oc --context="$HUB_CONTEXT" get placementbindings -n policies -o yaml
```

Check every binding for an additional placement that could bypass selection. Review that at most ten eligible spokes are opted in for the proposed customer cohort. The controller does not enforce a global active-upgrade cap: eleven opted-in eligible spokes can be selected. Do not add a replacement while the approved cohort is still active.

## Removal is not cancellation

Removing the label or setting it to `"false"` causes the member to fall out of this profile after reconciliation. That withdraws safety and completion observation as well as request placement. Keep the label through postchecks; retire consumed admission through its owning Policy configuration instead. Previously delivered intent or an accepted CVO upgrade may continue after removal.

## Migrate the earlier example

The previous example created a named-spoke health Placement and a hub-side ClusterCurator trigger. This version replaces that backend with local native CVO policies. The file extension and default disabled flags do not remove resources already installed from an earlier revision.

1. In the **old owning Git source**, disable `live-upgrade-trigger`, deliver the change and verify it is disabled before changing the actuator type. Record existing execution evidence.
2. Resolve any current or queued ClusterCurator work through its supported owner. Do not enable a second writer. Removing a Policy does not cancel the persisted curator or a running Job; never delete an active lifecycle object to simulate cancellation.
3. Remove the old repo-owned hub/spoke bindings and placements from their Git source. The obsolete names are PlacementBindings `live-trigger-on-hub`, `live-admission-on-hub`, `live-test-spoke-health`; Placements `live-trigger-on-hub`, `live-test-spoke`; and the informational root Policy `live-upgrade-admission`. If your source does not prune removals, explicitly retire only those verified repo-owned resources. Preserve unrelated bindings, shared namespaces/cluster-set bindings, managed-cluster namespaces and evidence.
4. Deliver the new prepared manifests with both channel and upgrade request disabled. Their Placement/Binding is named `live-upgrade-selected`. Check there are no old bindings to the changed root Policies that bypass `shouldupgrade`.
5. Set the membership label through its Git owner. Verify the new PlacementDecisions and per-member readiness before activating the channel or request. The former per-member view/approval examples are no longer required by this backend; retain or remove any old read-only views through their owner, never by deleting a cluster namespace.

A root Policy called `live-upgrade-trigger` now contains a ClusterVersion assertion rather than a curator assertion. The shared name is not evidence that existing deployment ownership or lifecycle has been migrated. Review the actual resource and bindings.

## Manual acceptance

- Missing label and `"false"` value: no real-upgrade Policy replica is newly placed after reconciliation.
- Exact `"true"` on an eligible spoke: all real-upgrade phase Policies select that member, subject to individual disabled flags.
- Exact `"true"` on the marked local hub: synthetic demo eligible; real upgrade excluded.
- Multiple eligible labelled spokes: all selected; there is no first-match limit or cross-member readiness dependency.
- A member fails precheck: its request is blocked, without holding another eligible member.
- Label removed during a test: selection eventually withdraws, observations disappear, and no claim is made that accepted work was cancelled.
- Ten eligible members: qualify actual overlapping CVO updates before promising ten-way capacity.

These are acceptance cases, not a statement that this repository has executed them on a particular environment. No credentials or private execution evidence belong in public Git.
