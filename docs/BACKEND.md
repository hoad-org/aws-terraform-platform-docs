# Terraform State Backend

## Shared resources (management account, `395101865577`)

| Resource | Name |
|---|---|
| State bucket | `hcp-cmc-euw1-platform-tfstate-prd` |
| Logs bucket | `hcp-cmc-euw1-platform-logs-prd` |
| KMS key alias | `alias/hcp-cmc-euw1-platform-tfstate` |
| Lock mechanism | Native S3 conditional-writes locking (`use_lockfile = true` in `backend "s3" {}`) — **no DynamoDB lock table**, this is the modern Terraform ≥1.11 + AWS provider ≥6.x approach |

One bucket, one KMS key, shared by every repo in the estate — management-account and workloads
alike. Workloads-account repos reach it via a **cross-account bucket policy + KMS key grant**
naming their specific plan/apply role ARNs (see [CICD_ROLES.md](CICD_ROLES.md)).

## State key prefix convention — one real exception, not aspirational

**The rule for most repos**: a repo's Terraform state key is passed explicitly to
`hoad-org/github-automation`'s `reusable-tf-plan-encrypt.yaml` / `reusable-tf-apply-decrypt.yaml`
via the `state_key` input on the `with:` block of the calling job in `.github/workflows/*.yaml`.
Never rely on the default.

**Real bug, found and fixed 2026-07-15**: until `hoad-org/github-automation#6`, neither reusable
workflow had a `state_key` input at all — both hardcoded the seed repo's own key
(`${ORG}/${PARTITION}/control-plane/terraform.tfstate`) with no override, so every caller other
than the seed itself would silently read/write the seed's own control-plane state file. This was
caught via a real 403 Forbidden on `aws-terraform-solutions-craighoad-blog`'s first Terraform
Plan run — that repo's scoped IAM policy correctly rejected access to a state key outside its own
granted prefix, which is what surfaced the bug instead of it silently corrupting the seed's state.
A caller with a broader IAM policy would not have been protected. Fixed by adding a `state_key`
input to both workflows (default = the seed's literal key, kept only for the seed's own backward
compatibility) — every non-seed caller now passes its own real key explicitly in its `with:`
block. See `personal-ai-cloud`'s and `aws-terraform-solutions-craighoad-blog`'s `deploy.yaml` for
the live pattern.

**The prefix shape** — two conventions, for two different classes of repo:

```
hcp/prd/platform/<repo-short-name>/<region>/terraform.tfstate      # management-account repos
workloads/<account-name>/<repo-short-name>/terraform.tfstate       # workloads-account repos
```

**Management-account repos** (`hcp/prd/platform/...`) — this convention is **already live and
consistently used today** by `aws-terraform-platform-aws-accounts`, `-aws-baselines`, and
`-aws-org` (confirmed by reading each repo's actual `backend.tf` directly — e.g. `aws-accounts`
uses key `hcp/prd/platform/aws-accounts/eu-west-1/terraform.tfstate`). Keep using it for any new
management-account repo — don't invent a third scheme.

**The real exception: the seed repo itself does NOT follow this convention, and can't without a
shared-workflow change.** `hoad-org/github-automation`'s `reusable-tf-plan-encrypt.yaml` hardcodes
the seed's own state key formula directly in its `terraform init -backend-config` step:
`${ORG}/${PARTITION}/control-plane/terraform.tfstate` (not read from any per-repo config value —
a genuine, deliberate exception to the "declared once in config" rule above, scoped to the one
repo that IS the shared backend). Originally this computed to `myorg/aws/control-plane/...` (a
stale literal "myorg" segment predating the org's real short name being set to "hcp"), which
didn't match `${ORG}` ("hcp") at all — meaning the first real CI run would have hit an empty state
and tried to recreate the OIDC provider, all 3 roles, the KMS key, and both S3 buckets from
scratch. Migrated for real (`terraform init -migrate-state -force-copy`) to
`hcp/aws/control-plane/terraform.tfstate`, matching what the reusable workflow actually computes.
Don't "fix" this to match the `hcp/prd/platform/seed/...` convention below without first changing
the hardcoded key in the reusable workflow — the two must always match exactly.

**Workloads-account repos** (`workloads/<account-name>/...`) — this convention is **new, not yet
implemented anywhere**, `<account-name>` is the account's friendly/domain name (`craighoad.com`,
`terrorgems`, `qa`) — not the raw AWS account ID, not the repo name. `<repo-short-name>`
disambiguates when multiple repos deploy into the same workloads account (e.g. `personal-ai-cloud`
and `website` both deploy into `hcp-craighoad-prod`).

**Examples**:

| Repo | Target account | State key | Status |
|---|---|---|---|
| `aws-terraform-platform-aws-accounts` | management | `hcp/prd/platform/aws-accounts/eu-west-1/terraform.tfstate` | live |
| `aws-terraform-platform-aws-baselines` | management | `hcp/prd/platform/aws-baselines/eu-west-1/terraform.tfstate` | live |
| `aws-terraform-platform-aws-org` | management | `hcp/prd/platform/aws-org/eu-west-1/terraform.tfstate` | live |
| `aws-terraform-platform-seed` | management | `hcp/aws/control-plane/terraform.tfstate` | live — the one repo allowed to rely on the reusable workflow's default, not this convention (see above), migrated from stale `myorg/...` |
| `personal-ai-cloud` | craighoad-prod (`624426145233`) | `hcp/prd/personal-ai-cloud/terraform.tfstate` | live — passed explicitly via `state_key` in `deploy-portal.yaml` (ad-hoc naming, doesn't match the `workloads/...` convention below; not yet renamed, low priority since no real CI run has used it yet) |
| `aws-terraform-solutions-craighoad-blog` | craighoad-prod (`624426145233`) | `hcp/prd/craighoad-website/craighoad-blog/eu-west-1/terraform.tfstate` | live — passed explicitly via `state_key` in `deploy.yaml`/`pr-validate.yml`; real infra is currently still deployed in `hcp-terrorgems-prod`, a known separate mismatch (see `REPOS.md`) |
| `website-static-html-craighoad.com` | hcp-craighoad-prod | `workloads/craighoad.com/website/terraform.tfstate` | target |
| `aws-terraform-solutions-terrorgem` | hcp-terrorgems-prod | `workloads/terrorgems/terrorgem/terraform.tfstate` | target |

See [REPOS.md](REPOS.md) for the full per-repo migration checklist.

## A live inconsistency to be aware of, not yet resolved

Multiple management-account repos (`aws-accounts`, `aws-baselines`, `aws-org`) have a **second,
conflicting** backend-ish config declared in `environments.yml`/`.tfctl.yml` — a bucket pattern
`infra-tfstate-hoad-org-seed` that doesn't match the real, live `backend.tf` bucket
(`hcp-cmc-euw1-platform-tfstate-prd`) at all, plus disagreeing region values (`eu-west-2` in one
file, `us-east-1` in another, vs the real `eu-west-1`). This looks like dead/legacy config from
an earlier design iteration, not active drift — the real `backend.tf` files are consistent and
appear to be what's actually applied — but it hasn't been confirmed dead or cleaned up. Don't
copy values from `environments.yml`/`.tfctl.yml` into a new repo's config without cross-checking
against that repo's actual `backend.tf` first.

## Bootstrap ordering constraint

The state bucket/KMS key are themselves managed by the seed repo's own Terraform, which uses
this same backend — a real chicken-and-egg the seed repo resolves by being the one exception
allowed to bootstrap via local human/agent credentials before any CI exists for it. See
[SEEDING.md](SEEDING.md).
