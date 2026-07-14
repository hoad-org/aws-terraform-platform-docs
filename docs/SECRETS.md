# Secrets — AWS Secrets Manager, not GitHub Environment secrets

## The target model

No repo's GitHub Actions workflow should hold `SEED_ROLE_ARN`, `TF_STATE_BUCKET`,
`PLAN_PASSPHRASE`, or any other infrastructure secret as a **GitHub Environment secret**. Instead:

1. The workflow authenticates via OIDC directly into that repo's **plan** role (see
   [CICD_ROLES.md](CICD_ROLES.md)) — this requires no secret at all, just the role ARN, which is
   derived mechanically from a naming formula, not stored anywhere.
2. Once authenticated, the workflow calls `hoad-org/github-automation`'s `get-aws-secrets`
   composite action, which fetches everything else from **AWS Secrets Manager**.

## `get-aws-secrets` — exact contract

**Inputs**: `org` (required — e.g. `hcp`), `aws_region` (default `eu-west-1`), `aws_account_id`
(required — the account this org's OIDC roles live in; never hardcode one, since different orgs
can live in different accounts), `require_github_token` (default `"true"` — set `"false"` for
repos with no private-Terraform-module dependency, so they don't need to provision a PAT they'll
never use).

**What it does**:
1. Derives the plan-role ARN: `arn:{partition}:iam::{aws_account_id}:role/{org}-{region_short}-platform-github-oidc-tf-plan-role`
2. Authenticates via `aws-actions/configure-aws-credentials` against that role
3. Fetches, from Secrets Manager under `github/{org}/*`:

| Secret name | Purpose |
|---|---|
| `github/{org}/tf-state-bucket` | State bucket name |
| `github/{org}/tf-logs-bucket` | Logs bucket name |
| `github/{org}/kms-key-id-alias` | KMS alias for state encryption |
| `github/{org}/oidc-plan-role-arn` | This org's plan role ARN (redundant with the derived value, but explicit) |
| `github/{org}/oidc-apply-role-arn` | This org's apply role ARN |
| `github/{org}/plan-passphrase` | Legacy — plan-encryption passphrase, see "Known future improvement" below |

Plus one **non-org-scoped, shared** secret: `github/github-automation/pat-token` — a GitHub PAT
for private Terraform module access, fetched only if `require_github_token != "false"`.

**Outputs**: `tf_state_bucket`, `tf_logs_bucket`, `kms_key_id_alias`, `oidc_plan_role_arn`,
`oidc_apply_role_arn`, `plan_passphrase`, `github_token` — all masked in logs.

**Fails loudly, not silently**: every one of the 6 org-scoped secrets is validated non-empty and
the action `exit 1`s with a specific message naming exactly which secret failed to resolve, rather
than letting a missing secret surface two steps later as a confusing generic AWS error.

## Who writes these secrets

**Each account's own Terraform writes its own `github/{org}/*` secrets** — there is no central
"secrets bootstrap" step run by hand. The seed repo's `seed-terraform/secrets.tf` writes the
management account's secrets; a workloads-account repo (if it owns its own one-hop OIDC roles,
see [CICD_ROLES.md](CICD_ROLES.md)) writes its own. `random_password.plan_passphrase` generates
the passphrase; everything else comes from `terraform-aws-modules` outputs already computed
elsewhere in that repo's Terraform.

## Known future improvement: retire `PLAN_PASSPHRASE`

The reference architecture this design was adapted from moved (in its own v3→v4 evolution) from
AES-256-encrypted plan artifacts (requiring a shared passphrase secret, prone to "bad decrypt"
errors on environment drift) to plaintext plan artifacts + SHA-256 checksum binding — reasoning
that GitHub Actions' own artifact storage already provides at-rest encryption, and an
approver being unable to read what they're approving is a worse problem than the marginal
confidentiality loss. **Not yet implemented here** — `get-aws-secrets` and `secrets.tf` both
still carry `plan-passphrase` today. Tracked as follow-up work, not blocking.

## What replaces `tools/create-gh-env.sh`

That script (in the seed repo) is the *old* mechanism — it pushes `SEED_ROLE_ARN`,
`TF_STATE_BUCKET`, etc. as literal GitHub Environment secrets on a target repo, generating
`PLAN_PASSPHRASE` locally via `openssl rand`. **It must not be run for any repo that has migrated
to the `get-aws-secrets` model** — running it re-introduces exactly the GitHub-secrets sprawl this
whole redesign exists to eliminate. See [REPOS.md](REPOS.md) for which repos have migrated.
