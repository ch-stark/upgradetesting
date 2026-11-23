# Manual acceptance checklist

Record installed versions, source commit, label membership, exact targets and outcomes privately. Do not commit credentials, kubeconfigs or environment-specific execution records.

## Label selection

- [ ] Root resources are in `policies`; the appropriate cluster set is bound to it.
- [ ] Membership comes from the exact bare label `shouldupgrade: "true"` in its owning Git source.
- [ ] Missing, false, and differently cased values do not select a member.
- [ ] The marked local hub is excluded from real upgrades even when opted into the synthetic demo.
- [ ] No ManagedCluster name is hardcoded in selectors or the actuator; no first-member numeric limit is present.
- [ ] PlacementDecisions match all reviewed eligible opted-in spokes; no extra binding bypasses the selector.
- [ ] At most ten members are opted in for the customer cohort; this is reviewed membership, not an automatic concurrency cap.
- [ ] Existing older deployments completed the label/backend migration before activation.

## Synthetic hub demo

- [ ] The local hub has both the membership and local-hub role labels.
- [ ] Synthetic ConfigMaps remain in `upgradetesting`; shared `policies` namespace/binding are not owned/deleted by the demo.
- [ ] All demo Policies/templates are informational and change no real upgrade target.
- [ ] Failed precheck leaves admission Pending via a Policy dependency.
- [ ] Completion template 2 waits on template 1 via ConfigurationPolicy extraDependencies.
- [ ] `ignorePending: false`; Pending is not counted as a passed check.
- [ ] Safety remains independent during injected failure or closed admission.
- [ ] Completion can pass with the synthetic admission lock closed.

## Real native CVO path

- [ ] Selected members are standard self-managed, non-hub test clusters with usable policy add-ons.
- [ ] Previous curator/upgrade ownership is resolved; no competing active or queued writer remains.
- [ ] Channel and trigger start disabled; prepared values and recommended update paths are reviewed.
- [ ] Required resource presence, usable conditions, critical alerts, pools, backup/application readiness and observation freshness are reviewed for every member.
- [ ] Channel preparation runs while the upgrade trigger stays disabled.
- [ ] Exact version and recommended payload are pinned consistently in health, target and request manifests.
- [ ] Each member's precheck, channel and exact-target Policies must be Compliant locally before its trigger can act.
- [ ] One blocked member does not hold another eligible member through a fleet-wide delivery barrier.
- [ ] The request contains only ClusterVersion desiredUpdate with `force: false`; no health observation is under an enforcing root.
- [ ] Platform templates use `pruneObjectBehavior: None`; dependency loss is not deletion or cancellation.
- [ ] Keep `shouldupgrade: "true"` through required safety/postchecks; do not use removal as a stop command.
- [ ] Latest CVO history confirms the exact target Completed; current conditions, actual operator/pool inventory, nodes and application outcomes agree.
- [ ] Consumed channel/target/request disabled through their owner; membership removed only after outcomes and monitoring needs are resolved.

## Qualification and delivery

- [ ] All policy definitions can be delivered without a global health barrier that serializes the cohort unintentionally.
- [ ] Ten real eligible CVO updates overlap before making the capacity claim; an unlabelled non-member remains untouched.
- [ ] Label removal, stale/unknown evidence, controller restart and repeated delivery have defined, verified outcomes.
- [ ] Argo examples deploy only the intended path; namespace permissions and manual overrides are reviewed.
- [ ] No claim that a label or delivery window enforces an exact downstream start deadline or cancels work.
