# Budget — $0/month, enterprise-lite

This landing zone runs on a strict **$0/month** budget. "Enterprise-lite" means real separation of
duties, real least-privilege IAM, a real multi-account structure, a real centralized Terraform
backend — at zero AWS spend, not "cheap." Nothing in this estate should ever incur cost without an
explicit, informed decision. This file is the source of truth for what's safe to provision and
what isn't.

**If you are a Claude session about to provision a new AWS resource type: check this file first.
If the service/action isn't in the "in active use" list below, don't assume it's free — ask.**

## In active use today (all free-tier-safe at current scale)

Lambda, API Gateway, S3, CloudFront, DynamoDB, SNS, SQS, Secrets Manager, IAM, STS, CloudFormation
(including StackSets — the service itself is free, only the resources it deploys can cost money),
Route53, ACM (public certs are free), KMS (see the one named exception below), native S3
conditional-writes state locking (`use_lockfile = true` — no DynamoDB lock table, avoiding that
cost entirely), AWS Organizations (SCPs are free), AWS Budgets (first 2 budgets/account free), IAM
Identity Center / SSO (free), CloudTrail (management events are free; data events cost — not
currently enabled).

## The one named, accepted exception: customer-managed KMS key

The shared Terraform state bucket (`hcp-cmc-euw1-platform-tfstate-prd`) is encrypted with a
**customer-managed** KMS key (`alias/hcp-cmc-euw1-platform-tfstate`), not AWS-managed SSE-S3
(free). A customer-managed key costs **~$1/month**. This is a real, known, deliberate exception —
not an oversight.

**Why it's kept, not switched to free SSE-S3**: the blast radius of switching is real.
`personal-ai-cloud`'s IAM policies (`infra/aws/environments/prod/oidc.tf`,
`infra/aws/configs/envs/prd.tfvars`) already hardcode `Resource` ARN statements pointing at this
exact key — committed Terraform source, not yet applied. Switching the bucket's encryption would
leave those statements dangling without a coordinated follow-up across at least that repo. If this
is ever revisited, grep the whole estate for `Resource = [var.shared_kms_key_arn]`-shaped
statements first — don't assume this is the only dependent.

## Guardrails — enforced at the IAM layer, not just in prose here

An AWS Budget with a low threshold (e.g. $5) and an SNS/email alert is configured on the
management account — pure monitoring, catches genuine cost drift regardless of what caused it.

A Service Control Policy denies the specific, well-known-costly services/actions below at the
**Workloads OU** (Prod + NonProd) — deliberately not at root, so Management/Security/Infrastructure
stay unrestricted for baseline tooling and for administering the SCP itself. This is the real
technical enforcement layer: even a future Claude session that reads this file wrong, or doesn't
read it at all, cannot actually provision these resources — the API call itself is denied.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyKnownCostlyComputeAndDatabase",
      "Effect": "Deny",
      "Action": [
        "rds:CreateDBInstance", "rds:CreateDBCluster", "redshift:CreateCluster",
        "es:CreateElasticsearchDomain", "es:CreateDomain", "opensearch:CreateDomain",
        "eks:CreateCluster", "sagemaker:CreateNotebookInstance", "sagemaker:CreateEndpoint",
        "sagemaker:CreateTrainingJob", "kafka:CreateCluster", "elasticache:CreateCacheCluster",
        "elasticache:CreateReplicationGroup", "memorydb:CreateCluster"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyNATGatewayCosts",
      "Effect": "Deny",
      "Action": ["ec2:CreateNatGateway"],
      "Resource": "*"
    },
    {
      "Sid": "DenyLargeAndGPUInstanceTypes",
      "Effect": "Deny",
      "Action": ["ec2:RunInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "ForAnyValue:StringNotLike": {
          "ec2:InstanceType": ["t2.micro", "t2.small", "t3.micro", "t3.small", "t4g.micro", "t4g.small"]
        }
      }
    },
    {
      "Sid": "DenyGuardDutyAndConfigEnable",
      "Effect": "Deny",
      "Action": [
        "guardduty:CreateDetector", "config:PutConfigurationRecorder",
        "config:StartConfigurationRecorder", "securityhub:EnableSecurityHub", "macie2:EnableMacie"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyManagedFileSystemsAndWorkspaces",
      "Effect": "Deny",
      "Action": [
        "fsx:CreateFileSystem", "elasticfilesystem:CreateFileSystem",
        "workspaces:CreateWorkspaces", "globalaccelerator:CreateAccelerator"
      ],
      "Resource": "*"
    }
  ]
}
```

Deliberately does **not** touch anything in active use above, or free-tier-eligible EC2 instance
types — a $0 budget with zero EC2 at all would be more restrictive than this estate actually needs;
`t2`/`t3`/`t4g` micro/small stay allowed.

`guardduty:CreateDetector`/`config:PutConfigurationRecorder`/etc. matter here specifically as a
guardrail against a **future Claude session "helpfully" enabling them** — see
[SECURITY_BASELINE.md](SECURITY_BASELINE.md) for why they're off by design, and
[SECURITY_ROADMAP.md](SECURITY_ROADMAP.md) for when to turn them on.

**Status of this SCP**: drafted and safe-tested (`aws organizations create-policy` — creates but
doesn't attach; `aws iam simulate-custom-policy` against the existing workload CI role policies to
confirm no collision with anything in active use). Attaching it to the Workloads OU is a deliberate
step, done once, not silently — check `PROVENANCE.md`/git history in this repo for whether it's
actually attached yet before assuming it is.
