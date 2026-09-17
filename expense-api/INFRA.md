# expense-api — CloudFormation infra (Week 6 Day 3)

How this gitops repo provisions **Manya-only** AWS stacks in the shared
training account (`625397071689`, `us-east-1`). Stack and resource names use
the `-manya` suffix so they do not collide with generic `expense-*-dev` or
teammate stacks.

Default branch: **`config-1`**. The `cfn-validate` workflow and PRs target
`config-1`. OIDC trust is pinned to
`AI-Native-2026-08-05-Intuit/manya-bhasin-expense-config` (with `*` wildcards
for Intuit GitHub entity IDs in the `sub` claim — see below).

## Stack layout

| Order | Stack name | Template | Purpose |
| --- | --- | --- | --- |
| 1 | `expense-bootstrap-dev-manya` | `cfn/expense-bootstrap-dev.yaml` | Bootstrap S3 bucket + `expense-api-cfn-deploy-manya` OIDC role |
| 2 | `expense-artifacts-dev-manya` | `cfn/expense-artifacts-dev.yaml` | Hardened artefacts bucket |
| 3 | `expense-network-dev-manya` | `cfn/expense-network-dev.yaml` | 3-AZ VPC, NAT (one in dev), app SG |
| 4 | `expense-app-dev-manya` | `cfn/expense-app-dev.yaml` | RDS Postgres only (Task 3 scope) |

Parameters shared across templates:

- **`PersonName`**: `manya` — suffix for globally unique names.
- **`OwnerTag`**: `manya-bhasin` — required SCP tag `user`.
- Every taggable resource also carries `Environment`, `env: sandbox`, and
  `Project: expense`. Use **`Environment`**, not `Env`, on IAM-tagged
  resources (duplicate keys are rejected case-insensitively).

Exports use **`${AWS::StackName}-<Output>`** (e.g.
`expense-network-dev-manya-PrivateSubnets`). The app stack imports via
`NetworkStackName` (default `expense-network-dev-manya`), not hardcoded IDs.

## Deploy order and ChangeSet flow

Never execute a blind `create-stack` / `update-stack`. Always create a change
set, review `describe-change-set`, then execute.

Pass stack tags on create (SCP):

```bash
--tags Key=env,Value=sandbox Key=user,Value=manya-bhasin Key=Project,Value=expense
```

Bootstrap and app need `--capabilities CAPABILITY_NAMED_IAM`.

### Bootstrap: SCP `s3:CreateBucket` deny

This account’s SCP (`p-upmysz2c`) denies **`s3:CreateBucket`** for some users.
If `expense-bootstrap-dev-manya` rolls back on bucket **CREATE**, use the same
pattern as the reference multistate capstone:

1. Create bucket **`expense-bootstrap-dev-manya-625397071689`** out-of-band with
   versioning, SSE-KMS (`alias/aws/s3`), PAB ×4, lifecycle, and SCP tags.
2. **IMPORT** change set with `docs/w6d3-evidence/import-resources.json`.
3. **UPDATE** change set to add `BootstrapBucketPolicy` and `CfnDeployRole`.

Template parameter **`ExistingBucketName`** points at that bucket.

### RDS credentials (out of band)

The app template does **not** create a Secrets Manager secret or
`SecretTargetAttachment`. Create the shared secret once:

```bash
aws secretsmanager create-secret \
  --name expense/dev/db-master \
  --secret-string '{"username":"expense_admin","password":"REPLACE_ME"}' \
  --region us-east-1
```

RDS `MasterUsername` / `MasterUserPassword` use dynamic references to
`expense/${EnvName}/db-master`.

### Network CIDR

Default **`VpcCidr`** is **`10.44.0.0/16`** (10.42 / 10.43 are used by
teammates). Do not change a live VPC CIDR in Task 4 — use a harmless tag-only
or additive update; **`ec2:DeleteTags`** is also denied by SCP, so do not
change the `user` tag value on NAT gateways.

## Cross-stack safety

Deleting `expense-network-dev-manya` while `expense-app-dev-manya` exists
must fail with an export-in-use error (`AppSgId`, `PrivateSubnets`, `VpcId`).

## Drift (Task 4)

On `expense-artifacts-dev-manya`: detect drift → add a console tag →
`DRIFTED` → remove tag → `IN_SYNC`. Evidence JSON lives under
`docs/w6d3-evidence/`.

## OIDC note (GitHub `sub` claim)

Trust policy uses:

```text
repo:${GitHubOrg}*/${GitHubRepo}*:ref:refs/heads/config-1
repo:${GitHubOrg}*/${GitHubRepo}*:pull_request
```

so internal `@<entity-id>` suffixes on org/repo names still match.

## cfn-author Skill audit (brief)

- **Accepted**: `!Cidr` / `!Select` subnet layout (portable `VpcCidr`).
- **Rejected**: `NoEcho` DB password parameter — use Secrets Manager dynamic
  reference to the existing `expense/dev/db-master` secret instead.
- **Checked**: `DeletionPolicy: Retain` paired with `UpdateReplacePolicy: Retain`
  on buckets, RDS, and stateful resources.

## Local validation

```bash
cfn-lint --regions us-east-1 -t cfn/*.yaml
cfn_nag_scan --fail-on-warnings --input-path cfn/
aws cloudformation validate-template --template-body file://cfn/<file>.yaml --region us-east-1
```

CI: `.github/workflows/cfn-validate.yml` assumes
`arn:aws:iam::625397071689:role/expense-api-cfn-deploy-manya`.
