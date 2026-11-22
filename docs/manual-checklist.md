# Manual acceptance checklist

Record the hub ACM/MCE/OpenShift versions, Git commit, selected cluster, exact target, and observed results. Do not commit tokens or kubeconfigs.

## Hub-only demo

- [ ] Only the dedicated demo namespace is used.
- [ ] PlacementDecision selects the registered local hub.
- [ ] All five root policies and their templates are informational.
- [ ] Closed admission is noncompliant without hiding progress-safety results.
- [ ] Progressing produces incomplete completion/steady-state results.
- [ ] A synthetic degradation is visible in progress safety.
- [ ] Stale observations fail safety/prechecks; deferred admission is not success.
- [ ] Completion can pass while admission is closed.
- [ ] No ClusterCurator or ClusterVersion mutation was created by the demo.

## Optional real upgrade

- [ ] Separate standard managed test cluster; not the hub.
- [ ] Every placeholder replaced in reviewed Git files.
- [ ] Health PlacementDecision names exactly that cluster.
- [ ] Existing curator and competing writers reviewed; no active operation, hooks, or custom override.
- [ ] Remote view conditions/results are usable and actual resources are present.
- [ ] Exact forward patch is recommended for that cluster.
- [ ] Prechecks and independent readiness checks reviewed before enabling the trigger.
- [ ] Only the curator assertion is in an enforcing policy.
- [ ] Latest CVO history confirms the exact target Completed.
- [ ] Operators, pools, counts, curator outcome, and application checks reviewed.
- [ ] Trigger disabled afterward through Git; active work is not deleted or cancelled.

## Argo integration, if used

- [ ] Test Application points only to a demo overlay.
- [ ] Matching AppProject window and any manual overrides inspected.
- [ ] Closed-window Git change waits; permitted delivery is observable.
- [ ] No claim that lock expiry/deletion or a strict execution deadline is automatic.
