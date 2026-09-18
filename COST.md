# Expense cost governance

## Two cost planes

The AWS-resident plane includes the NAT Gateway, RDS, and S3. It is governed
with the AWS monthly budget in `cfn/expense-cost-dev.yaml`, an account billing
alarm, and investigation in Cost Explorer.

The LLM plane is direct Anthropic Claude token usage. It is governed with an
Anthropic Console workspace spend limit, per-feature application cost logs,
and the `X-Cost-Usd` response header. AWS Budgets cannot see Anthropic spend.

## Cost allocation taxonomy

The course taxonomy is `service / env / tenant / feature`. This sandbox reserves
`env=sandbox` for SCP ownership controls, so replacing it with `env=dev` would
make otherwise valid resources undeployable. The deployable taxonomy is:

- `env=sandbox` for mandatory sandbox ownership
- `Environment=dev` for deployment-environment attribution
- `service=expense`
- `tenant=shared`
- `feature=categorize-expense`

Cost allocation tags must be activated in the payer account before they appear
in Cost Explorer or Budgets. Activation does not backfill historical spend.

## NAT cost lever

Development uses one NAT Gateway, approximately $32/month in the course
estimate. Staging and production use one gateway per availability zone: three
gateways are approximately $96/month. A single gateway lowers fixed development
cost but creates an availability-zone dependency; per-AZ gateways cost more
while avoiding cross-AZ dependency and improving resilience.

## Budget runbook

The monthly budget publishes to SNS when forecast spend is greater than 80% and
when actual spend is greater than 100%. On notification:

1. Open Cost Explorer for the same billing period.
2. Group or filter by the activated `service` and `Environment` cost tags.
3. Confirm NAT Gateway, RDS, and S3 usage and identify an unexpected increase.
4. Do not attribute Anthropic charges to this budget; inspect Anthropic Console
   and application cost logs separately.

## Billing alarm runbook

`AWS/Billing` `EstimatedCharges` is account-wide and cannot be tag filtered.
Billing samples update relatively infrequently, so the alarm uses
`TreatMissingData=ignore`. Use Cost Explorer after an alarm to attribute the
account total.

## Anthropic key rotation

1. Create or rotate the key in Anthropic.
2. Update the out-of-band Kubernetes Secret named `anthropic-api`.
3. Roll the `expense-api` Deployment.
4. Verify traffic, then revoke the old key.

Never put an Anthropic key in Git, source code, YAML, logs, tests, screenshots,
or documentation.

## cost-author audit

Run via the repo-local `.claude/skills/cost-author` skill on scratch branch
`w6d4-cost-author-audit`: `/cost-author expense-config --service expense --env dev --budget 100`.
Full report with per-rule verdicts and file:line citations:
[`docs/w6d4-evidence/cost-author-audit.md`](docs/w6d4-evidence/cost-author-audit.md).

- **Accepted:** Tagging every NAT gateway and the RDS instance with the deployable four-key taxonomy (`cfn/expense-network-dev.yaml:187-258`, `cfn/expense-app-dev.yaml:114-123`) — untagged NAT/RDS is the largest avoidable blind spot in Cost Explorer, and the template adds `Environment` alongside the SCP-mandated `env=sandbox` tag rather than replacing it.
- **Rejected:** Setting `TreatMissingData: notBreaching` on the `EstimatedCharges` alarm (the template correctly uses `ignore` at `cfn/expense-cost-dev.yaml:118`) — `AWS/Billing` metrics are account-wide and update only ~every 6 hours, so `notBreaching` would silently report "healthy" during a real gap in billing data delivery.

## Shared-account naming deviation

The course examples use unsuffixed physical names. SNS topics, AWS Budgets, and
CloudWatch alarms are account-scoped or account/region-scoped, so this template
adds `-manya` to avoid touching or colliding with another learner's resources.
The assignment's logical resource IDs remain unchanged.
