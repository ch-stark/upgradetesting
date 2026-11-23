# Policy lifecycle and Governance results

A Policy row in Governance means that the root object exists on the hub. It does not prove that the Policy is currently propagated, evaluated, enforcing, or that an upgrade has completed.

## Read the states separately

| Observation | Meaning | What it does not prove |
|---|---|---|
| Root Policy in `policies` | Authored definition exists | An active target or current evaluation |
| PlacementDecision | A Placement selected a cluster | All bound Policies are enabled or healthy |
| `spec.disabled: true` | Root Policy is disabled; replicas are removed | A passed check, cancellation, or rollback |
| Unmet dependency / Pending | Required compliance has not been satisfied | Evaluated readiness or successful execution |
| Green compliant-cluster count | That many targets report compliance for this Policy | Overall upgrade completion |
| Remediation `enforce` | Configured remediation mode | Current execution while the Policy is disabled |
| `Source: Local` | Locally authored Policy origin | Restriction to the managed cluster named `local-cluster` |

For a Policy expected to be active, inspect the disabled flag and actual PlacementDecisions first. Then check add-on availability, propagation, per-cluster template messages, and evidence freshness. A dash in a console table is not sufficient to distinguish these cases.

## An attended phase lifecycle

The following is a proposed operating pattern. It is not an automatic transition controller included in this repository.

1. **Prepare:** review the explicit cohort, approved channel, exact targets, current health, critical alerts, and recovery/application prerequisites. Keep upgrade requests disabled during preparation.
2. **Admit:** require the selected member's local readiness and target checks. Keep `ignorePending: false`; a blocked check must not authorize work. Another member's failure must not block an otherwise eligible member through a fleet-wide delivery barrier.
3. **Record acceptance:** distinguish written intent from actual CVO acceptance and progress. Preserve the Git revision, target, and per-member outcome.
4. **Close consumed admission:** after every member has an accepted/start or explicitly blocked outcome, retire consumed requests and admission checks through their owning Git configuration. Do not disable a shared gate when only the first member has started. Preserve non-pruning behavior for platform objects and verify generated resources actually reflect the intended change.
5. **Observe:** keep independent safety monitoring active. Do not remove its placement when withdrawing admission opt-in. Missing or stale required signals remain unresolved.
6. **Verify:** activate final postchecks when their phase is due. Once active, completion must not depend on an open maintenance window or a still-Compliant precheck. Disabled postchecks are not counted as passed.
7. **Handover:** retain appropriate informational health/completion checks; leave further upgrade intent disabled until another reviewed change authorizes it. Remove membership opt-in only after required monitoring/outcomes are resolved.

Disabling a Policy does not retract intent already accepted by another controller. `pruneObjectBehavior: None` preserves managed objects when checking/enforcing templates are retired; it is not a cancellation mechanism. Never treat removal of a Policy as a platform downgrade or safe deletion of an active execution object.

An Argo-managed resource can be reconciled back if changed imperatively. Phase activation belongs in the source that owns the resource. Confirm the installed AutoShift profile supports the intended lifecycle before relying on it; no unimplemented phase API is implied here.

## Reduce misleading failure interpretation

- `Progressing=True` and mixed platform versions can be expected during rollout. They are not completion and do not, by themselves, establish a failed upgrade.
- Strict readiness checks can become unsatisfied after work starts. Their admission meaning differs from an independent safety finding.
- An approved target can leave the available-update list after it has been consumed. That does not reverse already-accepted work.
- An optional empty pool can be valid. Verify the expected inventory and convergence; do not infer that every pool must contain a machine.
- Pool readiness for the desired configuration is distinct from Kubernetes node readiness. Check both before declaring completion.
- Historical failed Jobs, superseded installation plans, or retained failure fields need correlation to the current operation. An older successful history entry is also insufficient.
- Unknown or stale **required health signals** are unresolved. Specify the relevant conditions rather than treating every auxiliary condition identically.
- Critical alerts, missing prerequisites, and persistent degradation require investigation. Do not blanket-disable real safety Policies, suppress unknowns, or lower severity to manufacture a healthy result.

Informational `mustnothave` failure checks must stay under informational roots. An enforcing root or binding override can turn an absence assertion into a deletion request. Keep execution intent separate from health observations.

This approach reduces expected phase noise in a defined profile. It does not promise zero findings across unrelated fleet Policies, add an `Upgrading` compliance value to ACM, or make skipped checks successful.

## References

- [ACM Policy Dependencies](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.12/html/governance/policy-deployment)
- [Root Policy API: disabled, dependencies, remediation and Pending aggregation](https://github.com/open-cluster-management-io/governance-policy-propagator/blob/main/api/v1/policy_types.go)
- [ConfigurationPolicy pruning API](https://github.com/open-cluster-management-io/config-policy-controller/blob/main/api/v1/configurationpolicy_types.go)
- [Argo CD Sync Windows](https://argo-cd.readthedocs.io/en/stable/user-guide/sync_windows/)
