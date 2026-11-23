# Design boundaries

## Reuse

Git stores intent and Argo CD can gate delivery. The proposed customer path reuses AutoShift's existing native CVO upgrade flow and adds only the required local health gates. This repository packages a synthetic demo and a separate ClusterCurator reference example; it does not install that AutoShift integration. Select one upgrade writer per cluster. There is no new scheduling service, templating layer, or API.

The customer design scope is one explicit cohort of up to ten non-hub clusters, with exact targets approved in Git. This does not require a fleet queue, and it does not establish a global active-upgrade limit. See [customer cohort design](customer-cohort.md) for operating scope and acceptance criteria.

## Hub lock

The demo lock is a persistent ConfigMap whose active value is changed in Git. No component automatically deletes it at window close. Its policy is informational and demonstrates admission state only.

It cannot invalidate an upgrade instruction already delivered to a spoke. Hub-side rendering, cached policies, asynchronous propagation, and executor delay all prevent an exact-time claim. Do not advertise safe late-spoke no-op behavior from lock deletion.

## Policy phases

- Precheck: baseline evidence before admission. The demo uses an explicitly synthetic value.
- Admission: window state, dependent on the demo precheck Policy. The optional real trigger depends on a separate hub admission Policy that observes the selected spoke's reported precheck and an explicit Git approval.
- Progress safety: independent failure observations, never disabled by a failing precheck.
- Completion: target convergence; unsatisfied while progressing. In the demo, a template extraDependency gates settled-health checks on the target-convergence ConfigurationPolicy.
- Steady state: deliberately strict comparison to demonstrate why generic health checks can appear red during transitions.

Native ACM compliance does not gain a new Upgrading value. Read the phase and the underlying findings. A green admission/progress policy is not upgrade success.

Root Policy existence, placement, activation, evaluation, and execution completion are distinct. An attended profile may retire consumed admission Policies while preserving independent safety checks and running postchecks when due. Disabled and Pending do not mean passed. This lifecycle needs explicit wiring in the owning Git source; it is not automatic in these examples. See [policy lifecycle](policy-lifecycle.md).

Root Policies and placement resources live in the existing `policies` namespace. Test data lives in `upgradetesting`. The shared namespace and its cluster-set binding are prerequisites, not resources owned by this demo. See [Policy Dependency implementation](policy-dependencies.md) for execution locations, native dependency fields, and manual tests.

## Remediation separation

All demo policies and real health checks use informational root policies. The real enforcing root policy contains only a ClusterCurator assertion. Root/binding remediation overrides must never turn the failure detector's absence assertions into deletion requests.

No postcheck uses a deleted lock or a fallback target. The target is explicit static Git configuration and must match the trigger. No blanket operator exclusion, unknown-state suppression, or automatic cleanup is implemented.

## First real test

One explicit managed test cluster, one exact recommended same-minor patch, manual precheck approval, dependency-gated activation, manual verification. The hub admission policy observes the remote precheck's per-cluster root status; this asynchronous bridge does not establish evidence freshness or an atomic start decision. No Ansible hooks, EUS orchestration, fleet queue, automatic retry, or rollback. Existing curator ownership and lifecycle must be reviewed before applying intent.

This is a concept repository, not a supported product promise or evidence of ten-way parallel execution. Qualify a real managed-cluster upgrade, failure isolation, and then a ten-member overlapping run before a customer capacity claim. Strict downstream start-window enforcement remains a separate requirement.

## References

- [Argo CD Sync Windows](https://argo-cd.readthedocs.io/en/stable/user-guide/sync_windows/)
- [OCM remediation and policy concepts](https://open-cluster-management.io/docs/getting-started/integration/policy-controllers/policy/)
- [ClusterCurator source](https://github.com/stolostron/cluster-curator-controller)
- [Policy Dependency demo reviewed](https://github.com/ch-stark/policydependencydemo/tree/d7ecb5bb11adbea22cabba7688c5da00aa1f54ec)
- [Upstream Policy API: dependencies and per-cluster status](https://github.com/open-cluster-management-io/governance-policy-propagator/blob/main/api/v1/policy_types.go)
- [Customer trigger reviewed](https://github.com/ch-stark/clustercurator/blob/ed25df257c3682cde3a3d8dcc1f791c24dbe6d09/clustercuratorpolicy.yaml)
- [Community upgrade sample](https://github.com/open-cluster-management-io/policy-collection/blob/7aed4ecbcdfd5074002f2f460b652c85acfab9de/community/CM-Configuration-Management/policy-upgrade-openshift-cluster.yaml)
