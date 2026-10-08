# Foundry Operator Dashboard: Query Reference

Every query in the [Foundry Operator Dashboard](FOUNDRY_OPERATOR_WORKBOOK.md), with the KQL, a step-by-step explanation, how to read the result, and the pitfalls. For day-to-day use, see the [Operator Guide](FOUNDRY_OPERATOR_GUIDE.md).

There are 29 queries: 26 KQL panels against Application Insights, 2 Azure Resource Graph panels, and 1 Resource Graph parameter query. The workbook applies the time range from the `TimeRange` parameter to the KQL panels. The few panels with a fixed window (budget, self-check, anomaly check) say so.

**Validation:** all 26 KQL queries were executed against a live Application Insights resource (Logs API, 30-day range), and the 3 Resource Graph queries were executed with `az graph query`. All ran without errors. The KQL shown here is checked line by line against the workbook JSON.

## Contents

1. [Building blocks used by many queries](#1-building-blocks)
2. [Selection parameter](#2-selection-parameter)
3. [At a glance](#3-at-a-glance)
4. [Tokens](#4-tokens)
5. [Latency](#5-latency)
6. [Errors](#6-errors)
7. [Agent runs, tools, success, health](#7-runs-tools-success-and-health)
8. [Cost and capacity](#8-cost-and-capacity)
9. [Hidden failures](#9-hidden-failures)
10. [Operational depth](#10-operational-depth)
11. [Usage patterns](#11-usage-patterns)
12. [Telemetry health](#12-telemetry-health)
13. [Using the queries outside the workbook](#13-using-the-queries-outside-the-workbook)

---

## 1. Building blocks

### What the data looks like

| Table | Rows come from | Key columns used |
|---|---|---|
| `requests` | One per agent run (span `invoke_agent <agent>`) | `name`, `success`, `resultCode`, `duration` (ms), `operation_Id`, `customDimensions["gen_ai.agent.name"]` |
| `dependencies` | One per model call (`chat <model>`), tool call (`execute_tool <tool>`), or outbound call | `name`, `type`, `target`, `success`, `resultCode`, `duration` (ms), `operation_Id`, `customDimensions["gen_ai.usage.*"]`, `customDimensions["gen_ai.tool.name"]` |

`operation_Id` is the trace ID. It ties one run (`requests`) to all of its model and tool calls (`dependencies`).

### Repeated snippets

| Snippet | Meaning |
|---|---|
| `requests \| where name has "invoke_agent"` | Keep only agent runs. `has` matches the whole term, so it doesn't match unrelated request names. |
| `dependencies \| where name startswith "chat "` | Keep only model calls. The trailing space matters. |
| `dependencies \| where name startswith "execute_tool"` | Keep only tool calls. |
| `replace_regex(tostring(split(name, ' ')[1]), @'-\d{4}-\d{2}-\d{2}$', '')` | The model name: the second word of `chat gpt-5-mini`, with any trailing date version removed. Spans from some SDK versions report `gpt-5-mini-2025-08-07` and others `gpt-5-mini`. Without the cleanup, the same model splits into two rows and misses the price table. |
| `todouble(customDimensions["gen_ai.usage.input_tokens"])` | Token counts are stored as text in `customDimensions`, so they're cast to numbers. A missing value becomes null and is ignored by `sum`. |
| `extend agent = tostring(customDimensions["gen_ai.agent.name"])` then `iff(isempty(agent), "unknown-agent", agent)` | The agent name, with a label for runs that have none, so they don't vanish from group-by results. |
| `tostring(split(_ResourceId, "/")[8])` | The Application Insights resource name, shown as `foundryProject`. It lets rows from several selected projects be told apart. `_ResourceId` is only available because the workbook queries one or more resources. |
| `round(100.0 * x / total, 2)` | A percentage with two decimals. The `100.0` forces decimal division. |
| `percentile(duration, 95)` | The p95 duration in milliseconds. Prefer it to `avg`, which hides slow requests. |
| `bin(timestamp, 5m)` | Groups time into 5-minute buckets for charts. |

### The price table (cost queries)

Three queries embed this table. Update all three together.

```kusto
let pricing = datatable(model:string, in_per_1k:real, out_per_1k:real)
[
  "gpt-4o-mini", 0.00015, 0.0006,
  "gpt-4.1-mini", 0.0004, 0.0016,
  "gpt-5-mini", 0.00025, 0.002
];
```

Prices are USD per 1,000 tokens and are illustrative. A model that isn't listed gets null prices. The cost table flags it with `price_listed = NO - add to price table`, but the time chart and the budget panel leave it out.

---

## 2. Selection parameter

### `AppInsightsResources` (resource picker)

```kusto
Resources
| where type =~ 'microsoft.insights/components'
| order by name asc
| project value = id, label = name, selected = false, group = resourceGroup
```

- **Purpose:** fills the dropdown that chooses which Application Insights resources every panel reads.
- **How it works:** Azure Resource Graph lists all Application Insights components you can see. `value` is the resource ID used as the query scope, `label` is the display name, and `group` groups the list by resource group. `selected = false` means nothing is pre-selected.
- **Tune:** use `selected = id =~ '<resource id>'` to pre-select a default. Add `| where resourceGroup =~ '<rg>'` to shorten the list.

---

## 3. At a glance

### `kpi-tiles`: six headline numbers

```kusto
let reqs = requests | where name has "invoke_agent";
let dchat = dependencies | where name startswith "chat ";
let dtool = dependencies | where name startswith "execute_tool";
union
(reqs | summarize Value = toreal(count()) | extend Metric = "Agent Runs", Fmt = "short"),
(reqs | summarize Value = iff(count() == 0, 0.0, round(100.0 * countif(success == true) / count(), 2)) | extend Metric = "Success Rate (%)", Fmt = "pct"),
(reqs | summarize Value = iff(count() == 0, 0.0, round(100.0 * countif(success == false) / count(), 2)) | extend Metric = "Error Rate (%)", Fmt = "pct"),
(reqs | summarize Value = round(percentile(duration, 95), 1) | extend Metric = "P95 Latency (ms)", Fmt = "ms"),
(dtool | summarize Value = toreal(count()) | extend Metric = "Tool Calls", Fmt = "short"),
(dchat | extend intok = todouble(customDimensions["gen_ai.usage.input_tokens"]), outtok = todouble(customDimensions["gen_ai.usage.output_tokens"]) | summarize Value = round(sum(intok) + sum(outtok), 0) | extend Metric = "Total Tokens", Fmt = "short")
| project Metric, Value
```

- **Purpose:** one row per tile for a first look at health and volume.
- **How it works:** the three `let` lines name the three row sets. Each `union` branch reduces one of them to a single number and labels it. `Fmt` tells the tile renderer how to format the value (count, percent, milliseconds). The final `project` keeps the two columns the tiles need.
- **Read it as:** success plus error rate should be about 100%. A big gap means runs with no `success` value.
- **Pitfalls:** the `iff(count() == 0, ...)` guard returns 0 when there are no runs. A 0% error rate with 0 runs isn't "healthy", so read it next to **Agent Runs**.

---

## 4. Tokens

### `tokens-timechart`: tokens over time by model

```kusto
dependencies
| where name startswith "chat "
| extend model = replace_regex(tostring(split(name, ' ')[1]), @'-\d{4}-\d{2}-\d{2}$', '')
| extend input_tokens = todouble(customDimensions["gen_ai.usage.input_tokens"])
| extend output_tokens = todouble(customDimensions["gen_ai.usage.output_tokens"])
| summarize total_tokens = sum(input_tokens) + sum(output_tokens) by bin(timestamp, 5m), model
| order by timestamp asc
```

- **Purpose:** shows when and which model consumes tokens.
- **How it works:** keep model calls, parse the model name and token counts, add input and output tokens per 5-minute bucket and model.
- **Read it as:** a step change after a release suggests a prompt or model change. A steady climb with flat volume suggests growing prompts or conversation history.

### `tokens-table`: totals and averages per project and model

```kusto
dependencies
| where name startswith "chat "
| extend model = replace_regex(tostring(split(name, ' ')[1]), @'-\d{4}-\d{2}-\d{2}$', '')
| extend input_tokens = todouble(customDimensions["gen_ai.usage.input_tokens"])
| extend output_tokens = todouble(customDimensions["gen_ai.usage.output_tokens"])
| extend foundryProject = tostring(split(_ResourceId, "/")[8])
| summarize calls = count(), total_input = sum(input_tokens), total_output = sum(output_tokens), avg_input = round(avg(input_tokens), 1), avg_output = round(avg(output_tokens), 1) by foundryProject, model
| extend total_tokens = total_input + total_output
| order by total_tokens desc
```

- **Purpose:** compares models and projects on volume and prompt size.
- **How it works:** same base as above, grouped by project and model, with call count, totals and per-call averages. The biggest consumers sort first.
- **Read it as:** `avg_input` is the typical prompt size. A very high value usually means too much context is sent each call. `avg_output` shows answer length, which also drives cost.

---

## 5. Latency

### `latency-e2e-timechart`: end-to-end agent latency

```kusto
requests
| where name has "invoke_agent"
| summarize p50 = percentile(duration, 50), p90 = percentile(duration, 90), p95 = percentile(duration, 95) by bin(timestamp, 5m)
| order by timestamp asc
```

- **Purpose:** how long a user waits for an agent run, over time.
- **How it works:** the median, p90 and p95 of run duration per 5-minute bucket.
- **Read it as:** p50 moving means everything is slower. Only p95 moving means a slow tail, often a slow tool or throttling retries. A run's duration includes its model and tool calls.
- **Pitfalls:** a 5-minute bucket with few runs makes percentiles jumpy. Look at the trend, not single points.

### `latency-ttfb-barchart`: model-call latency (TTFB proxy)

```kusto
dependencies
| where name startswith "chat "
| extend model = replace_regex(tostring(split(name, ' ')[1]), @'-\d{4}-\d{2}-\d{2}$', '')
| summarize p50_ms = round(percentile(duration, 50), 1), p95_ms = round(percentile(duration, 95), 1) by model
| order by p95_ms desc
```

- **Purpose:** compare how fast each model answers.
- **How it works:** p50 and p95 duration of each `chat <model>` span per model, slowest first.
- **Pitfalls:** the span covers the whole model call, not the time to the first streamed token, so it's a proxy. Real TTFB needs instrumentation in the streaming layer. Reasoning models take longer for the same prompt, so compare like with like.

---

## 6. Errors

### `errors-timechart`: runs over time by agent and outcome

```kusto
requests
| where name has "invoke_agent"
| extend agent = tostring(customDimensions["gen_ai.agent.name"])
| extend agent = iff(isempty(agent), "unknown-agent", agent)
| extend outcome = iff(success == true, "success", "error")
| summarize count() by bin(timestamp, 5m), agent, outcome
| order by timestamp asc
```

- **Purpose:** see volume and failures together, so you know whether an error spike is real or just low traffic.
- **How it works:** classify each run as success or error, then count per bucket, agent and outcome.
- **Pitfalls:** `outcome` treats any non-true `success` (including missing) as an error.

### `errors-by-agent-table`: error rate per agent

```kusto
requests
| where name has "invoke_agent"
| extend agent = tostring(customDimensions["gen_ai.agent.name"])
| extend agent = iff(isempty(agent), "unknown-agent", agent)
| extend foundryProject = tostring(split(_ResourceId, "/")[8])
| summarize total = count(), failed = countif(success == false) by foundryProject, agent
| extend error_rate_pct = round(100.0 * failed / total, 2)
| order by error_rate_pct desc
```

- **Purpose:** find the worst agent first.
- **How it works:** total and failed runs per project and agent, then the failure percentage, worst first.
- **Read it as:** judge rate together with `total`. 1 failure in 2 runs is 50% but isn't an incident.
- **Note:** `agent-framework-agent-broken-model` is an intentional negative test. In the reference environment its runs report success even though the model call fails, so it shows about 0% here and appears in the [hidden failures](#9-hidden-failures) panel instead.

### `errors-top-codes-table`: failed runs by result code

```kusto
requests
| where name has "invoke_agent" and success == false
| extend agent = tostring(customDimensions["gen_ai.agent.name"])
| extend agent = iff(isempty(agent), "unknown-agent", agent)
| extend foundryProject = tostring(split(_ResourceId, "/")[8])
| summarize errors = count() by foundryProject, agent, resultCode
| order by errors desc
```

- **Purpose:** what kind of failure it is (for example 401, 404, 429, 500).
- **How it works:** keep failed runs only, count by project, agent and `resultCode`.
- **Read it as:** 401 or 403 means credentials or permissions. 404 often means a missing deployment. 429 means throttling. 5xx means a backend fault.

### `errors-4xx5xx`: client versus server errors

```kusto
requests
| where name has "invoke_agent" and success == false
| extend agent = tostring(customDimensions["gen_ai.agent.name"])
| extend agent = iff(isempty(agent), "unknown-agent", agent)
| extend error_class = case(resultCode startswith "4", "4xx (client/config error)", resultCode startswith "5", "5xx (server/backend error)", "other")
| summarize count() by agent, error_class
| order by agent asc
```

- **Purpose:** decide who should act: whoever owns the caller or configuration (4xx), or whoever owns the backend (5xx).
- **How it works:** `case` buckets the result code by its first digit, then counts per agent and class.
- **Pitfalls:** `other` collects non-HTTP codes, such as exception names or empty values.

---

## 7. Runs, tools, success and health

### `runs-sessions-table`: turns per trace (top 50)

```kusto
requests
| where name has "invoke_agent"
| extend agent = tostring(customDimensions["gen_ai.agent.name"])
| extend agent = iff(isempty(agent), "unknown-agent", agent)
| summarize turns = count(), last_seen = max(timestamp) by trace_id = operation_Id, agent
| order by turns desc
| take 50
```

- **Purpose:** find runaway loops or very long conversations.
- **How it works:** count runs per trace and agent, record the latest time, show the top 50.
- **Pitfalls:** `operation_Id` is a trace, not a conversation. It matches a conversation only if one trace spans the whole conversation.

### `tools-volume-timechart`: tool calls over time

```kusto
dependencies
| where name startswith "execute_tool"
| summarize count() by bin(timestamp, 5m)
| order by timestamp asc
```

- **Purpose:** how much the agents rely on tools. A sudden drop to zero can mean tools are no longer being called (a prompt regression), and a spike can mean a loop.

### `tools-latency-errors-table`: tool latency and errors

```kusto
dependencies
| where name startswith "execute_tool"
| extend tool = tostring(customDimensions["gen_ai.tool.name"])
| extend tool = iff(isempty(tool), name, tool)
| extend foundryProject = tostring(split(_ResourceId, "/")[8])
| summarize calls = count(), p95_ms = round(percentile(duration, 95), 1), errors = countif(success == false) by foundryProject, tool
| extend error_rate_pct = round(100.0 * errors / calls, 2)
| order by calls desc
```

- **Purpose:** find the tool that slows or breaks agents. Tool problems often show up before agent-level errors.
- **How it works:** tool name from `gen_ai.tool.name`, falling back to the span name. Then calls, p95 and error rate per project and tool.

### `success-rate-barchart`: success rate per agent

```kusto
requests
| where name has "invoke_agent"
| extend agent = tostring(customDimensions["gen_ai.agent.name"])
| extend agent = iff(isempty(agent), "unknown-agent", agent)
| summarize total = count(), success = countif(success == true) by agent
| extend success_rate_pct = round(100.0 * success / total, 2)
| order by success_rate_pct asc
```

- **Purpose:** the mirror of the error-rate table, as a chart. The worst agent is at the top.

### `health-status-table`: composite health

```kusto
requests
| where name has "invoke_agent"
| extend agent = tostring(customDimensions["gen_ai.agent.name"])
| extend agent = iff(isempty(agent), "unknown-agent", agent)
| extend foundryProject = tostring(split(_ResourceId, "/")[8])
| summarize last_seen = max(timestamp), total = count(), failed = countif(success == false), p95_ms = percentile(duration, 95) by foundryProject, agent
| extend minutes_since_last_seen = round(datetime_diff('minute', now(), last_seen), 1)
| extend error_rate_pct = round(100.0 * failed / total, 2)
| extend p95_ms = round(p95_ms, 1)
| extend Status = case(
    minutes_since_last_seen > 60, "⚪ No Recent Activity",
    error_rate_pct >= 20, "🔴 Down",
    error_rate_pct >= 5 or p95_ms > 15000, "🟡 Degraded",
    "🟢 Healthy"
  )
| project foundryProject, agent, Status, error_rate_pct, p95_ms, minutes_since_last_seen, last_seen
| extend severity_rank = case(Status startswith "🔴", 0, Status startswith "🟡", 1, Status startswith "⚪", 2, 3)
| order by severity_rank asc, error_rate_pct desc
| project-away severity_rank
```

- **Purpose:** one status per agent.
- **How it works:** per project and agent, compute last activity, error rate and p95. `case` returns the **first** rule that matches, so order matters: silence is checked first, then Down, then Degraded.

| Status | Rule |
|---|---|
| ⚪ No Recent Activity | last run more than 60 minutes ago |
| 🔴 Down | error rate at least 20% |
| 🟡 Degraded | error rate at least 5%, or p95 above 15,000 ms |
| 🟢 Healthy | none of the above |

`order by` ranks Down first, then Degraded, then No Recent Activity, then Healthy, with the highest error rate first inside each group.
- **Pitfalls:** the rules use the whole selected time range. With a 7-day range, a problem that ended yesterday still counts. An agent with little traffic can read "No Recent Activity" while being fine.
- **Tune:** change `60`, `20`, `5` and `15000` to your SLOs.

---

## 8. Cost and capacity

All three cost queries use the [price table](#the-price-table-cost-queries). They estimate list-price spend from token counts. They are not the invoice.

### `cost-timechart`: estimated spend per hour by model

```kusto
let pricing = datatable(model:string, in_per_1k:real, out_per_1k:real) [ ... ];   // see price table
dependencies
| where name startswith "chat "
| extend model = replace_regex(tostring(split(name, ' ')[1]), @'-\d{4}-\d{2}-\d{2}$', '')
| extend input_tokens = todouble(customDimensions["gen_ai.usage.input_tokens"])
| extend output_tokens = todouble(customDimensions["gen_ai.usage.output_tokens"])
| join kind=leftouter pricing on model
| extend est_cost_usd = (input_tokens / 1000.0 * in_per_1k) + (output_tokens / 1000.0 * out_per_1k)
| summarize est_cost_usd = round(sum(est_cost_usd), 4) by bin(timestamp, 1h), model
| order by timestamp asc
```

- **How it works:** the left outer join attaches prices to each call by model name. Cost is `tokens / 1000 * price` for input plus output, then summed per hour and model.
- **Pitfalls:** an unlisted model has null prices and is left out of the chart. Check `cost-table` for `price_listed = NO` and add every model you deploy.

### `cost-table`: spend totals by model

```kusto
let pricing = ... ;   // same price table
dependencies
| where name startswith "chat "
| extend model = replace_regex(tostring(split(name, ' ')[1]), @'-\d{4}-\d{2}-\d{2}$', '')
| extend input_tokens = todouble(customDimensions["gen_ai.usage.input_tokens"])
| extend output_tokens = todouble(customDimensions["gen_ai.usage.output_tokens"])
| join kind=leftouter pricing on model
| extend est_cost_usd = (input_tokens / 1000.0 * in_per_1k) + (output_tokens / 1000.0 * out_per_1k)
| extend foundryProject = tostring(split(_ResourceId, "/")[8])
| summarize calls = count(), input_tokens = sum(input_tokens), output_tokens = sum(output_tokens), est_cost_usd = round(sum(est_cost_usd), 4), price_listed = iff(max(toint(isnotnull(in_per_1k))) == 1, "yes", "NO - add to price table") by foundryProject, model
| order by est_cost_usd desc
```

- **Purpose:** which project and model spends the most. Same method as the chart, grouped per project and model for the selected range.

### `cost-budget-burndown`: month-to-date spend versus budget

```kusto
let MONTHLY_BUDGET_USD = 50.0; // set your approved monthly budget here
let pricing = ... ;   // same price table
let mtd_cost = toscalar(
  dependencies | where name startswith "chat " and timestamp > startofmonth(now())
  | extend model = replace_regex(tostring(split(name, ' ')[1]), @'-\d{4}-\d{2}-\d{2}$', '')
  | extend input_tokens = todouble(customDimensions["gen_ai.usage.input_tokens"])
  | extend output_tokens = todouble(customDimensions["gen_ai.usage.output_tokens"])
  | join kind=leftouter pricing on model
  | summarize sum((input_tokens / 1000.0 * in_per_1k) + (output_tokens / 1000.0 * out_per_1k))
);
print Metric = "Month-to-date est. spend (USD)", Value = round(mtd_cost, 4), Budget = MONTHLY_BUDGET_USD, PctOfBudget = round(100.0 * mtd_cost / MONTHLY_BUDGET_USD, 1), Status = case(mtd_cost >= MONTHLY_BUDGET_USD, "🔴 Over budget", mtd_cost >= 0.8 * MONTHLY_BUDGET_USD, "🟡 >=80% of budget", "🟢 On track")
```

- **How it works:** `toscalar` computes one number: the cost of all model calls since the first of the month. `print` builds a single result row with the budget, the percent used and a status.
- **Note:** this panel ignores the dashboard time range on purpose. It always means "this calendar month".
- **Pitfalls:** it covers only the selected projects. Set `MONTHLY_BUDGET_USD` for your team. It does not project end-of-month spend.

### `throttling-timechart`: HTTP calls versus 429s

```kusto
dependencies
| where type == "HTTP"
| summarize calls = count(), throttled_429 = countif(resultCode == "429") by bin(timestamp, 5m)
| extend throttle_rate_pct = round(iff(calls == 0, 0.0, 100.0 * throttled_429 / calls), 2)
| order by timestamp asc
```

- **Purpose:** detect quota or capacity limits. HTTP 429 means "too many requests".
- **Read it as:** occasional 429s with retries are normal. A sustained rate means raise quota or capacity, add backoff, or spread load across deployments.
- **Pitfalls:** it sees only calls recorded as `HTTP` dependencies. If the SDK retries inside the client, retried attempts can be missing.

---

## 9. Hidden failures

A run can finish "successfully" even though a model or tool call inside it failed and the agent answered anyway. Alerts on `requests.success` can't see that.

### `hidden-failures-table`: per agent

```kusto
let failed_children = dependencies
| where success == false
| project operation_Id, dep_name = replace_regex(name, @'[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}', '<id>'), dep_resultCode = resultCode;
requests
| where name has "invoke_agent" and success == true
| extend agent = tostring(customDimensions["gen_ai.agent.name"])
| extend agent = iff(isempty(agent), "unknown-agent", agent)
| extend foundryProject = tostring(split(_ResourceId, "/")[8])
| join kind=inner failed_children on operation_Id
| summarize hidden_failures = dcount(operation_Id), failed_dependencies = make_set(dep_name, 10) by foundryProject, agent
| order by hidden_failures desc
```

- **How it works:** `failed_children` lists every failed dependency with its trace ID. Successful runs are inner-joined to it on `operation_Id`. The result counts **distinct** traces per agent and lists up to 10 distinct failing dependency names.
- **Read it as:** `failed_dependencies` tells you where to look: a tool name, a model, or an external target.
- **Pitfalls:** `dcount(operation_Id)` counts each affected run once, however many children failed. Any failed dependency counts, not only model and tool calls. The failed child may belong to a different agent in the same trace.

### `hidden-failures-rate`: overall rate

```kusto
let failed_children = dependencies | where success == false | project operation_Id;
let total_success = toscalar(requests | where name has "invoke_agent" and success == true | summarize count());
let hidden = toscalar(requests | where name has "invoke_agent" and success == true | join kind=inner failed_children on operation_Id | summarize dcount(operation_Id));
print Metric = "Hidden failure rate", SuccessfulRuns = total_success, RunsWithFailedChild = hidden, HiddenFailureRatePct = iff(total_success == 0, 0.0, round(100.0 * hidden / total_success, 2))
```

- **How it works:** two scalars: all successful runs, and the **distinct** successful traces that have at least one failed child. `dcount` avoids the double counting above. The result is `hidden / total`, guarded against division by zero.
- **Read it as:** above about 1% deserves a look. Fixing or alerting on the top entry of the table usually moves it the most.

---

## 10. Operational depth

### `ops-alert-rules` (Azure Resource Graph)

```kusto
Resources
| where type =~ 'microsoft.insights/scheduledqueryrules'
| project name, severity = properties.severity, enabled = properties.enabled, description = properties.description
```

- **Purpose:** an inventory of scheduled-query alert rules, to confirm they exist and are enabled.
- **Pitfalls:** this is Resource Graph, so it follows the `Subscriptions` parameter, not the project picker, and it shows every alert rule in those subscriptions. Add `| where resourceGroup =~ '<rg>'` to narrow it. It doesn't show whether an action group is attached.

### `ops-alert-self-check`

```kusto
let err = toscalar(requests | where timestamp > ago(15m) | where name has "invoke_agent" | where success == false | summarize n = count());
let lat = toscalar(requests | where timestamp > ago(15m) | where name has "invoke_agent" | summarize p95 = percentile(duration, 95));
print AlertRule = "invoke_agent errors > 0 (last 15m)", CurrentValue = tostring(err), Threshold = "> 0", WouldFireNow = iff(err > 0, "🔴 Yes", "🟢 No")
| union (print AlertRule = "invoke_agent p95 latency > 30s (last 15m)", CurrentValue = iff(isnull(lat), "no runs", strcat(tostring(round(lat, 0)), "ms")), Threshold = "> 30000ms", WouldFireNow = iff(lat > 30000, "🔴 Yes", "🟢 No"))
```

- **Purpose:** shows whether the two deployed alert conditions would fire **right now**, without opening the alert blade.
- **How it works:** two scalars over the last 15 minutes (failed runs, and p95 duration) are compared with the same thresholds as the alert rules, producing two rows.
- **Pitfalls:** it uses a fixed `ago(15m)` and ignores the dashboard range. With no runs in the window the latency row shows "no runs" and the answer is "No". Keep the thresholds in step with the deployed alert rules.

### `ops-dependency-health`

```kusto
dependencies
| where type in ("HTTP", "AI")
| extend foundryProject = tostring(split(_ResourceId, "/")[8])
| summarize calls = count(), success_rate_pct = round(100.0 * countif(success == true) / count(), 2), p95_ms = round(percentile(duration, 95), 1) by foundryProject, type, target
| order by calls desc
```

- **Purpose:** health of what the agents call: model endpoints and web services.
- **How it works:** keep outbound HTTP and AI calls, then group by project, type and target host, with call count, success rate and p95.
- **Read it as:** a target with a falling success rate or rising p95 is the likely upstream cause. Check Azure Service Health for it.

### `ops-model-inventory` (Azure Resource Graph)

```kusto
Resources
| where type =~ 'microsoft.cognitiveservices/accounts/deployments'
| project name, model = properties.model.name, version = properties.model.version, sku = sku.name, capacity = sku.capacity, provisioningState = properties.provisioningState
```

- **Purpose:** what is deployed: model, version, SKU, capacity and state.
- **Use it for:** confirming approved models and versions, spotting models nearing retirement, and checking that capacity matches the traffic you see in the throttling panel.
- **Pitfalls:** it lists every deployment in the selected subscriptions. Narrow it with a `resourceGroup` filter.

---

## 11. Usage patterns

### `usage-conversation-length`

```kusto
requests
| where name has "invoke_agent"
| summarize turns = count() by operation_Id
| summarize sessions = count() by turns
| order by turns asc
```

- **Purpose:** a histogram of how many runs a trace contains.
- **How it works:** the first `summarize` counts runs per trace. The second counts how many traces have each run count.
- **Read it as:** a long tail at high turn counts suggests loops or runaway sessions. Open them in the Sessions panel.

### `usage-anomaly-check`

```kusto
let last24h = toscalar(requests | where timestamp > ago(1d) | where name has "invoke_agent" | summarize count());
let baseline_daily_avg = toscalar(requests | where timestamp between (ago(8d) .. ago(1d)) | where name has "invoke_agent" | summarize count()) / 7.0;
print Metric = "Agent runs: last 24h vs. prior 7-day daily average", Last24h = last24h, DailyAvg7d = round(baseline_daily_avg, 1), PctChange = iff(baseline_daily_avg == 0, 0.0, round(100.0 * (last24h - baseline_daily_avg) / baseline_daily_avg, 1))
```

- **Purpose:** flag unusual traffic.
- **How it works:** runs in the last day, versus the average per day over the 7 days before it, and the percent change.
- **Pitfalls:** it ignores weekday patterns, so Monday versus a weekend baseline can look like a spike. It needs 8 days of retained data. With no baseline the change shows 0.

---

## 12. Telemetry health

Check these first. If they're bad, the other panels can't be trusted.

### `telemetry-ingestion-freshness`

```kusto
union requests, dependencies
| summarize last_ingested = max(timestamp)
| extend minutes_since_last_event = round(datetime_diff('minute', now(), last_ingested), 1)
| extend Status = iff(minutes_since_last_event > 60, "🟡 Stale (>60min since last event)", "🟢 Fresh")
```

- **How it works:** the newest timestamp across both tables, converted to minutes ago, with a stale flag at 60 minutes.
- **Pitfalls:** a quiet system with no traffic also looks stale. Check whether traffic is expected before treating it as an outage. The Application Insights daily cap stops ingestion and looks the same.

### `telemetry-trace-completeness`

```kusto
requests
| where name has "invoke_agent"
| extend has_child = iff(operation_Id in ((dependencies | project operation_Id)), true, false)
| summarize total = count(), with_children = countif(has_child == true)
| extend trace_completeness_pct = iff(total == 0, real(null), round(100.0 * with_children / total, 2))
```

- **Purpose:** checks that runs have child spans (model or tool calls). Without them, tokens, cost, tool and hidden-failure panels are incomplete.
- **How it works:** for each run, test whether any dependency shares its `operation_Id`, then compute the percent that do.
- **Pitfalls:** an agent that legitimately makes no model call would lower the value, but that is rare. With no runs, the result is null. The `in` subquery is limited in size (about one million values), so on very large data sets use a shorter time range.

---

## 13. Using the queries outside the workbook

- **Application Insights:** open the resource, then **Logs**, paste a query, and set the time range in the picker. The queries start from `requests` and `dependencies`, so they run as written.
- **Log Analytics workspace:** if the resource is workspace-based, the tables are `AppRequests` and `AppDependencies`, with columns such as `Name`, `Success`, `DurationMs`, `OperationId` and `Properties`. The queries need translating.
- **Resource Graph panels** run in **Azure Resource Graph Explorer**, not in Logs.
- **Alerts:** to alert from a panel, copy the query into a scheduled-query rule and add a threshold. Use the `ops-alert-self-check` logic as the starting point.
- **Scripts and APIs:** the same KQL runs through the Logs Query API (for example the `azure-monitor-query` Python package, `LogsQueryClient.query_resource`) against an Application Insights resource id.
