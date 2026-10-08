# Foundry Operator Dashboard

An Azure Monitor workbook that gives Foundry operators one page for the health, cost and risk of their agents. It reads the OpenTelemetry data the agents already send to Application Insights. It needs no code changes in the agents.

This package contains the workbook and the documentation around it. Start with the [Quick start](#quick-start), then use the [reading order](#what-to-read-and-when) below.

## What's in this package

| File | What it is | Who needs it |
|---|---|---|
| [`artifacts/workbooks/foundry-operator-dashboard.workbook.json`](artifacts/workbooks/foundry-operator-dashboard.workbook.json) | The workbook definition. Import it into Azure Monitor. | Whoever deploys it |
| [`docs/FOUNDRY_OPERATOR_GUIDE.md`](docs/FOUNDRY_OPERATOR_GUIDE.md) | How operators use it: routines, triage playbooks, thresholds, best practices, limitations. | Operators and team leads |
| [`docs/FOUNDRY_OPERATOR_WORKBOOK.md`](docs/FOUNDRY_OPERATOR_WORKBOOK.md) | Every section and panel explained, the data model, parameters, import steps, what to change per environment. | Operators and workbook owners |
| [`docs/FOUNDRY_OPERATOR_QUERY_REFERENCE.md`](docs/FOUNDRY_OPERATOR_QUERY_REFERENCE.md) | The full KQL, Resource Graph and REST call text of all 30 queries, each with a step-by-step explanation and known pitfalls. | Anyone who edits or reuses a query |

## The five questions it answers

1. Is the agent reachable and processing requests?
2. Which inner operation failed: agent runtime, model call, tool call, or dependency?
3. Is the agent behavior useful, safe, compliant and aligned with its task?
4. What is changing over time in latency, errors, tokens, cost and evaluation scores?
5. Who must act, through which channel, and using which runbook?

## What the workbook covers

| Section | What an operator learns |
|---|---|
| At a glance | Runs, success rate, error rate, p95 latency, tokens and cost for the selected range |
| Tokens | Token use over time, by model and by project |
| Latency | End-to-end and per-model latency (p50, p95), and a time-to-first-token proxy |
| Errors | Failures by layer: agent runs, model calls, tool calls, throttling (HTTP 429) |
| Runs, tools and health | Run volume, tool performance, and a composite health status per agent (Healthy, Degraded, Down, No Recent Activity) |
| Cost and capacity | Estimated spend by model, month-to-date budget burn-down, throttling (HTTP 429) |
| Hidden failures | Runs that report success but contain a failed model or tool call |
| Operational depth | Alert rule inventory, live alert self-check, external dependency health, model deployment inventory |
| Usage patterns | Conversation length distribution and a volume anomaly check (last 24 hours against the previous 7-day average) |
| Telemetry health | Ingestion freshness and trace completeness, so you can trust the other panels |
| Quality and safety | Pointers to the Foundry portal Evaluations and Red team views |

See [`docs/FOUNDRY_OPERATOR_WORKBOOK.md`](docs/FOUNDRY_OPERATOR_WORKBOOK.md) for the panel-by-panel description.

## Prerequisites

- Foundry agents that send OpenTelemetry GenAI spans to an Application Insights resource. Hosted agents need `ENABLE_INSTRUMENTATION=true`.
- Reader access on that Application Insights resource. Reader on the subscription is also needed for the alert-rules panel and the project and account pickers, and Reader on the Foundry account for the model inventory panel.
- Permission to create workbooks in the resource group that holds the Application Insights resource.
- At least 8 days of retained data for the usage anomaly check.

## Quick start

1. In the Azure portal, open the **Application Insights** resource, then **Workbooks**, then **New**.
2. Click **Advanced editor** (`</>`), switch to the **Gallery Template** tab, and paste the contents of [`foundry-operator-dashboard.workbook.json`](artifacts/workbooks/foundry-operator-dashboard.workbook.json).
3. Click **Apply**, then **Save**. Choose a name, subscription and resource group.
4. At the top of the workbook, choose your **Subscriptions** and one or more **AppInsightsResources**. The resource picker has no default, so the panels stay empty until you choose. The **FoundryAccount** picker defaults to the first Foundry account. Change it to list the model deployments of another account.
5. Set the **TimeRange**, then walk through the sections.
6. Complete the checklist below before operators rely on it.

## Before you rely on it

The workbook is generic, so a few items need your values:

- [ ] **Budget:** set `MONTHLY_BUDGET_USD` in the `cost-budget-burndown` query to your approved monthly budget. It ships with 50.
- [ ] **Prices:** the price table in the cost queries covers three models (gpt-4o-mini, gpt-4.1-mini, gpt-5-mini). The prices are Azure list prices for Global Standard deployments, checked on 2026-10-08. Update them if you use Data Zone or regional deployments (about 10% more) or Batch (about half), and add every model you deploy. The cost table flags unpriced models in its `price_listed` column.
- [ ] **Thresholds:** the suggested values are starting points. Agree them against your own SLOs. See section 5 of the operator guide.
- [ ] **Escalation text:** the ownership and escalation notes in the text blocks are generic. Replace them with your teams, channels and runbooks.
- [ ] **Alert rules:** the workbook shows alert rules but doesn't create them. Create scheduled-query alerts for the signals you need. The `ops-alert-self-check` panel is a starting point.

The full list is in the [per-environment section](docs/FOUNDRY_OPERATOR_WORKBOOK.md#things-to-change-per-environment) of the workbook doc.

## What to read and when

| You are | Read |
|---|---|
| An operator on shift | [Operator guide](docs/FOUNDRY_OPERATOR_GUIDE.md), sections 3 and 4 (routines and triage playbooks) |
| Setting up the workbook | This page, then [workbook doc](docs/FOUNDRY_OPERATOR_WORKBOOK.md) |
| Setting alert thresholds | [Operator guide](docs/FOUNDRY_OPERATOR_GUIDE.md), section 5 |
| Changing or reusing a query | [Query reference](docs/FOUNDRY_OPERATOR_QUERY_REFERENCE.md) |
| Wondering whether a number can be trusted | [Operator guide](docs/FOUNDRY_OPERATOR_GUIDE.md), section 2, and the telemetry health panels |

## Things to know

- **Estimates, not invoices.** Cost is calculated from token counts and the price table. Cached input tokens are billed lower than the table assumes, so an estimate can run above the invoice. Reconcile with Azure Cost Management.
- **Three panels ignore the time range.** The self-check (last 15 minutes), the budget burn-down (month to date) and the usage anomaly check (24 hours against 7 days) use fixed windows.
- **Hidden failures matter.** An agent can report `success=true` while its model calls fail. It then shows 0% errors in the error panels. Check the hidden-failures panels.
- **Quiet is not healthy.** A system with no traffic looks the same as an outage in the ingestion freshness panel.
- **Time to first token is a proxy.** It is the duration of the whole model-call span, not a true streaming measurement.
- **Quality and safety isn't live in the workbook.** Use the Foundry portal Evaluations and Red team views for current results.

The complete list is in section 7 of the [operator guide](docs/FOUNDRY_OPERATOR_GUIDE.md) and in the workbook doc.

## Validation status

All 26 KQL queries, the Resource Graph queries and the model-inventory REST call were executed read-only against a live Foundry environment and ran without errors. The workbook was also deployed as a separate workbook, and the deployed definition matched the file in this package. Executing without errors doesn't prove the numbers match your data. After you import the workbook, check a few panels against raw telemetry in Application Insights Logs, and use the telemetry health section to confirm data is arriving.

## Support

Questions or changes to the workbook: contact Eran Alshech.
