# expense-api — CloudFormation infra (Week 6 Day 3)

Four stacks live in `cfn/` of this gitops repo. They are deployed in
**strict order**. Later stacks import earlier exports; deleting a producer
while a consumer exists is refused by CloudFormation.

Default branch for this repo is `config-1` (not `main`). PRs and the
`cfn-validate` workflow target `config-1`.

| Order | Stack name | Template | What it creates |
| --- | --- | --- | --- |
| 1 | `expense-bootstrap-dev` | `cfn/expense-bootstrap-dev.yaml` | Packaged-template S3 bucket + OIDC role `expense-api-cfn-deploy` |
| 2 | `expense-network-dev` | `cfn/expense-network-dev.yaml` | 3-AZ VPC, public/private subnets, NAT (one in `dev`), app SG |
| 3 | `expense-artifacts-dev` | `cfn/expense-artifacts-dev.yaml` | Hardened artefacts bucket `uptimecrew-expense-artifacts-dev` |
| 4 | `expense-app-dev` | `cfn/expense-app-dev.yaml` | RDS Postgres 16, ECS Fargate + ALB; imports network exports |

GitHub OIDC trust is pinned to
`AI-Native-2026-08-05-Intuit/manya-bhasin-expense-config`, not the
application repo.

## Deploy ordering and ChangeSet flow

Bootstrap first (nothing to import). Network second. Artifacts can run
in parallel with app **after** network exists; app **must** wait for
network exports.

Every create or update uses a ChangeSet. Never `deploy` that auto-executes.

```bash
# CREATE (stack does not exist yet)
aws cloudformation create-change-set \
  --stack-name expense-bootstrap-dev \
  --change-set-name bootstrap-create \
  --change-set-type CREATE \
  --template-body file://cfn/expense-bootstrap-dev.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

aws cloudformation describe-change-set \
  --stack-name expense-bootstrap-dev \
  --change-set-name bootstrap-create \
  --region us-east-1

aws cloudformation execute-change-set \
  --stack-name expense-bootstrap-dev \
  --change-set-name bootstrap-create \
  --region us-east-1

aws cloudformation wait stack-create-complete \
  --stack-name expense-bootstrap-dev \
  --region us-east-1
```

For an in-place update (Task 4 CIDR/tag tweak), use `--change-set-type UPDATE`
and confirm `Replacement: False` on every `Modify` before execute.

Network / app / artifacts follow the same pattern. App and bootstrap need
`--capabilities CAPABILITY_NAMED_IAM`. App also needs
`--parameters ParameterKey=CertificateArn,ParameterValue=<acm-arn>`
(and `ImageUri` when pinning a digest).

One-time secret (password never in git or in a `NoEcho` parameter):

```bash
aws secretsmanager create-secret \
  --name expense/dev/db-master \
  --secret-string '{"username": "expense_admin", "password": "REPLACE_ME_LOCAL_DEV_ONLY"}' \
  --region us-east-1
```

RDS `MasterUsername` / `MasterUserPassword` are
`{{resolve:secretsmanager:expense/${EnvName}/db-master:SecretString:username|password}}`.

## Cross-stack export names

Exports use `expense-<layer>-${EnvName}-<Output>` so they stay unique per env.

**Network → app**

- `expense-network-dev-VpcId`
- `expense-network-dev-PublicSubnets` (comma-joined)
- `expense-network-dev-PrivateSubnets`
- `expense-network-dev-AppSgId`
- `expense-network-dev-VpcCidr`

App consumes them with `Fn::ImportValue` / `!Split` on the subnet lists.
Subnet IDs are never hardcoded.

**Bootstrap** (`expense-bootstrap-dev-BootstrapBucketName`,
`…BootstrapBucketArn`, `…CfnDeployRoleArn`) is for `cloudformation package`
and the GitHub Actions assume-role ARN.

After app (or any importer) is up, `delete-stack` on `expense-network-dev`
must fail with **Export is in use**. That is the safety net; cancel the
delete.

## Drift verification (Task 4)

Deliberate console edit: add a tag on `uptimecrew-expense-artifacts-dev`.

```bash
aws cloudformation detect-stack-drift --stack-name expense-artifacts-dev --region us-east-1
aws cloudformation describe-stack-resource-drifts --stack-name expense-artifacts-dev --region us-east-1
```

Expect `DRIFTED`. Revert the tag in the console; a second detect should
return `IN_SYNC`. Paste 4–6 lines of the drift JSON into the PR body.

## CI (`cfn-validate.yml`)

On every PR to `config-1` that touches `cfn/`:

1. `cfn-lint` with `-a cfn_lint_serverless.rules`
2. `cfn_nag_scan --fail-on-warnings --input-path cfn/`
3. `aws cloudformation validate-template` for each file (OIDC into
   `expense-api-cfn-deploy`)

`validate-template` needs the bootstrap role to already exist in the
account. Until Task 1 is `CREATE_COMPLETE`, that last step cannot pass
in Actions — run it locally after `aws sso login`.

Mark `cfn-validate` as a required status check on `config-1` once the
workflow has run once.

## NAT Conditions

`IsProdLike` is true for `staging` and `prod`. `IsDev` is the inverse
(also written onto the VPC `NatPattern` tag). Dev creates a single NAT
in AZ-a; private route tables B and C point at it. Staging/prod create
`NatGatewayBPerAz` and `NatGatewayCPerAz`. That is a cost vs HA split,
not two templates.

## Why dynamic reference instead of `NoEcho`

`NoEcho` only hides the value in `describe-stacks`. It still lives in
the ChangeSet / console parameter blob and in operator shell history.
Secrets Manager stores the secret outside the template; CFN resolves it
at create/update. Rotating the secret does not require a template edit
of the password string.

## Why cfn-lint and cfn-nag together

`cfn-lint` is CloudFormation-aware syntax and resource-spec checking
(plus serverless extra rules). `cfn-nag` is a security opinion set
(open SGs, unencrypted storage, IAM wildcards). A template can be
lint-clean and still fail nag; both must be 0 for the PR.

## cfn-author skill audit (run on a scratch branch)

Compared to `/cfn-author expense --region us-east-1` output, these
templates already avoid the usual skill mistakes:

| Skill quirk | What we did instead |
| --- | --- |
| `StringLike` on the OIDC **aud** claim | `StringEquals` on `aud` = `sts.amazonaws.com`; `StringLike` only on `sub` |
| `NoEcho: true` password parameter | Secrets Manager dynamic reference; secret created out of band |
| `DeletionPolicy: Retain` on S3 without `UpdateReplacePolicy: Retain` | Both set on bootstrap + artefacts buckets |
| GitHub org/repo defaults `uptimecrew/expense-config` | Defaults `AI-Native-2026-08-05-Intuit` / `manya-bhasin-expense-config` |

Re-run the skill on a throwaway branch before merge and paste any new
deltas here if the generated YAML drifts.

## Deploy status (this branch)

Templates and CI are in git. AWS ChangeSets are **blocked** until
`aws sts get-caller-identity` succeeds for account `625397071689`
(SSO token was expired when the templates were authored). After login:

1. Create `expense/dev/db-master` if missing
2. ChangeSet-create bootstrap → network → artifacts → app
3. Paste `describe-change-set` JSON into the PR
4. Drift + UPDATE changeset on network
5. Confirm `cfn-validate` green
