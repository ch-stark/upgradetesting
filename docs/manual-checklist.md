# Manual acceptance checklist

Record the hub ACM/MCE/OpenShift versions, Git commit, selected cluster, exact target, and observed results. Do not commit tokens or kubeconfigs.

## Hub-only demo

- [ ] Root resources are in `policies`; synthetic objects are in `upgradetesting`.
- [ ] Shared `policies` namespace and `global` binding already exist and are not owned/deleted by the demo.
- [ ] PlacementDecision selects the registered local hub.
- [ ] All five root policies and their templates are informational.
- [ ] Closed admission is noncompliant without hiding progress-safety results.
- [ ] Progressing produces incomplete completion/steady-state results.
- [ ] A synthetic degradation is visible in progress safety.
- [ ] Stale observations fail safety/prechecks; deferred admission is not success.
- [ ] Blocked precheck leaves admission's template Pending with a Policy dependency message.
- [ ] Completion template 2 waits on completion template 1 via ConfigurationPolicy extraDependencies.
- [ ] Every template has `ignorePending: false`; blocked is not treated as passed.
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
- [ ] Hub admission and trigger select only the local hub; remote health checks select only the intended spoke.
- [ ] Approval initially false; exact target/cluster agree across all prepared files.
- [ ] Hub admission reads this cluster's Compliant entry in `policies/live-precheck.status.status` and Joined/Available conditions.
- [ ] Admission template access and actual installed controller behavior verified; missing or stale evidence is not accepted.
- [ ] Native trigger dependency names `policies/live-upgrade-admission`; admission Compliant before activation.
- [ ] Only the curator assertion is in an enforcing policy.
- [ ] Curator ConfigurationPolicy uses `pruneObjectBehavior: None`; loss of admission is not cancellation/deletion.
- [ ] Latest CVO history confirms the exact target Completed.
- [ ] Operators, pools, counts, curator outcome, and application checks reviewed.
- [ ] Trigger disabled and both approval flags revoked afterward through Git; active work is not deleted or cancelled.

## Argo integration, if used

- [ ] Test Application points only to a demo overlay.
- [ ] Matching AppProject window and any manual overrides inspected.
- [ ] Argo can deliver explicit resources into both `policies` and `upgradetesting`, including the test Namespace; no blanket namespace rewrite.
- [ ] Closed-window Git change waits; permitted delivery is observable.
- [ ] No claim that lock expiry/deletion or a strict execution deadline is automatic.
