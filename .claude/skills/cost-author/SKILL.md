---
name: cost-author
description: >
  Audit AWS cost-governance choices in this repo's CloudFormation templates against a
  fixed rubric, and propose/verify the four-key cost-allocation tag taxonomy
  (service/env-or-Environment/tenant/feature). Invoke as
  `/cost-author <repo> --service <svc> --env <env> --budget <usd>`. Reads the real
  cfn/*.yaml files in the target repo, checks each rule below against what is actually
  written there, and reports one ACCEPTED and one REJECTED finding minimum, each with a
  file:line citation and the reasoning. Never invents findings that aren't backed by an
  actual line in the templates. Trigger on: "run cost-author", "cost-author audit",
  "/cost-author", "audit the cost stack", "audit cost tags".
---

# Skill: cost-author

A lightweight, repo-grounded cost-governance auditor for CloudFormation-based AWS spend.
It does not call any external service — it reads the target repo's `cfn/*.yaml` files and
scores them against the rubric below. Every finding must cite a real `file:line`. If a
rule can't be checked because the relevant resource doesn't exist in this repo, say so
explicitly rather than fabricating a verdict.

## Invocation

```
/cost-author <repo-path-or-name> --service <svc> --env <env> --budget <usd-int>
```

- `repo-path-or-name` — the repo to audit (e.g. `expense-config`, or a path to it).
- `--service` — expected value of the `service` cost-allocation tag.
- `--env` — expected value of the `Environment`/`env` tag (see Rule 2 for why this repo
  may split the two).
- `--budget` — expected `AWS::Budgets::Budget` `BudgetLimit` in USD.

## Procedure

1. Locate the repo's CloudFormation templates (default: `cfn/*.yaml` at the repo root;
   fall back to `**/*.yaml` under a `cfn/` or `cloudformation/` directory if not found).
2. Grep/read every template for the resource types and tag blocks named in the rubric.
3. Evaluate each rule below against what is actually present. Record PASS, FAIL, or
   N/A (resource type not present in this repo) with a `file:line` citation for every
   verdict that isn't N/A.
4. Produce the audit report in the format specified under "Output format".
5. Never mark something ACCEPTED or REJECTED without a citation. If you can't find the
   line, the verdict is N/A, not a guess.

## Rubric

| # | Rule | Check |
|---|---|---|
| 1 | **Four-key cost-allocation tags on taggable spend.** Every `AWS::EC2::NatGateway` and `AWS::RDS::DBInstance` resource must carry all four keys: `service`, `tenant`, `feature`, and an environment key (`Environment` or `env`, see Rule 2). | Grep the resource's `Tags:` block for all four. FAIL if any NAT gateway or the DB instance is missing one. |
| 2 | **Sandbox `env` vs deployment `Environment` are not conflated.** If the account/repo reserves `env=sandbox` for SCP ownership (check for a comment or a literal `{Key: env, Value: sandbox}` alongside `{Key: Environment, Value: !Ref EnvName}`), then `env` must stay `sandbox` and deployment attribution must use `Environment`, never replacing `env`. | FAIL if a resource sets `env` to anything other than `sandbox` where the sandbox convention is in use, since that would make the resource undeployable per the account's SCP. |
| 3 | **SNS TopicPolicy allows both `budgets.amazonaws.com` and `cloudwatch.amazonaws.com` to publish.** | Find the `AWS::SNS::TopicPolicy` (or inline policy) attached to the cost-alarms topic. FAIL if either principal's `sns:Publish` statement is missing — a Budget or Alarm bound to a topic without this silently never notifies. |
| 4 | **`AWS::Budgets::Budget` has both a FORECASTED>80% and an ACTUAL>100% notification, and `BudgetLimit` matches `--budget`.** | Check `NotificationsWithSubscribers`. FAIL if either notification type/threshold pair is missing, or if `BudgetLimit.Amount` doesn't resolve to `--budget`. |
| 5 | **Billing alarm never uses `TreatMissingData: notBreaching`.** `AWS::Billing` `EstimatedCharges` updates infrequently (~every 6h); `notBreaching` would mask real gaps as healthy. The correct value is `ignore`. | Grep for `TreatMissingData` on any `AWS::CloudWatch::Alarm` on `EstimatedCharges`. FAIL if the value is `notBreaching`. REJECT any proposal to set it to `notBreaching`, citing this reasoning. |
| 6 | **No cost governance for LLM/Anthropic spend is attempted via `AWS::Budgets::Budget`.** AWS Budgets cannot see non-AWS spend (Anthropic token usage is billed outside AWS). | REJECT any budget or cost filter that references `anthropic`, `claude`, or an LLM-spend tag/key — that spend must be governed by an Anthropic Console workspace limit and in-app cost logging (e.g. `X-Cost-Usd`, EMF cost logs), never an AWS Budget. |
| 7 | **NAT gateway count matches the stated dev-vs-per-AZ cost lever.** A `dev`/non-prod environment should provision exactly one NAT gateway (~$32/mo); a prod-like environment should provision one per AZ (~$96/mo total). | Check the `Conditions` block for something like `IsProdLike` gating additional `AWS::EC2::NatGateway` resources or their route associations. FAIL if the same NAT gateway is unconditionally reused across all AZs regardless of environment, or if extra NAT gateways are always created regardless of the condition. |

## Output format

Write the audit as a single markdown block, structured exactly like this (drop rows marked
N/A from the printed table but keep them out of the accepted/rejected counts):

```
## cost-author audit — <repo> (service=<svc> env=<env> budget=<budget>)

| Rule | Verdict | Citation | Reasoning |
|---|---|---|---|
| 1. Four-key tags | PASS/FAIL | file:line | one sentence |
| ... | | | |

### Accepted
- <one sentence: what's right, citing file:line, and why it's the correct call>

### Rejected
- <one sentence: what would be wrong to do, citing the rule it violates, and why>

Audited by: cost-author skill, <ISO 8601 timestamp>, against <repo>@<git short SHA>
```

The "Accepted" and "Rejected" sections must each contain at least one real item backed by
a PASS/FAIL row above — never placeholder text. If every rule but one is N/A, say so
plainly rather than padding the report.

## Non-negotiables

- Every verdict needs a `file:line` citation from an actual grep/read of the repo's
  templates in this invocation. A citation from memory or a prior run is not valid —
  re-read the files every time.
- Do not add commit metadata, a "generated by AI" watermark, or restate the rubric table
  in the final report beyond the format above.
- If the repo has no `cfn/` templates at all, say so and stop — do not fabricate an audit
  of a template that doesn't exist.
