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

## State key prefix convention — every repo declares its own, nothing hardcoded in workflow YAML

**The rule**: a repo's Terraform state key prefix is declared **once**, in that repo's own config
file (`configs/orgs/<org>.tfvars` or equivalent) — never hardcoded into `.github/workflows/*.yaml`.
The workflow reads the prefix from config and passes it through to `terraform init -backend-config`.
Hardcoding a prefix into a reusable workflow defeats the entire point of it being reusable.

**The prefix shape** — two conventions, for two different classes of repo:

```
hcp/prd/platform/<repo-short-name>/<region>/terraform.tfstate      # management-account repos
workloads/<account-name>/<repo-short-name>/terraform.tfstate       # workloads-account repos
```

**Management-account repos** (`hcp/prd/platform/...`) — this convention is **already live and
consistently used today** by `aws-terraform-platform-aws-accounts`, `-aws-baselines`, and
`-aws-org` (confirmed by reading each repo's actual `backend.tf` directly — e.g. `aws-accounts`
uses key `hcp/prd/platform/aws-accounts/eu-west-1/terraform.tfstate`). Keep using it for any new
management-account repo — don't invent a third scheme. The **one exception** is the seed repo
itself, whose state key is the pre-existing `myorg/aws/control-plane/terraform.tfstate` — a rename
to match this convention is tracked as follow-up work (requires a real backend/state migration,
not just a config edit), not yet done.

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
| `aws-terraform-platform-seed` | management | `myorg/aws/control-plane/terraform.tfstate` | live, pre-dates convention, rename tracked as follow-up |
| `personal-ai-cloud` | hcp-craighoad-prod | `workloads/craighoad.com/personal-ai-cloud/terraform.tfstate` | **target** — currently `hcp/prd/personal-ai-cloud/*` (ad-hoc, in the seed repo's cross-account bucket policy) |
| `website-static-html-craighoad.com` | hcp-craighoad-prod | `workloads/craighoad.com/website/terraform.tfstate` | target |
| `aws-terraform-solutions-craighoad-blog` | hcp-craighoad-prod | `workloads/craighoad.com/blog/terraform.tfstate` | target |
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
