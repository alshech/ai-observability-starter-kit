# Foundry Operator Dashboard (Azure Monitor Workbook)

An Azure Monitor **workbook** that gives operators one page for the health, cost and risk of Foundry agents. It reads the OpenTelemetry data the agents send to Application Insights.

- **Workbook definition:** [`artifacts/workbooks/foundry-operator-dashboard.workbook.json`](../artifacts/workbooks/foundry-operator-dashboard.workbook.json)
- **Query reference:** [`FOUNDRY_OPERATOR_QUERY_REFERENCE.md`](FOUNDRY_OPERATOR_QUERY_REFERENCE.md) has the full KQL of every query with a step-by-step explanation.
- **Operator guide:** [`FOUNDRY_OPERATOR_GUIDE.md`](FOUNDRY_OPERATOR_GUIDE.md) explains how to use it day to day.
- **Origin:** exported from a reference Foundry environment, with environment-specific IDs removed.
- **Contents:** 56 items. About 30 are KQL panels, and the rest are headings, notes and parameters.

> The definition is stored as a workbook "gallery template" JSON (the `serializedData` value), so it can be imported in the portal or deployed from an ARM/Bicep template.

## The five questions it answers

1. Is the agent reachable and processing requests?
2. Which inner operation failed: agent runtime, model call, tool call, or dependency?
3. Is the agent behavior useful, safe, compliant and aligned with its task?
4. What is changing over time in latency, errors, tokens, cost and evaluation scores?
5. Who must act, through which channel, and using which runbook?

## How to import it into another environment

1. In the Azure portal, open the target **Application Insights** resource, then **Workbooks**, then **New**.
2. Click **Advanced editor** (`</>`) and switch to the **Gallery Template** tab.
3. Paste the contents of `foundry-operator-dashboard.workbook.json` and click **Apply**, then **Save**.
4. Edit the items below before relying on it (see [Things to change per environment](#things-to-change-per-environment)).

## Data model the queries rely on

The agents run with `ENABLE_INSTRUMENTATION=true` and emit GenAI OpenTelemetry spans. Application Insights stores them as:

| Table | Span | How the workbook uses it |
|---|---|---|
| `requests` | `invoke_agent ...` (one per agent run) | Runs, success and error rate, end-to-end latency. Agent name is `customDimensions["gen_ai.agent.name"]`. |
| `dependencies` | `chat <model>` (one per model call) | Tokens (`gen_ai.usage.input_tokens` / `output_tokens`), model-call latency, cost. Model is parsed from the span name. |
| `dependencies` | `execute_tool ...` (one per tool call) | Tool volume, latency and errors. Tool is `gen_ai.tool.name`. |
| `dependencies` | `type` = `HTTP` / `AI` | Throttling (HTTP 429) and external dependency health. |
| Azure Resource Graph | alert rules, model deployments | Operations inventory panels. |

`operation_Id` is the trace ID. It links a run in `requests` to its model and tool calls in `dependencies`. The "hidden failures" and "trace completeness" panels depend on that join.

Table panels add a `foundryProject` column, taken from `split(_ResourceId, "/")[8]`. That is the Application Insights resource name, and it keeps rows distinguishable when several projects are selected.

## Parameters (top of the workbook)

| Parameter | Type | Purpose |
|---|---|---|
| `TimeRange` | time range picker, default 24 h | Applies to every query panel. |
| `Subscriptions` | subscription picker | Scopes the two Resource Graph panels (alert rules, model inventory). |
| `AppInsightsResources` | resource picker (Resource Graph query on `microsoft.insights/components`) | Selects which Foundry project(s) every KQL panel reads. No default: choose one or more after import. |

## Sections and queries

### At a glance (`kpi-tiles`)
Six tiles from one `union` query: **Agent Runs**, **Success Rate (%)**, **Error Rate (%)**, **P95 Latency (ms)**, **Tool Calls**, **Total Tokens**.
- Runs, success and error come from `requests` where `name has "invoke_agent"`.
- Tool calls count `dependencies` where `name startswith "execute_tool"`.
- Tokens sum input and output tokens over `chat ` dependencies.
- Rates return 0 when there are no runs, so the tile stays empty-safe.

### 1. Token utilization
| Panel | What it shows | Query logic |
|---|---|---|
| Token usage over time, by model | Input plus output tokens per 5-minute bin, split by model | `chat ` dependencies, model = second word of the span name, `summarize ... by bin(timestamp, 5m), model` |
| Token totals and averages by model | Calls, total and average input and output tokens per project and model | Same base, summarized by `foundryProject, model` |

### 2. Latency
| Panel | What it shows | Query logic |
|---|---|---|
| End-to-end agent latency | p50, p90 and p95 of `invoke_agent` duration per 5 minutes | `percentile(duration, ...)` on `requests` |
| Model-call latency (TTFB proxy) | p50 and p95 of `chat <model>` span duration by model | Instrumentation emits one span per full model call, not per streamed token, so this is a proxy for time-to-first-token. True TTFB needs streaming-layer instrumentation. |

### 3. Error rates
| Panel | What it shows |
|---|---|
| Agent run volume over time, by agent and outcome | Counts per 5 minutes, split by agent and `success` / `error` |
| Error rate (%) by agent | `failed / total` per project and agent, worst first |
| Top error codes by agent | Failed runs grouped by `resultCode` |
| Invocation errors by class | Failed runs classed as 4xx (client or config) or 5xx (server or backend) from `resultCode` |

`agent-framework-agent-broken-model` is an intentional negative test. In the reference environment its runs report success even though the model call fails, so it shows about 0% here and appears under hidden failures instead. This is a good example of why that panel exists.

### 4. Agent runs
**Sessions: turn count per conversation (top 50)** counts `invoke_agent` requests per `operation_Id` and agent, with the last-seen time. A high turn count flags runaway loops or very long conversations. Note that `operation_Id` is a trace, so this equals "turns per trace", which matches a conversation only when one trace spans the whole conversation.

### 5. Tool calls
| Panel | What it shows |
|---|---|
| Tool call volume over time | `execute_tool` dependencies per 5 minutes |
| Tool calls, p95 latency and error rate, by tool | Calls, p95 duration, failed count and error rate per tool (`gen_ai.tool.name`, falling back to the span name) |

Rising tool latency or tool errors often appears before agent-level errors.

### 6. Success rates
**Success rate (%) by agent**: `success / total` per agent, sorted ascending so the worst agent is on top.

### 7. Health status
**Composite health status by agent** combines recency, error rate and latency:

| Status | Rule (evaluated top to bottom) |
|---|---|
| No Recent Activity | last invocation more than 60 minutes ago |
| Down | error rate at least 20% |
| Degraded | error rate at least 5%, or p95 above 15,000 ms |
| Healthy | everything else |

The table also shows the error rate, p95, minutes since last seen and last-seen time. It is sorted by severity (Down, Degraded, No Recent Activity, Healthy), then by error rate, so problems appear first.

A hosted agent that reports `success=true` while its model calls fail shows 0% errors here. Check the hidden-failures panels for it.

### 8. Cost and capacity
| Panel | What it shows |
|---|---|
| Estimated spend over time, by model | Hourly estimated USD per model |
| Estimated spend totals by model | Calls, tokens and estimated USD per project and model |
| Month-to-date spend vs. approved budget | Spend this month, budget, percent of budget and a status (`On track`, `>=80% of budget`, `Over budget`) |
| HTTP dependency calls vs. 429 responses | All HTTP dependency calls versus HTTP 429 (throttled) per 5 minutes, with the throttle rate |

Cost is `tokens / 1000 * price_per_1k`, using an inline `datatable` of list prices:

| Model | Input per 1K | Output per 1K |
|---|---|---|
| gpt-4o-mini | 0.00015 | 0.0006 |
| gpt-4.1-mini | 0.0004 | 0.0016 |
| gpt-5-mini | 0.00025 | 0.002 |

These are estimates. Reconcile with Azure Cost Management. Spans report versioned model names (for example `gpt-4.1-mini-2025-04-14`). The queries strip the trailing date so they match the price table. A model that isn't in the table gets a null price and is left out of the cost chart and budget. The cost table flags it with `price_listed = NO - add to price table`.

### Hidden failures
A successful `invoke_agent` run can still contain a failed tool or model call that the agent recovered from or ignored. Alert rules that only check `requests.success` can't see this.
- **Successful runs with a failed child dependency, by agent:** inner-joins successful runs to failed `dependencies` on `operation_Id`, and lists the failed dependency names (up to 10).
- **Overall hidden failure rate:** distinct traces with a failed child, divided by all successful runs.

### 9. Quality and safety (static)
A text note, not a live query. Evaluation scores and red-team results come from the Foundry Evaluations service, not from the telemetry in Application Insights. The note says where to find current results (Foundry portal Evaluations and Red team panes) and suggests a cadence.

### 10. Operational depth
| Panel | Source | What it shows |
|---|---|---|
| Configured alert rules | Resource Graph | Scheduled query rules in the resource group: name, severity, enabled, description |
| Live self-check | KQL | Whether the two main alerts would fire right now: any failed `invoke_agent` in the last 15 minutes, and p95 above 30 s in the last 15 minutes |
| External dependency health | KQL | Calls, success rate and p95 per target for `HTTP` and `AI` dependencies |
| Deployed model inventory | Resource Graph | Model deployments: model, version, SKU, capacity, provisioning state |

A note lists the alert payload each action group should carry (environment, agent, trace IDs, runbook, owner).

### 11. Usage patterns
| Panel | What it shows |
|---|---|
| Conversation length distribution | How many traces had 1, 2, 3 ... runs |
| Volume anomaly check | Runs in the last 24 hours versus the average daily count of the previous 7 days, with the percent change |

### Telemetry and evaluation health
Check these before trusting the other charts.
- **Ingestion freshness:** minutes since the last `requests` or `dependencies` row. More than 60 minutes shows as stale.
- **Trace completeness:** the percent of `invoke_agent` runs that have at least one child span. A low value means model and tool spans are missing, so confirm `ENABLE_INSTRUMENTATION=true`.

### Reference sections (text only)
- **Known challenges and mitigations:** missing child spans, portal rollup lag, evaluation cadence, sensitive content in traces, alert noise, evaluator scale differences, preview or SDK changes.
- **KPI thresholds and ownership:** illustrative warn and page thresholds per KPI, with an owner and an escalation flow.
- **Open governance decisions:** telemetry content, data controls, SLOs, ownership, routing, release authority, standardization.
- **Footer:** pointers to Application Insights Logs, Azure Monitor Alerts and the Foundry Evaluations and Red team panes.

## Things to change per environment

1. **Project selection:** the `AppInsightsResources` parameter has no default. Pick your Application Insights resource(s) in the dropdown after import. To pre-select one, edit `selected = false` in the parameter's query to `selected = id =~ '<resource id>'`.
2. **Resource Graph scope:** `ops-alert-rules` and `ops-model-inventory` list every alert rule and model deployment in the selected subscriptions. Add `| where resourceGroup =~ '<rg>'` to narrow them.
3. **Prices:** edit the `datatable` in the three cost queries, and `MONTHLY_BUDGET_USD` (50.0) in the burn-down query.
4. **Alert thresholds in the self-check:** the 15-minute window, "errors greater than 0" and "p95 greater than 30 s" mirror the two deployed alert rules. Keep them in sync.
5. **Quality and safety note:** it points at the portal Evaluations and Red team panes. Adjust the cadence and wording to your process.

## Known limitations and gotchas

- **Hidden-failures table counts distinct runs.** `hidden_failures` is `dcount(operation_Id)`, so a run with several failed children counts once. Any failed dependency counts, not only model and tool calls.
- **Cost covers only three models.** See the pricing note above. Verify the prices against the current Azure price sheet for your region and deployment type before using them for budgets.
- **TTFB is a proxy.** See section 2.
- **Quality and safety isn't live.** Use the Foundry portal Evaluations and Red team views for current results.
- **Validation status.** All 26 KQL queries and the 3 Resource Graph queries were executed against a live environment and ran without errors. Re-run them against your own environment after import, because results depend on your data. See the query reference.
