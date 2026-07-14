# Process

## `main` only — never `master`, on any repo

Hard guardrail, no exceptions. Every repo's default branch must be `main`. Two repos
(`aws-terraform-platform-seed`, `aws-terraform-solutions-craighoad-blog`) were on `master` and
were renamed via GitHub's branch-rename API (which correctly retargets open PRs and moves the
default-branch pointer atomically — don't do this by hand with `git push --delete` +
`git push -u`, it's easy to orphan open PRs or leave the default pointer stuck). If you find a
repo still on `master`, fix it the same way, and check for any OIDC trust subject
(`ref:refs/heads/master`) that needs updating alongside the rename — a stale subject silently
breaks that repo's next deploy.

## PR-required for every change, no direct pushes to `main`

Every change in this landing zone goes through a branch + PR, reviewed before merge — including
changes made by a Claude session. This applies even to changes that feel small or obviously
correct (a one-line data fix, a dead-code removal) — the discipline is the point, not the size of
the individual change. The only exception this session actually used: pushing directly to `main`
on a genuinely brand-new, empty repo (nothing to bypass review of, since there's no prior history)
— not a precedent for anything else.

## Update the docs repo alongside every AWS-platform-affecting change

This repo (`rhyscraig/aws-terraform-platform-docs`) is the canonical home of the landing zone
design — established explicitly, not incidentally. Any change to OIDC roles, secrets, the
state-backend prefix convention, the account/OU tree, or a reusable workflow's contract must
update the relevant file here **as part of the same PR**, not as a follow-up. This is enforced by
instruction in every AWS-platform-touching repo's own `CLAUDE.md` — check that repo's CLAUDE.md if
you're unsure whether your change qualifies; when in doubt, update the docs.

## Verify before writing, always

Every real, fabricated-data incident this landing zone has hit (a bogus organization ID, bogus OU
IDs, a dead-but-misleadingly-named backend block) traces back to the same root cause: a value was
copied from a template, another repo, or an adjacent tool's output, and never checked against a
live API call before being committed as fact. Before writing any AWS account ID, OU ID,
organization ID, or resource ARN into a config file: verify it with a live query
(`aws organizations describe-organization`, `aws sts get-caller-identity`, etc.), don't copy it
from somewhere that looked authoritative.
