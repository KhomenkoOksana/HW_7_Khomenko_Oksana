# GitOps Delivery Assessment for an ML Service

**Argo CD | Dev / Staging / Production | PM Decision Framework**

## Purpose

Assess whether a proposed GitOps delivery process for an inference API or MLflow service is controlled, auditable, recoverable, and safe enough for production.

The PM owns delivery governance and release decisionsвЂ”not Kubernetes or Helm implementation.

## 1. Proposed GitOps Process

Git is the source of truth for the desired deployment state: versioned manifests or Helm values describe what should run, where, and with which configuration. Argo CD continuously compares that desired state with the live Kubernetes state and reports differences. Depending on policy, it can apply approved changes and reconcile drift.

| Stage | Expected control | PM focus |
|---|---|---|
| **1. Change in Git** | A traceable change updates the chart, manifests, or environment values. Application image versions are immutable and traceable to a build. | Can we identify exactly what changed and which ML model, image, and configuration it deploys? |
| **2. Review / approval** | Pull request review, required checks, and environment-specific approval. Production changes use protected branches and least-privilege access. | Are the right reviewers involved? Is the change linked to a ticket, test evidence, and release owner? |
| **3. Argo CD sync** | Argo CD applies the Git-defined state manually or automatically according to environment policy. | Is sync authorized for this environment? Is auto-sync or prune enabled, and what is its blast radius? |
| **4. Verify status and service** | Review `Synced` / `OutOfSync` and `Healthy` / `Degraded`, rollout events, logs, readiness probes, API smoke tests, and ML-specific quality, latency, and error metrics. | A successful sync is not proof of a healthy service or acceptable model behavior. |
| **5. Release decision** | The release owner confirms acceptance criteria and monitoring; pause or roll back if health or ML metrics breach agreed thresholds. | Record the go/no-go decision, evidence, decision-maker, and rollback trigger. |

Manual edits directly in the cluster create **drift**: live state no longer matches Git, so the environment becomes harder to reproduce and audit. With self-heal enabled, Argo CD may overwrite the manual edit; with it disabled, the discrepancy can persist unnoticed.

The PM should review the Git diff, Argo CD status and history, operational signals, and a tested recovery pathвЂ”not just whether a deployment command completed.

## 2. Manual Deployment vs. GitOps

| Criterion | Manual Helm / kubectl deployment | GitOps with Argo CD |
|---|---|---|
| **Source of truth** | May be a command, local values file, runbook, or engineer's knowledge. | Version-controlled Git configuration defines the intended state. |
| **Who starts a change** | An engineer runs deployment commands. | A PR merge and/or authorized sync; automation follows policy. |
| **Change history** | May be split across terminal logs, tickets, and chat. | Git commits and PRs provide reviewable history; Argo CD provides sync and resource-status history. |
| **Drift detection** | Often depends on a person comparing live state with files. | Argo CD continuously detects differences from desired state. |
| **Rollback** | Re-run a previous command or chart version; steps may vary by operator. | Usually revert the Git change and reconcile; emergency recovery follows a documented procedure. |
| **Production risks** | Human error, inconsistent environments, weak auditability, undocumented changes. | A bad Git change can propagate quickly; incorrect auto-sync, prune, or permissions can amplify impact. |

**Conclusion:** Manual deployment can be acceptable for a prototype, a one-off sandbox, or a documented emergency when GitOps is unavailable. For a production ML service with repeated releases, multiple environments, audit needs, or several operators, GitOps is preferableвЂ”provided approvals, permissions, monitoring, and rollback are designed rather than assumed.

## 3. Environment Policies

| Environment | Auto-sync | Self-heal | Prune | Approval | Policy rationale |
|---|---|---|---|---|---|
| **Dev** | On | On | On only for disposable, reviewed resources | Developer / code-owner review and CI checks | Fast feedback is valuable. Keep test data and persistent resources protected. |
| **Staging** | Usually on after merge to the staging branch/path | On | Controlled; review deletions | PR review plus QA / ML owner acceptance | Production-like validation with less release friction; catch configuration and model issues. |
| **Production** | Off by default or tightly gated; enable only with explicit safeguards | Policy-based; define incident exceptions | Off or tightly restricted for critical resources | Protected PR, required technical review, release-owner approval | Limit blast radius and avoid automatic deletion or reconciliation during an incident. |

### Why these controls matter

- **Auto-sync** reduces manual steps but can rapidly apply a faulty merge.
- **Self-heal** restores Git's desired state after out-of-band edits, but can undo an emergency change. Responders should record any exception and reconcile it back into Git.
- **Prune** deletes live resources no longer declared in Git. A mistaken removal can delete workloads or supporting resources.

Production policy should explicitly identify protected resources, deletion review, and emergency handling. Exact settings should be agreed with the platform team based on service criticality.

## 4. Risk Register

Likelihood and impact below are initial qualitative estimates. The team should calibrate them using service criticality, traffic, data sensitivity, and incident history.

| Risk | Cause and consequence | Likelihood | Impact | Mitigation / owner |
|---|---|---:|---:|---|
| **Incorrect merge** | Weak review or wrong environment values allow a bad release to reach the cluster. | Medium | High | Protected branches, required reviewers and checks, small PRs. **Owner:** Tech Lead. |
| **Auto-sync applies a faulty change** | Automation syncs immediately, causing an outage or bad inference behavior. | Medium | High | Gate production changes, use canary / gradual rollout where feasible, add smoke tests and alerts. **Owner:** Platform + Service Owner. |
| **Prune deletes a critical resource** | A resource is removed from Git unintentionally and a service or data dependency is disrupted. | LowвЂ“Medium | High | Restrict prune in production, review deletions, protect persistent resources. **Owner:** Platform. |
| **Production drift from manual edits** | A hotfix is not committed; environments diverge or self-heal reverses the fix. | Medium | High | Emergency-change runbook; reconcile every hotfix into Git; alert on unexplained `OutOfSync`. **Owner:** On-call Engineer. |
| **Rollback is untested** | No known-good version or unclear authority prolongs recovery. | Medium | High | Document and rehearse rollback; retain known-good image, model, and configuration. **Owner:** Release Owner. |
| **No service owner** | Unclear accountability delays decisions and leaves alerts unresolved. | Medium | High | Name primary and backup owners and an escalation path before launch. **Owner:** PM. |
| **Argo CD permissions are too broad** | Shared or admin access enables unauthorized sync or configuration changes. | LowвЂ“Medium | High | Role-based access, least privilege, SSO, and audit logs. **Owner:** Platform / Security. |
| **Model quality regresses while infrastructure looks healthy** | Health checks pass while predictions degrade, harming product outcomes. | Medium | High | Track latency, errors, data/model-quality indicators, and model version. **Owner:** ML Owner. |

## 5. Reading Argo CD UI as a PM

| Status | Meaning | PM action |
|---|---|---|
| **Synced** | Live resources match the desired Git state. | Confirm alongside health and service-level checks; `Synced` alone does not mean the service works. |
| **OutOfSync** | Live state differs from the desired state in Git. | Ask whether this is expected, an in-progress change, a failed sync, or manual drift; check duration and policy. |
| **Healthy** | Argo CD's health assessment considers the resources healthy. | Verify smoke tests and ML/API metrics too; health checks do not guarantee model quality. |
| **Degraded** | One or more resources are unhealthy or a rollout failed. | Escalate to the service owner / on-call; assess user impact and rollback criteria immediately. |
| **Progressing** | A resource is still deploying or transitioning. | Check expected rollout time and events; escalate if it exceeds the agreed deployment window. |
| **Missing** | An expected resource is absent from the live cluster. | Determine whether it is not yet created, was deleted, or failed to apply; treat critical production resources as urgent. |

### When to observe, escalate, or declare an incident

- **Observe:** a known rollout is within its agreed window and service metrics remain normal.
- **Contact engineering:** `OutOfSync` is unexplained, `Progressing` is prolonged, or health is degraded.
- **Potential incident:** production users, availability, latency, prediction quality, or critical dependencies are affectedвЂ”or impact is unknown and time-sensitive.

Questions the PM should ask:

1. What changed?
2. Which commit, image, and model version are live?
3. Is this state expected?
4. What is the user impact?
5. Who is the incident commander / service owner?
6. What is the rollback trigger and expected recovery time?

## 6. Production Rollback and Incident Response

Scenario: after a merge into the GitOps repository, the production service becomes `Degraded`.

1. **Detect and declare.** Alerting, customer reports, or Argo CD identifies the issue. On-call acknowledges it and records the incident start time, affected service/environment, and initial impact.
2. **Triage.** The service owner / on-call checks Argo CD application status, sync history, resource events and logs, recent Git diff, API smoke tests, and ML service metrics. The PM coordinates impact assessment, communications, and decision loggingвЂ”not command execution.
3. **Decide.** The incident commander / technical lead and service owner decide whether to roll back, using predefined thresholds such as sustained errors, failed readiness, critical latency, or model-quality regression. The PM ensures the decision and rationale are recorded.
4. **Recover.** The preferred normal path is to **revert the faulty Git commit** (or submit a reviewed corrective commit) and let Argo CD reconcile. If that path is too slow or unavailable, use the documented emergency release procedure to restore a known-good version. Manual sync to a prior state is allowed only if explicitly supported and audited. Do not leave an untracked manual change as the final state.
5. **Verify.** Confirm Argo CD is `Synced` and `Healthy`, rollout is complete, smoke tests pass, and service / ML metrics return to agreed thresholds. Monitor for a defined observation window before closing the incident.
6. **Close and learn.** Maintain a timeline of detection, decisions, actions, and recovery. Communicate status, run a blameless post-incident review, assign owners and due dates for prevention actions, and update runbooks, tests, alerts, and GitOps policies.

## 7. Production-Readiness Checklist

Before approving GitOps delivery for production, confirm:

- [ ] The GitOps repository and each production application have a named owner and backup.
- [ ] Production changes require protected-branch review and required CI checks.
- [ ] Each release identifies an immutable application image, model artifact/version, and configuration revision.
- [ ] Dev, staging, and production have explicit configuration and access boundaries.
- [ ] Auto-sync, self-heal, and prune policies are documented for each environment.
- [ ] Production prune is disabled or tightly controlled; critical and persistent resources are protected.
- [ ] Argo CD roles follow least privilege; Sync permissions and emergency access are known.
- [ ] The team can explain `Synced` vs. `Healthy` and knows who responds to each failure state.
- [ ] Release acceptance includes smoke tests and ML-specific latency, error, and quality checks.
- [ ] Alerts and an incident-response / escalation path are tested.
- [ ] Rollback to a known-good image, model, and configuration is documented and has been rehearsed.
- [ ] Emergency manual changes must be logged and reconciled back into Git.
- [ ] Git PRs, approvals, sync events, and incident decisions provide an auditable trail.
- [ ] Production rollout has agreed go/no-go criteria, an observation window, and rollback triggers.
- [ ] The post-incident review process assigns measurable corrective actions and owners.

## PM Recommendation

Approve GitOps for production only when ownership, environment policies, review gates, observability, access control, and a rehearsed rollback path are demonstrated.

The minimum release decision is not **вЂњArgo CD synced.вЂќ** It is: **вЂњThe approved version is deployed, the application is healthy, ML/API acceptance signals are within limits, and the team can recover safely.вЂќ**