# Design boundaries

## Reuse

Git stores intent. Argo CD can gate delivery. ACM expresses checks and the explicitly enabled curator intent. ClusterCurator and CVO perform the real upgrade. There is no new scheduling service, templating layer, or API.

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

Root Policies and placement resources live in the existing `policies` namespace. Test data lives in `upgradetesting`. The shared namespace and its cluster-set binding are prerequisites, not resources owned by this demo. See [Policy Dependency implementation](policy-dependencies.md) for execution locations, native dependency fields, and manual tests.

## Remediation separation

All demo policies and real health checks use informational root policies. The real enforcing root policy contains only a ClusterCurator assertion. Root/binding remediation overrides must never turn the failure detector's absence assertions into deletion requests.

No postcheck uses a deleted lock or a fallback target. The target is explicit static Git configuration and must match the trigger. No blanket operator exclusion, unknown-state suppression, or automatic cleanup is implemented.

## First real test

One explicit managed test cluster, one exact recommended same-minor patch, manual precheck approval, dependency-gated activation, manual verification. The hub admission policy observes the remote precheck's per-cluster root status; this asynchronous bridge does not establish evidence freshness or an atomic start decision. No Ansible hooks, EUS orchestration, fleet queue, automatic retry, or rollback. Existing curator ownership and lifecycle must be reviewed before applying intent.

This is a concept repository, not a supported product promise or evidence of 100-cluster capacity.

## References

- [Argo CD Sync Windows](https://argo-cd.readthedocs.io/en/stable/user-guide/sync_windows/)
- [OCM remediation and policy concepts](https://open-cluster-management.io/docs/getting-started/integration/policy-controllers/policy/)
- [ClusterCurator source](https://github.com/stolostron/cluster-curator-controller)
- [Policy Dependency demo reviewed](https://github.com/ch-stark/policydependencydemo/tree/d7ecb5bb11adbea22cabba7688c5da00aa1f54ec)
- [Upstream Policy API: dependencies and per-cluster status](https://github.com/open-cluster-management-io/governance-policy-propagator/blob/main/api/v1/policy_types.go)
- [Customer trigger reviewed](https://github.com/ch-stark/clustercurator/blob/ed25df257c3682cde3a3d8dcc1f791c24dbe6d09/clustercuratorpolicy.yaml)
- [Community upgrade sample](https://github.com/open-cluster-management-io/policy-collection/blob/7aed4ecbcdfd5074002f2f460b652c85acfab9de/community/CM-Configuration-Management/policy-upgrade-openshift-cluster.yaml)
