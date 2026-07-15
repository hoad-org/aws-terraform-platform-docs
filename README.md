# AWS Terraform Platform — Documentation

**Canonical home for the AWS landing zone design.** Every repo, every OIDC role, every state
bucket prefix, every secret name used across the `hcp` (Hoad Cloud Platform) AWS estate is
documented here. If a change to any `aws-terraform-*` repo, `hoad-org/github-automation`, or the
AWS Organization itself isn't reflected here, the change isn't done yet.

**The brief this landing zone exists to satisfy**: an enterprise-lite AWS Well-Architected landing
zone, run for **$0/month** baseline cost, for a single person (not a company with customers).
"Enterprise-lite" means: real separation of duties (plan vs apply, least-privilege OIDC roles),
a real multi-account structure (management / security / infrastructure / workloads), a real
centralized Terraform backend — implemented at the scale and cost appropriate for one person's
personal infrastructure, not a multi-tenant SaaS platform. Maximum security hygiene achievable at
that budget, with a documented, phased path to more once there's revenue to justify it.

## If you're a Claude session, read this first

**[docs/INDEX.md](docs/INDEX.md)** — the AI-friendly entry point: a directive on checking the
budget guardrail before proposing new resources, a topic lookup table, an account-ID lookup, and
the "known accepted exceptions, don't re-flag these" list. Read it before making any change here.

## Full doc set

- [INDEX.md](docs/INDEX.md) — start here, especially if you're an AI session
- [ARCHITECTURE.md](docs/ARCHITECTURE.md) — accounts, OUs, naming conventions, the whole shape of the estate
- [BUDGET.md](docs/BUDGET.md) — the $0/month stance: what's free, the one named exception, the cost-guardrail SCP
- [SECURITY_BASELINE.md](docs/SECURITY_BASELINE.md) — maximum security hygiene achievable at $0
- [SECURITY_ROADMAP.md](docs/SECURITY_ROADMAP.md) — phased enhancements once there's revenue
- [SEEDING.md](docs/SEEDING.md) — how a brand-new AWS Organization goes from empty to fully bootstrapped
- [BACKEND.md](docs/BACKEND.md) — the shared Terraform state bucket, KMS key, and state-key prefix convention
- [CICD_ROLES.md](docs/CICD_ROLES.md) — the OIDC role split (plan/apply/module-skills) and the StackSet-deployed member CICD role
- [SECRETS.md](docs/SECRETS.md) — Secrets Manager structure, `hoad-org/github-automation`'s `get-aws-secrets` contract
- [REPOS.md](docs/REPOS.md) — every repo in scope, its purpose, its current CI/CD model, migration status
- [PROCESS.md](docs/PROCESS.md) — `main`-only, PR-required, verify-before-writing
- [PROVENANCE.md](docs/PROVENANCE.md) — origin of the design (adapted from a BeyondTrust reference architecture, for inspiration only — no BeyondTrust content, accounts, or identifiers remain)

## The guardrail

**This repo must be updated as part of every change that affects the AWS platform** — a new
OIDC role, a changed state prefix, a new onboarded repo, a new account, a changed secret
contract. Not as an afterthought PR later. See [PROVENANCE.md](docs/PROVENANCE.md) for how this
is enforced across the other repos' own `CLAUDE.md` files.
