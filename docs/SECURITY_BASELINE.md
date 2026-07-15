# Security Baseline — maximum hygiene achievable at $0

This is the security posture this landing zone actually runs, given the [$0 budget](BUDGET.md).
Everything below is free. Nothing below is a compromise for cost reasons — it's genuinely the
right baseline regardless of budget. What's *not* here (GuardDuty, Config, Security Hub, Macie) is
absent because those cost money, not because they were overlooked — see the end of this file and
[SECURITY_ROADMAP.md](SECURITY_ROADMAP.md).

## On, by design

- **CloudTrail (management events)** — free, org-wide, always on. Data events (S3 object-level,
  Lambda invocations) are NOT enabled — those cost per-event at any real volume.
- **IAM Access Analyzer** — free, flags unintended external/cross-account access.
- **MFA on human/root identities** — free, always required for any console access.
- **Least-privilege IAM** — the split plan/apply/module-skills OIDC role model (see
  [CICD_ROLES.md](CICD_ROLES.md)) exists specifically for this: a compromised plan-phase token
  (runs unattended, on every push) can never mutate real infrastructure, full stop, not just by
  convention.
- **S3 block-public-access** — on for every bucket, always, no exceptions.
- **KMS encryption at rest** — the shared state bucket and (where applicable) workload data stores
  use KMS, not plaintext.
- **Native S3 state locking** (`use_lockfile = true`) — prevents concurrent-apply corruption,
  free (replaces the older DynamoDB-lock-table pattern, which would cost money at any real scale).
- **AWS Organizations SCPs** — free, the primary technical enforcement layer for the cost
  guardrail in [BUDGET.md](BUDGET.md), and available for genuine security guardrails too (e.g. deny
  root-user API calls, deny leaving the org, deny disabling CloudTrail) — worth expanding over
  time, at zero cost.
- **`main`-only branch discipline, PR-required for every change** — see [PROCESS.md](PROCESS.md).
  Free, and it's the actual review mechanism standing between "someone/something proposes a
  change" and "that change touches real infrastructure."

## Deliberately off — cost, not oversight

- **GuardDuty** — 30-day free trial, then a real, usage-based cost. Off.
- **AWS Config** — no free tier at all; cost per configuration item recorded + per rule
  evaluation. Off.
- **Security Hub** — aggregates findings from GuardDuty/Config/Inspector, all of which cost money
  themselves. Off (nothing to aggregate yet anyway).
- **Macie** — per-object/per-GB scanning cost. Off.
- **VPC Flow Logs to CloudWatch Logs** — CloudWatch Logs ingestion/storage costs at volume. Off
  unless actively debugging (turn off again after).
- **X-Ray** — free tier is generous but not infinite; not enabled by default, opt in per-repo if a
  specific debugging need justifies it.

Every one of these is denied at the IAM layer for the Workloads OU by the SCP in
[BUDGET.md](BUDGET.md) — specifically so a future Claude session can't "helpfully" turn one on
without it being a deliberate, reviewed decision.

## What a real incident response looks like at $0

CloudTrail management events + IAM Access Analyzer + the OIDC trust-subject model (every action is
attributable to a specific repo/workflow run, not a shared long-lived credential) already give
enough signal to reconstruct "what happened and from where" for anything short of a
sophisticated, sustained intrusion. That's the realistic bar at this budget and this threat model
(personal infrastructure, not a company handling other people's regulated data) — see
[SECURITY_ROADMAP.md](SECURITY_ROADMAP.md) for what changes once that's no longer true.
