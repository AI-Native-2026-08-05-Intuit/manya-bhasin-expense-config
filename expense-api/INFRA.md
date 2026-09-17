# expense-api — CloudFormation infra (Week 6 Day 3)

How this gitops repo provisions **Manya-only** AWS stacks in the shared
training account (`625397071689`, `us-east-1`). Stack and resource names use
the `-manya` suffix so they do not collide with generic `expense-*-dev` or
teammate stacks.

Default branch: **`config-1`**. The `cfn-validate` workflow and PRs target
`config-1`. OIDC trust is pinned to
`AI-Native-2026-08-05-Intuit/manya-bhasin-expense-config`, matched with
wildcards on the org/repo names (see the OIDC note below).

## Stack layout

| Order | Stack name | Template | Purpose |
| --- | --- | --- | --- |
| 1 | `expense-bootstrap-dev-manya` | `cfn/expense-bootstrap-dev.yaml` | Bootstrap S3 bucket + `expense-api-cfn-deploy-manya` OIDC role |
| 2 | `expense-artifacts-dev-manya` | `cfn/expense-artifacts-dev.yaml` | Hardened artefacts bucket |
| 3 | `expense-network-dev-manya` | `cfn/expense-network-dev.yaml` | 3-AZ VPC, NAT (one in dev), app SG |
| 4 | `expense-app-dev-manya` | `cfn/expense-app-dev.yaml` | RDS Postgres + SecretTargetAttachment (Task 3 scope) |

Parameters shared across templates:

- **`PersonName`**: `manya` — suffix for globally unique names.
- **`OwnerTag`**: `manya-bhasin` — required SCP tag `user`.
- Taggable resources carry resource-level tags `env: sandbox`,
  `user: !Ref OwnerTag`, and `Project: expense`. Do not also use `Env`
  (IAM treats tag keys as case-insensitive). Some resources also have `Name`.

Exports use **`${AWS::StackName}-<Output>`** (e.g.
`expense-network-dev-manya-PrivateSubnets`). The app stack imports via
`NetworkStackName` (default `expense-network-dev-manya`), not hardcoded IDs.

## Deploy order and ChangeSet flow

Never execute a blind `create-stack` / `update-stack`. Always create a change
set, review `describe-change-set`, then execute.

SCP tagging is applied **on the resources in the templates**, not via
stack-level `--tags`.

Bootstrap and app need `--capabilities CAPABILITY_NAMED_IAM`.

### Bootstrap bucket

Bucket name is
`expense-bootstrap-${EnvName}-manya-${AWS::AccountId}`
(for this account: `expense-bootstrap-dev-manya-625397071689`).

This account’s SCP can deny **`s3:CreateBucket`** for some users. If CREATE
fails that way, create the bucket out-of-band with the same hardening
(versioning, SSE-KMS `alias/aws/s3`, PAB ×4, lifecycle, SCP tags) and
**IMPORT** it into the stack, then UPDATE to add the bucket policy and role.

### RDS credentials

Create the shared secret **out of band** (the template does not create it):

```bash
aws secretsmanager create-secret \
  --name expense/dev/db-master \
  --secret-string '{"username":"expense_admin","password":"REPLACE_ME"}' \
  --region us-east-1
```

RDS `MasterUsername` / `MasterUserPassword` use dynamic references to
`expense/${EnvName}/db-master`.

`DbSecretTargetAttachment` in `cfn/expense-app-dev.yaml` still attaches that
existing secret to the RDS instance (`TargetType: AWS::RDS::DBInstance`) so
rotation can target the live DB ARN.

### Network CIDR

Default **`VpcCidr`** is **`10.44.0.0/16`** (10.42 / 10.43 are used by
teammates). Do not change a live VPC CIDR in Task 4 — use a harmless additive
tag update; **`ec2:DeleteTags`** is also denied by SCP, so do not change the
`user` tag value on NAT gateways.

## Cross-stack safety

Deleting `expense-network-dev-manya` while `expense-app-dev-manya` exists
must fail with an export-in-use error (`AppSgId`, `PrivateSubnets`, `VpcId`).

## Drift (Task 4)

On `expense-artifacts-dev-manya`: detect drift → add a console tag →
`DRIFTED` → remove tag → `IN_SYNC`. Capture `describe-stack-resource-drifts`
and the network UPDATE change set from the CLI/console for the PR (not stored
as a `docs/` folder in this repo).

## OIDC note (GitHub `sub` claim)

This org's GitHub Enterprise setup appends internal numeric entity IDs to
the org and repo names in the token's `sub` claim, e.g.

```text
repo:AI-Native-2026-08-05-Intuit@311288174/manya-bhasin-expense-config@123456:pull_request
```

An exact-match trust policy therefore fails with
`Not authorized to perform sts:AssumeRoleWithWebIdentity`. The policy in
`cfn/expense-bootstrap-dev.yaml` wildcards around those suffixes:

```text
repo:${GitHubOrg}*/${GitHubRepo}*:ref:refs/heads/config-1
repo:${GitHubOrg}*/${GitHubRepo}*:pull_request
```

with `GitHubOrg=AI-Native-2026-08-05-Intuit` and
`GitHubRepo=manya-bhasin-expense-config`. `aud` stays `StringEquals`.

Changing the template alone does not fix CI — the live
`expense-api-cfn-deploy-manya` role only picks this up after an **UPDATE**
change set on `expense-bootstrap-dev-manya`.

## cfn-author Skill audit (brief)

- **Accepted**: `!Cidr` / `!Select` subnet layout (portable `VpcCidr`).
- **Rejected**: `NoEcho` DB password parameter — use Secrets Manager dynamic
  reference to the existing `expense/dev/db-master` secret instead.
- **Checked**: `DeletionPolicy: Retain` paired with `UpdateReplacePolicy: Retain`
  on buckets and RDS.

## Local validation

```bash
cfn-lint --regions us-east-1 -t cfn/*.yaml
cfn_nag_scan --fail-on-warnings --input-path cfn/
aws cloudformation validate-template --template-body file://cfn/<file>.yaml --region us-east-1
```

CI: `.github/workflows/cfn-validate.yml` assumes
`arn:aws:iam::625397071689:role/expense-api-cfn-deploy-manya`.
