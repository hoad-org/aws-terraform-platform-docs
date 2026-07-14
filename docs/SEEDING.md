# Seeding — bootstrapping a brand-new AWS Organization

## The chicken-and-egg

The seed repo's own Terraform creates the OIDC provider, the plan/apply/module-skills roles, the
shared state bucket, and the KMS key that everything else (including the seed repo's own CI)
depends on. Nothing can authenticate via OIDC into a role that doesn't exist yet — so the very
first apply of the seed repo **must** run via real, human/agent-held AWS credentials (SSO,
`cloudctl`, or equivalent), never via CI. Every subsequent change can go through the normal
plan→approve→apply pipeline once the roles exist.

## One-time bootstrap sequence

1. **Authenticate with real credentials** against the management account.
2. `terraform init` against the real shared state backend (the state bucket/key already exist
   by this point if this is truly the first-ever apply — otherwise `terraform init -backend=false`
   for a completely fresh Organization, then a first apply that also creates the backend itself).
3. **Verify AWS Organizations prerequisites are enabled** before attempting anything StackSet-related:
   - `aws organizations enable-aws-service-access --service-principal=member.org.stacksets.cloudformation.amazonaws.com`
   - `aws cloudformation activate-organizations-access`
   - Both usually take a few minutes to propagate; if `CreateStackSet` fails with
     `"You must enable organizations access..."` immediately after enabling either, wait and retry
     rather than assuming the setting didn't take.
4. `terraform plan` — review carefully. On a genuinely fresh org this should be almost entirely
   `+ create`; on an existing-but-drifted org (see "Reconciling drift" below) expect some
   `~ update in-place` for resources that exist in AWS but were never imported into this
   particular state file.
5. `terraform apply`.
6. Manually trigger the seed repo's own `.github/workflows/deploy.yaml` once
   (`gh workflow run deploy.yaml`) to confirm the newly-created OIDC roles actually work
   end-to-end — this is the only way to validate the bootstrap actually succeeded, since a
   successful `terraform apply` only proves the roles exist, not that GitHub Actions can use them.
7. Onboard other `aws-terraform-*` repos one at a time: add their subjects to the relevant
   `github_oidc_*_subjects` variable, apply, confirm their own CI works, then move to the next.
   Never bulk-migrate every repo in one change — each migration is independently verifiable and
   independently revertible.

## Reconciling drift (a real, recurring problem in this landing zone)

More than once, real AWS resources have existed for months without ever being correctly tracked
in this Terraform's own state — not because they were created outside Terraform, but because an
earlier apply attempt silently failed partway through and was never re-run to completion. Symptoms:
- `terraform plan` shows `will be created` for a resource (e.g. an IAM policy) that
  `aws iam list-policies` shows already exists → `terraform import` it, don't let apply try to
  `CreatePolicy` a second time (it will fail with `EntityAlreadyExists`).
- An `aws_iam_role_policy_attachment` shows as `will be created` even though the attachment is
  already live — this one is usually safe to just apply, since `iam:AttachRolePolicy` is
  idempotent (re-attaching an already-attached policy succeeds silently), unlike `CreatePolicy`.
- Before trusting any `expected_account_id`, `organization_id`, or OU ID committed in a
  `configs/orgs/*.tfvars` file, **verify it against a live AWS API call**. This landing zone has
  had fabricated organization/OU IDs sit undetected in multiple repos' config files for an
  extended period — see [ARCHITECTURE.md](ARCHITECTURE.md)'s "past, now-corrected mistake" note.

## Onboarding a new AWS account (workloads or otherwise)

1. `aws-terraform-platform-aws-accounts` vends the account (`aws_organizations_account`), placing
   it in the correct OU (see the account tree in [ARCHITECTURE.md](ARCHITECTURE.md)).
2. The seed repo's CloudFormation StackSet (targeting that account's OU) automatically deploys
   the member CICD role into it — no separate step required, since `auto_deployment.enabled = true`.
3. The account gets its own state-key prefix under `workloads/{account-name}/` — see
   [BACKEND.md](BACKEND.md).
4. `aws-terraform-platform-aws-baselines` applies "Day 1" baseline config (Config, CloudTrail,
   guardrails) into the new account.
5. Application repos (e.g. a new workloads repo) declare their `expected_account_id` and state
   prefix in their own config file, then follow the [SECRETS.md](SECRETS.md) model — no manual
   GitHub secret setup required.
