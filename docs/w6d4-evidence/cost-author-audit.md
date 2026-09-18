## cost-author audit — expense-config (service=expense env=dev budget=100)

| Rule | Verdict | Citation | Reasoning |
|---|---|---|---|
| 1. Four-key tags on NAT/RDS | PASS | `cfn/expense-network-dev.yaml:187-198` (NatGatewayA), `:219-228` (NatGatewayBPerAz), `:249-258` (NatGatewayCPerAz), `cfn/expense-app-dev.yaml:114-123` (DBInstance) | All three NAT gateways and the DB instance carry `service`, `tenant`, `feature`, and `Environment`. |
| 2. `env=sandbox` preserved, `Environment` carries deployment env | PASS | `cfn/expense-network-dev.yaml:191`, `cfn/expense-app-dev.yaml:117-118` | Every taggable resource keeps `env=sandbox` (required by the account SCP) and uses a separate `Environment: !Ref EnvName` key for cost attribution instead of overloading `env`. |
| 3. SNS TopicPolicy allows Budgets + CloudWatch to publish | PASS | `cfn/expense-cost-dev.yaml:53-60` | Single statement grants `sns:Publish` to both `budgets.amazonaws.com` and `cloudwatch.amazonaws.com` on `CostAlarmsTopic`. |
| 4. Budget has FORECASTED>80% and ACTUAL>100%, `BudgetLimit` matches `--budget` | PASS | `cfn/expense-cost-dev.yaml:78-94` (notifications), `:10-14` + `:69-71` (`MonthlyBudgetUsd` default 100, wired to `BudgetLimit.Amount`) | Both notification types are present and wired to the SNS topic; `--budget 100` matches the template's own `Default: 100` for `MonthlyBudgetUsd`. |
| 5. Billing alarm never uses `TreatMissingData: notBreaching` | PASS | `cfn/expense-cost-dev.yaml:118` | Alarm on `AWS/Billing` `EstimatedCharges` uses `TreatMissingData: ignore`, not `notBreaching` — correct, since billing samples update only every ~6h and `notBreaching` would mask a real gap as healthy. |
| 6. No AWS Budget attempts to govern LLM/Anthropic spend | PASS | `cfn/expense-cost-dev.yaml` (whole file), `cfn/*.yaml` (repo-wide grep) | No `anthropic`, `claude`, or LLM-spend tag/filter appears anywhere in the CloudFormation templates — Anthropic spend correctly stays outside AWS Budgets scope. |
| 7. NAT gateway count gated by dev-vs-per-AZ cost lever | PASS | `cfn/expense-network-dev.yaml:214`, `:244` (`Condition: IsProdLike` on `NatGatewayBPerAz`/`NatGatewayCPerAz`) | Only `NatGatewayA` is unconditional; the B/C per-AZ gateways are gated behind `IsProdLike`, so `dev` deploys one NAT (~$32/mo) and a prod-like environment deploys three (~$96/mo). |

### Accepted

- **Tagging every NAT gateway and the RDS instance with the deployable four-key taxonomy** (`cfn/expense-network-dev.yaml:187-258`, `cfn/expense-app-dev.yaml:114-123`) is correct and should stay as-is: untagged NAT/RDS is the largest avoidable blind spot in Cost Explorer, and the template achieves this without touching the SCP-mandated `env=sandbox` tag — it adds `Environment` alongside it rather than replacing it.

### Rejected

- **Setting `TreatMissingData: notBreaching` on the `EstimatedCharges` alarm** (the template correctly uses `ignore` at `cfn/expense-cost-dev.yaml:118`) would be wrong: `AWS/Billing` metrics are account-wide and only update ~every 6 hours, so a `notBreaching` policy would silently report "healthy" during multi-hour gaps in billing data delivery, defeating the alarm's purpose. `ignore` is the only value that treats missing samples as "no new information" instead of a false-positive health signal.

Audited by: cost-author skill, 2026-09-18T07:08:37Z, against expense-config@1cc94c5
