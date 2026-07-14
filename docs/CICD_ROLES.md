# CI/CD OIDC Roles

## The split: plan / apply / module-skills

Every repo's GitHub Actions pipeline authenticates via OIDC into one of three narrow roles,
never one broad combined role:

| Role | Purpose | Trust condition | Permissions |
|---|---|---|---|
| **plan** | `terraform plan` — read-only | `repo:{owner}/{repo}:ref:refs/heads/main` (no approval gate — runs on every push) | AWS-managed `ReadOnlyAccess` + a narrow custom policy: state-lock S3 object R/W (not real state writes — just the native S3 lock file), `kms:Decrypt` on the state key, `sts:AssumeRole` on the member CICD role pattern only, audit-log S3 write to one prefix, `sts:GetCallerIdentity`, and `secretsmanager:GetSecretValue`/`DescribeSecret` scoped to `secret:github/{org}/*` (AWS's `ReadOnlyAccess` deliberately excludes secret *values*) |
| **apply** | `terraform apply` — write | `repo:{owner}/{repo}:environment:{env}-approve` (a GitHub Environment with required reviewers — the environment itself is the approval gate) | Full write access to the specific AWS services that repo's Terraform manages — scoped per-repo, not blanket `*:*` |
| **module-skills** (optional) | Non-infra pipelines (test runners, Claude skills, analytics) | `repo:{owner}/{repo}:environment:*` or wildcard subjects, only created when actually needed | Audit-log write + KMS for that write + caller-identity only — zero state/IAM access, smallest possible blast radius |

**Why a role split, not one role for everything**: separation of duties. A compromised or
misconfigured plan-phase token (which runs on every push, unattended) can never mutate real
infrastructure — it's mathematically incapable of it, not just conventionally discouraged. Only
a token from the gated `-approve` environment, which requires a human reviewer to click approve,
can assume the apply role.

## Naming formula (load-bearing — must match exactly)

```
{org}-{region_short}-{system}-github-oidc-tf-plan-role
{org}-{region_short}-{system}-github-oidc-tf-apply-role
{org}-{region_short}-{system}-github-oidc-module-skills-role
```

No `partition_short` segment — deliberately different from the general resource-naming formula
in [ARCHITECTURE.md](ARCHITECTURE.md). `hoad-org/github-automation`'s `get-aws-secrets` action
derives the plan-role ARN mechanically from exactly this formula
(`{org}-{region_short}-platform-github-oidc-tf-plan-role`, `system` hardcoded as `platform` in
that action today). If you rename the role, `get-aws-secrets` breaks silently until the action is
updated to match.

## Two deployment models — management-account roles vs. workloads-account roles

**Management-account roles** (created by the seed repo's own `seed-terraform/main.tf`): used by
repos whose Terraform targets the management account directly — the seed repo itself,
`aws-accounts`, `aws-baselines`, `aws-org`.

**Workloads-account roles**: each workloads account gets its OWN plan/apply/module-skills roles,
created either by (a) the seed repo's CloudFormation StackSet deploying a **member CICD role**
into every workloads account (see below), assumed cross-account from the management-account roles
— the "two-hop" model — or (b) a workloads-account repo creating its own roles directly in that
account via a one-hop OIDC provider — used when a repo doesn't need the shared state backend's
cross-account complexity. Prefer (a) for anything that should be manageable centrally; (b) is an
escape hatch, not the default.

## The member CICD role (StackSet-deployed)

A CloudFormation StackSet (`SERVICE_MANAGED` permission model, `auto_deployment.enabled = true`)
deploys a single IAM role — `{name_prefix}-cicd-role` — into **every** account in the target OUs
(`Workloads/Prod`, `Workloads/NonProd`, plus `Management`/`Security`/`Infrastructure` as needed).
This is what lets the landing zone "deploy to mgmt alone, or any member account alone, or all of
them" — the StackSet handles fan-out; each individual repo's Terraform only ever needs to know
its own target account.

**A real bug found and fixed in this template**: the role's `AssumeRolePolicyDocument` originally
trusted only `Principal: {Service: "cloudformation.amazonaws.com"}` — meaning nothing except the
CloudFormation service itself could ever assume it, defeating its entire purpose as a cross-account
CI/CD execution role. Fixed to trust the calling management-account OIDC role's ARN
(`AWS: {management_role_arn}`) instead, with `aws:PrincipalOrgID` as defense-in-depth.

**Prerequisites for the StackSet to deploy at all** (both required, commonly conflated as one):
1. `aws organizations enable-aws-service-access --service-principal=member.org.stacksets.cloudformation.amazonaws.com`
2. `aws cloudformation activate-organizations-access` — a **separate** CloudFormation-specific
   switch. Both must show enabled (`aws cloudformation describe-organizations-access`) before
   `CreateStackSet` with `permission_model = SERVICE_MANAGED` will succeed.

**Target OU IDs must be real, live-queried values** — see the corrected list in
[ARCHITECTURE.md](ARCHITECTURE.md). A previous attempt used fabricated OU IDs that didn't exist
in the Organization at all, which fails with `Target organizational unit could not be found`
rather than any clearer error.
