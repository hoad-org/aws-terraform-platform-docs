# Provenance — and the guardrail this repo enforces

## Where this design came from

This landing zone's patterns (account-vending, OU structure, split plan/apply OIDC roles,
StackSet-deployed member CICD roles, centralized Terraform state) were adapted from a
BeyondTrust internal reference architecture ("AVM tenant seed"), used **for inspiration only**.
BeyondTrust is Craig's employer — this is Craig's own, entirely separate, personal $0/month AWS
environment. Nothing about it should retain any BeyondTrust-specific content: no BeyondTrust git
module sources, no BeyondTrust account/org identifiers, no BeyondTrust naming literals
(`bt-avm`, `bt-dev`, `btsb`, `infra-co`, etc.), no `@beyondtrust.com` email addresses, no
references to BeyondTrust's own internal repos or tooling (`BT-IT-Infrastructure-CloudOps/*`,
`cloudctl-skill`'s BeyondTrust PyPI index, BeyondTrust's Jira instance).

**A closed incident**: on 2025-12-29, three real AWS accounts were created using BeyondTrust's
example email addresses, copied verbatim from a reference template rather than substituted with
real values. See [ARCHITECTURE.md](ARCHITECTURE.md) for the full detail — already remediated
(accounts closed), recorded here so the underlying mistake (treating template/example values as
if they were real input) isn't repeated.

**Ongoing audit status — a full pass across every `aws-terraform-*` repo in both `rhyscraig` and
`hoad-org`, plus adjacent tooling, is now complete.** Findings, not all resolved:

1. **Current-tree content in `aws-terraform-*` repos**: clean, except the seed repo itself
   (`configs/orgs/{bt-avm,bt-dev,fdr-cmc,fdr-gvc}.tfvars` — deleted this session, PR open; and
   `README.md`/`AI.md` still reference `git clone https://github.com/BT-IT-Infrastructure-CloudOps/...`
   — not yet fixed).
2. **Git history exposure — real, not yet remediated**: `aws-terraform-platform-aws-accounts` and
   `aws-terraform-platform-aws-org` both have commits in their history (not the current tree)
   exposing `@beyondtrust.com` email addresses (`cloud-security-emergency@`, `aws-avm-audits@`,
   `aws-avm-security@`, `aws-avm-shared-svcs@`) and a committer identity
   `choadbt <choad@beyondtrust.com>` used across multiple commits in both repos. **This cannot be
   fixed by a normal commit** — removing it requires rewriting git history (`git filter-repo` or
   equivalent) and force-pushing, which breaks every existing clone, PR, and commit-SHA reference.
   This is a real decision with real tradeoffs, not something to do unilaterally — flagged here,
   not yet actioned.
3. **Adjacent tooling still actively BeyondTrust-branded**: `hoad-org/cloudctl` (used this session,
   for real, to get AWS credentials), `hoad-org/awsctl-platform-aws`, and
   `rhyscraig/aws-python-platform-tfctl` all have READMEs stating they are for "BeyondTrust
   Engineering internal use only," origin-clone from `BT-IT-Infrastructure-CloudOps/*`, and (for
   `cloudctl`) use `bt-avm` as the example org name throughout their own usage docs. Also
   BeyondTrust-branded, but not `aws-terraform-*`-related infra: none found beyond these three
   tooling repos. See [REPOS.md](REPOS.md)'s "Adjacent tooling" section — this needs an explicit
   decision (rebrand vs. stop using), not a silent fix.

## The guardrail — this repo must stay current

**Every change that affects the AWS platform must update this repo, as part of the same body of
work — not as a follow-up, not "later."** This includes but isn't limited to:

- A new or renamed OIDC role, policy, or trust subject
- A new onboarded repo, or a repo migrating between CI/CD models
- A changed state-key prefix or backend convention
- A new AWS account, OU, or a change to the account tree
- A changed secret name or contract in `get-aws-secrets`
- A new or changed reusable workflow in `hoad-org/github-automation`

**How this is enforced**: every repo's own `.claude/CLAUDE.md` (or equivalent orientation doc)
must instruct Claude to update the relevant file(s) in `hoad-org/aws-terraform-platform-docs`
before considering AWS-platform-affecting work complete. See each repo's own CLAUDE.md for the
exact instruction text — the canonical wording lives in this repo's own root `README.md`.

**Audit status**: this guardrail is now in every `aws-terraform-*` repo's `CLAUDE.md` that's
locally cloneable — `aws-terraform-platform-seed`, `aws-accounts`, `aws-baselines`, `aws-org`,
`aws-terraform-solutions-terrorgems-platform`, `aws-terraform-solutions-websites`,
`aws-terraform-solutions-craighoad-blog`, `hoad-org/personal-ai-cloud`,
`hoad-org/github-automation` (9 repos, each its own PR, none merged yet — check each repo's own
open PRs before assuming this is live everywhere). `craighoad-portfolio-website` wasn't locally
cloned this session, not yet audited.

## Known cross-repo inconsistencies found during the audit that produced this documentation

These are real, live discrepancies found while writing this documentation — not resolved yet,
listed here so they're not lost. See [REPOS.md](REPOS.md) for per-repo detail.

1. **At least four different CI/CD secrets models are live simultaneously**: (a) GitHub
   Environment secrets via the seed repo's `tools/create-gh-env.sh` (`SEED_ROLE_ARN`,
   `TF_STATE_BUCKET`, etc.), (b) a *different* set of GitHub Environment secrets
   (`AWS_CI_ROLE_ARN`/`AWS_OIDC_ROLE_ARN`, `GH_PAT`) consumed by `aws-accounts`, `aws-baselines`,
   `aws-org` via `rhyscraig/aws-terraform-platform-aws-workflows` (a third reusable-workflow repo,
   distinct from `hoad-org/github-automation`), (c) the new `hoad-org/github-automation` +
   Secrets-Manager model (`personal-ai-cloud`, and now the seed repo's own new workflow), (d) SSM
   Parameter Store, referenced inconsistently in `aws-accounts`' own README vs its actual
   Terraform (`/org/configuration` vs the documented `/platform/outputs/org`).
2. **Backend bucket/region values disagree across config files within the same repo**, in
   multiple repos (`aws-accounts`, `aws-baselines`, `aws-org` all show this pattern): the actual,
   live `backend.tf` uses bucket `hcp-cmc-euw1-platform-tfstate-prd` / region `eu-west-1`, while
   `environments.yml` and `.tfctl.yml` in the same repos reference a different, apparently-stale
   bucket (`infra-tfstate-hoad-org-seed`) and inconsistent regions (`eu-west-2` in one file,
   `us-east-1` in another). **One instance confirmed and fixed**: `aws-baselines`' own
   `terraform/security/versions.tf` had a second `backend "s3" {}` block naming this same bucket —
   confirmed genuinely dead two ways (it's a child module, Terraform ignores backend blocks
   outside the root; the bucket itself returns a live 404 on `head-bucket`) and removed (PR #5,
   with ADR-0001). The `environments.yml`/`.tfctl.yml` references in `aws-accounts`/`aws-org` are
   still unconfirmed dead-vs-live — same pattern, not yet independently checked in those two repos.
3. **CI is red across the board**: `drift_check`/`drift-check` workflows in `aws-accounts`,
   `aws-baselines`, and `aws-org` are all failing on every scheduled run, consistently completing
   in 0 seconds — indicating a fast pre-flight failure (likely auth/OIDC), not real infrastructure
   drift. Not yet root-caused.
4. **README files in multiple repos describe a different reality than their own Terraform** —
   wrong region claims, wrong secrets-storage claims. Not yet corrected.
5. **Broken `pip install` targets in CI**: the old `aws-workflows` repo's `reusable-drift-check.yaml`
   and `reusable-tf-destroy.yaml` install from `github.com/rhyscraig/tfctl.git` and
   `github.com/rhyscraig/awsctl.git` — neither repo exists under those exact names (real names:
   `aws-python-platform-tfctl`, `awsctl-platform-aws`). These installs are broken regardless of
   the BeyondTrust-branding question above.
6. **A billing lockout has blocked `aws-terraform-solutions-websites`'s CI for ~2.5 months**,
   unrelated to any code — GitHub Actions itself refused to start jobs
   ("recent account payments have failed or your spending limit needs to be increased"). Simple to
   fix once noticed; flagged here because it's exactly the kind of thing this audit exists to
   surface, not because it's architecturally interesting.
7. **Two stale OIDC trust subjects** in the seed repo's legacy `github_oidc_subjects` — **fixed**:
   removed the dead `aws-terraform-solutions-terrorgem`/`website-static-html-craighoad.com`
   entries (renamed/transferred repos, already trusted separately under their real current
   names).
8. **The fabricated `organization_id`/OU IDs — fixed.** Real, live-queried values now in both this
   doc and the seed repo's `configs/orgs/myorg.tfvars`. Writing the data did not itself apply the
   StackSet — see [CICD_ROLES.md](CICD_ROLES.md).
9. **Two repos were on `master`, not `main` — fixed.** `aws-terraform-platform-seed` and
   `aws-terraform-solutions-craighoad-blog` renamed via GitHub's branch-rename API (open PRs
   auto-retargeted, default-branch pointer moved atomically). The blog repo's OIDC subject updated
   to match. See [PROCESS.md](PROCESS.md) for the "main only" guardrail this enforces going
   forward.
