# Index

Machine-readable lookup. **Read this file first.**

> **BEFORE proposing any new AWS resource, check [BUDGET.md](BUDGET.md)'s deny-list and the SCP
> below. If unsure, it's not free — ask.**

## Known accepted exceptions — don't re-flag these as bugs

- **Customer-managed KMS key on the shared state bucket** (~$1/month) — deliberate, see
  [BUDGET.md](BUDGET.md#the-one-named-accepted-exception-customer-managed-kms-key).
- **`personal-ai-cloud` and `craighoad-portfolio-website` aren't `aws-terraform-*` prefixed** —
  defensible exceptions (app repos with embedded infra / non-Terraform infra), see
  [REPOS.md](REPOS.md#naming-convention-violations).
- **The seed repo's own state key** (`hcp/aws/control-plane/terraform.tfstate`) doesn't match
  the `hcp/prd/platform/{repo}/{region}/` convention used by `aws-accounts`/`aws-baselines`/
  `aws-org` — pre-dates that convention, migrating a root-of-trust repo's live state for cosmetic
  consistency isn't worth the risk, see [BACKEND.md](BACKEND.md).
- **8 repos still consume the older `aws-terraform-platform-aws-workflows` repo**, not
  `hoad-org/github-automation` — the real remaining migration work, not a bug in any one repo, see
  [REPOS.md](REPOS.md#the-central-architectural-fact).

## Guardrails — do not violate without explicit human sign-off

- Never enable GuardDuty, Config, Security Hub, or Macie, or create RDS/Redshift/EKS/SageMaker/
  managed-Kafka/ElastiCache/NAT-Gateway resources, or `ec2:RunInstances` outside
  t2/t3/t4g micro/small — enforced by an SCP at the Workloads OU, see [BUDGET.md](BUDGET.md).
- Never use `master` as a branch name on any repo — `main` only, see [PROCESS.md](PROCESS.md).
- Never write an AWS account ID, OU ID, organization ID, or resource ARN into a config file
  without verifying it against a live API call first — see [PROCESS.md](PROCESS.md) and
  [PROVENANCE.md](PROVENANCE.md) for why this rule exists (a fabricated org ID and 5 fabricated OU
  IDs sat undetected across multiple repos for an extended period).
- Never apply the CloudFormation StackSet (member CICD role deployment) without confirming the
  target OU IDs in `myorg.tfvars` are current — see [CICD_ROLES.md](CICD_ROLES.md).

## By topic

| I need to know... | Read | Summary |
|---|---|---|
| What accounts/OUs exist, real IDs | [ARCHITECTURE.md](ARCHITECTURE.md) | Org structure, account IDs, OU IDs, naming conventions, regional gotchas |
| $0-budget stance, what's free vs costly, the cost-guardrail SCP | [BUDGET.md](BUDGET.md) | Allow-list, the one named KMS exception, the SCP JSON |
| Maximum security hygiene achievable at $0 | [SECURITY_BASELINE.md](SECURITY_BASELINE.md) | What's on (free), what's deliberately off (costs money) |
| Phased security upgrades once there's revenue | [SECURITY_ROADMAP.md](SECURITY_ROADMAP.md) | Ordered enhancement list, one trigger condition each |
| How the state backend/bucket/key convention works | [BACKEND.md](BACKEND.md) | Shared bucket, per-repo-class key convention, KMS exception |
| Which repo has which CI/CD role, OIDC trust subjects | [CICD_ROLES.md](CICD_ROLES.md) | Plan/apply/module-skills role split, StackSet-deployed member role |
| How secrets are stored/read in CI | [SECRETS.md](SECRETS.md) | AWS Secrets Manager convention, `github/hcp/*`, `get-aws-secrets` contract |
| How to bootstrap a brand-new repo/account | [SEEDING.md](SEEDING.md) | Seed repo mechanism, ordering, drift reconciliation |
| What every repo is, its CI/CD state, migration checklist | [REPOS.md](REPOS.md) | Full inventory across both GitHub orgs, naming exceptions |
| Branch/PR/docs-update conventions | [PROCESS.md](PROCESS.md) | `main`-only, PR-required, verify-before-writing |
| Why something is named/structured the way it is, past corrections | [PROVENANCE.md](PROVENANCE.md) | Root-caused mistakes, so they aren't re-investigated from scratch |

## By AWS account ID

| Account | ID | OU |
|---|---|---|
| management | 395101865577 | Management |
| hcp-audit | 824737149338 | Security |
| hcp-log-archive | 404459110295 | Security |
| hcp-shared-services | 995039994868 | Infrastructure |
| hcp-craighoad-prod | 624426145233 | Workloads/Prod |
| hcp-terrorgems-prod | 767828739298 | Workloads/Prod |
| hcp-qa | 821868546447 | Workloads/NonProd |

Full detail, including real OU IDs: [ARCHITECTURE.md](ARCHITECTURE.md).
