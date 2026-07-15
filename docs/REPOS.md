# Repo Inventory

Every repo touching AWS infrastructure, its purpose, its current CI/CD model, and its migration
status toward the target design in [SECRETS.md](SECRETS.md), [CICD_ROLES.md](CICD_ROLES.md), and
[BACKEND.md](BACKEND.md). Findings below come from direct inspection (workflow YAML, backend
config, `gh run list` CI history, and full-repository BeyondTrust-content greps) during the audit
that produced this documentation — not from assumption. Both `rhyscraig` and `hoad-org` GitHub
orgs are covered — **`rhyscraig` was renamed/repos progressively transferred to `hoad-org`**, but
this migration is incomplete and inconsistent: some repos still live under `rhyscraig` with no
`hoad-org` counterpart, some redirect, some have been transferred outright. Treat each repo
individually — don't assume a blanket migration.

## The central architectural fact

**`hoad-org/aws-terraform-platform-aws-workflows` is a separate, older, less mature
reusable-workflow repo than `hoad-org/github-automation`** — not an earlier name for the same
thing. Different design entirely: GitHub Environment secrets (not Secrets Manager), no
`get-aws-secrets`-style OIDC-derivation, hardcoded/duplicated OPA policies, broken self-references
(checks out its own stale `rhyscraig/` name; `pip install`s from `rhyscraig/tfctl.git` and
`rhyscraig/awsctl.git`, neither of which exist under those names — the real repos are
`aws-python-platform-tfctl` and `awsctl-platform-aws`).

**Every one of the following repos currently consumes the older `aws-workflows` repo, either via
its `rhyscraig/` alias or the `hoad-org/` name directly**: `aws-accounts`, `aws-baselines`,
`aws-org`, `aws-terraform-platform-aws-templates`, `aws-terraform-platform-aws-modules`,
`aws-terraform-solutions-terrorgems-platform`, `aws-terraform-solutions-websites`,
`personal-ai-cloud`. **Zero repos consume `github-automation` yet** except the seed repo's own new
`deploy.yaml` (this session). **Migrating every caller from `aws-workflows` to `github-automation`
is the actual redesign work — not a green-field build.**

## Summary table

| Repo | Org | Naming OK | Secrets model | State backend (real) | CI health | BT content |
|---|---|---|---|---|---|---|
| `aws-terraform-platform-seed` | rhyscraig | ✅ | `github-automation` (new, this session) | `myorg/aws/control-plane/terraform.tfstate` | new workflow, untriggered | **Yes** — `configs/orgs/{bt-avm,bt-dev,fdr-cmc,fdr-gvc}.tfvars` (deleted this session, PR open), README/AI.md still reference BT clone URLs (not yet fixed) |
| `aws-terraform-platform-docs` | rhyscraig | n/a | n/a | n/a | n/a | this repo |
| `aws-terraform-platform-aws-accounts` | hoad-org | ✅ | old `aws-workflows` + `AWS_CI_ROLE_ARN`/`GH_PAT` | `hcp/prd/platform/aws-accounts/eu-west-1/terraform.tfstate` | drift-check red (3+ failures) | **git history only** — `aws-avm-{audits,security,shared-svcs}@beyondtrust.com`, author `choadbt <choad@beyondtrust.com>` |
| `aws-terraform-platform-aws-baselines` | rhyscraig | ✅ | old `aws-workflows` + `AWS_CI_ROLE_ARN`/`GH_PAT` | `hcp/prd/platform/aws-baselines/eu-west-1/terraform.tfstate` (second dead backend block confirmed and removed — PR #5, ADR-0001) | drift_check red (5/5 failures) | none in current tree |
| `aws-terraform-platform-aws-org` | hoad-org | ✅ | old `aws-workflows` + `AWS_OIDC_ROLE_ARN`/`GH_PAT` | `hcp/prd/platform/aws-org/eu-west-1/terraform.tfstate` | drift-check red (5/5 failures, every 4h) | **git history only** — author `choadbt <choad@beyondtrust.com>` |
| `aws-terraform-platform-aws-templates` | rhyscraig | ✅ | old `aws-workflows` + `SEED_ROLE_ARN`/`TF_STATE_BUCKET`/etc | via `-backend-config`; `.tfctl.yml` cites a **different management account** (`235494790978`) than everywhere else (`395101865577`) — unresolved discrepancy | no real deploy runs in recent history | ambiguous — `.tfctl.yml` bucket pattern `bt-terraform-remote-state-{region}` |
| `aws-terraform-platform-aws-modules` | rhyscraig | ✅ | old `aws-workflows`, no deploy (pure module catalog) | none (no backend anywhere — correct, it's a module registry) | green (Dependabot only) | none |
| `aws-terraform-platform-aws-workflows` | hoad-org | ✅ (but see "central architectural fact" above) | n/a — this **is** the older secrets/workflow provider | n/a | n/a | none directly, but see "Adjacent tooling" below |
| `aws-terraform-solutions-terrorgems-platform` (renamed from `-terrorgem`) | rhyscraig | ✅ | old `aws-workflows`, many GH secrets incl. app secrets (`JWT_SECRET`, `TMDB_API_KEY`) mixed with infra secrets | root `backend.tf` orphaned/stale; real backends per-environment under `infra/aws/environments/*/backend.tf`; `scripts/deploy.sh` hardcodes a bucket/region that **doesn't match** the CI workflow's region | 2 real failures (Trivy findings, workflow parse error), otherwise green | none |
| `aws-terraform-solutions-websites` | rhyscraig | ✅ | old `aws-workflows` + `SEED_ROLE_ARN` etc | `backend.hcl` — **yet another distinct bucket-naming convention** (`prd-<account>-eu-west-2-tf-state-bucket`) | **5/5 failures** — blocked by a GitHub **billing lockout**, unresolved ~2.5 months, unrelated to code | none |
| `aws-terraform-solutions-craighoad-blog` | hoad-org (redirects from rhyscraig) | ✅ | inline `SEED_ROLE_ARN` only, no reusable workflow calls at all | `terraform/backend.tf` (hardcoded) | **never actually executed** except Dependabot | none |
| `craighoad-portfolio-website` (renamed from `website-static-html-craighoad.com`) | hoad-org | ❌ not `aws-terraform-*` prefixed | unknown — not independently re-audited this pass beyond OIDC trust | unknown | unknown | not re-confirmed |
| `personal-ai-cloud` | hoad-org | ❌ not `aws-terraform-*` prefixed (defensible exception — app repo with embedded infra) | old `aws-workflows` (`hoad-org/` direct, not the `rhyscraig/` alias) — **not yet migrated to `github-automation` despite that being built this session for it** | `hcp/prd/personal-ai-cloud/*` (target: `workloads/craighoad.com/personal-ai-cloud/`) | 5/5 failures (all within 24h — likely the now-fixed OIDC subject issue, unconfirmed re-run) | none |
| `github-automation` | hoad-org | n/a — shared workflow hub, correctly unprefixed | is the target model — AWS Secrets Manager / Azure Key Vault, zero GH secrets | n/a | n/a (pure `workflow_call` library) | none |

## Adjacent tooling with real, unresolved BeyondTrust content — flag prominently

These aren't `aws-terraform-*` repos, but they're actively used to operate this landing zone
(including by Claude, this session, via `cloudctl`) and still carry real BeyondTrust content:

- **`hoad-org/cloudctl`** — the credential-vending tool used throughout this session to get real
  AWS access. Its own README says **"internal use by BeyondTrust Engineering only"**, its origin
  remote references `BT-IT-Infrastructure-CloudOps/aws-terraform-infra-cloudops-cloudctl`, and its
  own usage examples use `bt-avm` as the example org name throughout. This needs a real decision:
  is this Craig's own tool (in which case it needs a full rebrand/decouple from BT), or is it
  something that should stop being used for personal infrastructure entirely?
- **`hoad-org/awsctl-platform-aws`** — same pattern: README says "internal use by BeyondTrust
  Engineering only," installs via `pipx install "git+https://github.com/BT-IT-Infrastructure-CloudOps/awsctl.git@v2.8.1"`.
- **`rhyscraig/aws-python-platform-tfctl`** — README says it "Fetches secrets (AWS Secrets
  Manager, BeyondTrust)" and documents installing from
  `BT-IT-Infrastructure-CloudOps/tfctl.git`. Referenced (and broken — wrong repo name) from CI
  workflows in `aws-org` and the old `aws-workflows` repo.

**None of this has been assessed for whether it's safe to keep using** — flagging here rather than
making a unilateral call, since "is it OK to keep running a tool whose own README claims it's for
BeyondTrust-internal use only" is a real, non-technical judgment call.

## Stale OIDC trust subjects — fixed

The seed repo's `github_oidc_subjects` list (the legacy combined-role trust list) had two entries
referencing renamed/moved repos that would never match a real token again — removed:
- `repo:rhyscraig/aws-terraform-solutions-terrorgem:*` — the real repo is
  `aws-terraform-solutions-terrorgems-platform` (renamed).
- `repo:rhyscraig/website-static-html-craighoad.com:*` — the real repo is
  `hoad-org/craighoad-portfolio-website` (renamed **and** transferred org, already trusted
  separately under its real name).

These never granted excess access (a stale subject just never matches) — flagged here only so a
future session doesn't waste time re-discovering that those old names are
still meaningful.

## Naming convention violations

- `craighoad-portfolio-website` — not `aws-terraform-*` prefixed, despite provisioning real AWS
  infrastructure. Either rename or explicitly document as an accepted exception.
- `personal-ai-cloud` — same issue, more defensible (app repo with embedded infra under
  `infra/aws/`) — should still be an explicit, documented exception.
- `aws-terraform-platform-docs` (this repo) and `aws-terraform-platform-seed` are correctly
  prefixed but are non-standard in their own way: `docs` isn't Terraform/AWS infra itself (by
  design), and `seed` had no working CI/CD pipeline at all until this session.

## Currently-broken CI, ranked by urgency

1. **`aws-org`** — every 4-hour drift-check cron failing, 100% failure rate.
2. **`aws-baselines`** — daily drift-check failing 5/5 consecutive days.
3. **`aws-accounts`** — drift-check failing (3/5 recent).
4. **`personal-ai-cloud`** — deploy failing 5/5 in the last 24h, likely the now-fixed OIDC subject
   gap, unconfirmed by re-run.
5. **`aws-terraform-solutions-websites`** — fully blocked by a GitHub billing lockout, unrelated to
   code, unresolved ~2.5 months.
6. **`aws-terraform-solutions-craighoad-blog`** — has never executed even once outside Dependabot.

All the `drift-check`/`drift_check` failures across `aws-org`/`aws-baselines`/`aws-accounts`
complete in **0 seconds** — consistent with a fast pre-flight failure (auth/OIDC/secrets
resolution), not real infrastructure drift. Not yet root-caused.

## Cross-repo backend/region inconsistencies (see [BACKEND.md](BACKEND.md) for the canonical target)

At least **four distinct state-bucket-naming conventions** coexist: `hcp-cmc-euw1-platform-tfstate-prd`
(the real, live one, used consistently by `aws-accounts`/`aws-baselines`/`aws-org`),
`infra-tfstate-hoad-org-seed` (referenced in `environments.yml`/`.tfctl.yml` in those same repos,
but not matching their actual `backend.tf` — looks like dead/legacy config, not confirmed dead),
`bt-terraform-remote-state-{region}` (`aws-terraform-platform-aws-templates`'s `.tfctl.yml`), and
`prd-<account>-eu-west-2-tf-state-bucket` (`aws-terraform-solutions-websites`'s `backend.hcl`).
Region values disagree within single repos too (`eu-west-1` vs `eu-west-2` vs `us-east-1` cited in
different config files for the same environment, in the same repo, in multiple repos).

## Per-repo migration checklist (for any repo not yet on the target model)

- [ ] State key matches [BACKEND.md](BACKEND.md)'s convention, declared in a config file, not
      hardcoded in workflow YAML
- [ ] Workflow calls `hoad-org/github-automation`'s reusable workflows and `get-aws-secrets`, not
      `aws-terraform-platform-aws-workflows` (either org) or raw GitHub secrets
- [ ] No `secrets.*` reference to `SEED_ROLE_ARN`, `AWS_CI_ROLE_ARN`, `AWS_OIDC_ROLE_ARN`,
      `TF_STATE_BUCKET`, `KMS_KEY_ID`, or `PLAN_PASSPHRASE` anywhere in the repo's workflows
- [ ] Repo's OIDC subjects live in the seed repo's `github_oidc_plan_subjects`/
      `github_oidc_apply_subjects` (not the legacy combined `github_oidc_subjects`), using the
      repo's **current** name/org, not a stale pre-rename one
- [ ] `expected_account_id`/org/OU IDs (if any) verified against a live AWS API call, not copied
      from another repo, a template, or an SSO URL
- [ ] Zero BeyondTrust content (grep for `beyondtrust`, `BT-IT-Infrastructure`, `bt-avm`, `bt-dev`,
      `btsb`, `btc`, `@beyondtrust.com` across every file — and check git history, not just the
      current tree, if the repo has ever committed real credentials/emails)
- [ ] No dependency on `aws-python-platform-tfctl`/`awsctl-platform-aws`/`cloudctl` until their own
      BeyondTrust-content status is resolved (see "Adjacent tooling" above)
- [ ] `rhyscraig/aws-terraform-platform-docs` updated in the same change (this repo — see the
      guardrail in [PROVENANCE.md](PROVENANCE.md))
