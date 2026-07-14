# Landing Zone Architecture

## Organization

- **AWS Organization ID**: `o-h07s8pk406` (root: `r-4if6`)
- **AWS SSO / IAM Identity Center**: home region **`eu-west-2`** (not `eu-west-1` — this matters,
  see "Regional gotchas" below). Directory ID `d-9c67661145`, start URL
  `https://d-9c67661145.awsapps.com/start`.
- **Management account**: `395101865577` ("management")

> **A past, now-corrected mistake, recorded here so it never happens again**: earlier tooling
> fabricated `organization_id = "o-9c67661145"` and five `ou-9c67-*` target OU IDs. These were
> never real — `9c67661145` is the SSO **directory ID**'s numeric suffix, misread as if it were
> the org ID's suffix, then propagated by copy-paste into `configs/orgs/myorg.tfvars` in multiple
> repos. None of these IDs ever resolved to anything in the real Organization. Always verify
> account/OU/org IDs via a live `aws organizations` call before writing them into a config file —
> never copy them from a template, another repo, or an SSO URL.

## Account structure

```
Root (r-4if6)
├── Management OU        (ou-4if6-0shp9icd)
│   └── management                    395101865577
├── Security OU          (ou-4if6-m4l4xw13)
│   ├── hcp-audit                     824737149338
│   └── hcp-log-archive               404459110295
├── Infrastructure OU    (ou-4if6-3ou6vcd6)
│   └── hcp-shared-services           995039994868
├── Production OU        (ou-4if6-1edkqgjc)   — reserved, currently empty
├── QA OU                (ou-4if6-qqwvvwvy)   — reserved, currently empty
└── Workloads OU         (ou-4if6-yvxs57c8)
    ├── Prod OU          (ou-4if6-on6pmizj)
    │   ├── hcp-craighoad-prod        624426145233
    │   └── hcp-terrorgems-prod       767828739298
    └── NonProd OU       (ou-4if6-mjcqxg7t)
        └── hcp-qa                    821868546447
```

**Every workloads account is a child of `Workloads/Prod` or `Workloads/NonProd`** — never a
direct child of the root or of `Production`/`QA` (those top-level OUs are reserved for future use,
not currently where workload accounts live).

### A closed incident, recorded for the record

On 2025-12-29, three AWS accounts were created inside this Organization using example email
addresses from a BeyondTrust reference template (`aws-avm-audits@beyondtrust.com`,
`aws-avm-security@beyondtrust.com`, `aws-avm-shared-svcs@beyondtrust.com`), named "Log Archive",
"Audit", and "Shared Services". These were **real accounts created for real**, not just
config drift. They have since been closed (`State: CLOSED`, account IDs `033462815202`,
`159012149044`, `093624542399`) and will be fully removed from the Organization by AWS after the
standard ~90-day closure window. **Never copy example/template values (emails, account names,
IDs) directly into a real `terraform apply` without substituting real values first.**

## Naming conventions

**Repos**: every repo that provisions or manages AWS infrastructure via Terraform is prefixed
`aws-terraform-*`. A repo that touches AWS but isn't Terraform-based, or isn't prefixed this way,
is a naming-convention violation — see [REPOS.md](REPOS.md) for the current compliance list.

**Terraform/AWS resource naming** (`name_prefix` in the seed repo's `locals.tf`):
```
{org}-{partition_short}-{region_short}-{system}
```
- `org` = `hcp` ("Hoad Cloud Platform")
- `partition_short` = `cmc` (commercial `aws` partition) or `gvc` (`aws-us-gov`)
- `region_short` = `euw1` (`eu-west-1`), `usw2`, `use1`, etc.
- `system` = `platform` for the seed/landing-zone repos

Example: `hcp-cmc-euw1-platform` → state bucket `hcp-cmc-euw1-platform-tfstate-prd`, KMS alias
`alias/hcp-cmc-euw1-platform-tfstate`, logs bucket `hcp-cmc-euw1-platform-logs-prd`.

**OIDC role naming is a DIFFERENT formula — deliberately drops `partition_short`**:
```
{org}-{region_short}-{system}-github-oidc-tf-{plan,apply}-role
```
Example: `hcp-euw1-platform-github-oidc-tf-plan-role`. This exact formula is load-bearing —
`hoad-org/github-automation`'s `get-aws-secrets` action derives the plan-role ARN from it
mechanically (`{org}-{region_short}-platform-github-oidc-tf-plan-role`), so the role name and
this formula must always match. See [CICD_ROLES.md](CICD_ROLES.md).

## Regional gotchas

- **AWS IAM Identity Center (SSO Admin API) only returns results in its home region, `eu-west-2`**
  — even though everything else in this landing zone runs in `eu-west-1`. Any Terraform touching
  `aws_ssoadmin_*` resources needs a second, aliased AWS provider pinned to `eu-west-2`. Getting
  this wrong doesn't error clearly — `data.aws_ssoadmin_instances` just silently returns an empty
  list, which then fails downstream with a confusing "invalid index" error.
- **CloudFormation StackSets need TWO separate activations**, not one:
  1. `aws organizations enable-aws-service-access --service-principal=member.org.stacksets.cloudformation.amazonaws.com`
  2. `aws cloudformation activate-organizations-access` (a **separate**, CloudFormation-specific
     switch — `aws cloudformation describe-organizations-access` shows its true status; enabling
     #1 alone is not sufficient and fails with a misleadingly identical error message).
