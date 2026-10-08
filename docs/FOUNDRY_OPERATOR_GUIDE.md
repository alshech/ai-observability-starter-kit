# Foundry Operator Guide

How an operator should use the [Foundry Operator Dashboard](FOUNDRY_OPERATOR_WORKBOOK.md) to run Foundry agents in production. The workbook doc explains what each panel and query does. This guide explains **when to look at it, in what order, and what to do next**.

> Thresholds below are starting points. Tune them to your own SLOs, and record the decisions in the workbook's "KPI thresholds and ownership" section.

## 1. Set up once

| Step | Action |
|---|---|
| Import | Follow [How to import it](FOUNDRY_OPERATOR_WORKBOOK.md#how-to-import-it-into-another-environment). |
| Select projects | Set `AppInsightsResources` to the Application Insights resource of each Foundry project. Select several to compare projects side by side. |
| Telemetry on | Agents must run with `ENABLE_INSTRUMENTATION=true`. Keep `ENABLE_SENSITIVE_DATA=false` in production so prompts and completions aren't stored in traces. |
| Prices and budget | Update the price `datatable` and `MONTHLY_BUDGET_USD` in the cost queries. |
| Alerts | Create scheduled-query alerts that mirror the "Live self-check" (failed runs, p95 latency). Route them to an action group with an owner and a runbook link. |
| Save | Save a copy to a shared resource group so the whole team uses one version. Pin it in the Azure portal dashboard. |
| Access | Operators need **Monitoring Reader** (or **Log Analytics Reader**) on the Application Insights resource, and **Reader** on the subscription for the Resource Graph panels. |

## 2. Trust the data first

Always check **Telemetry and evaluation health** before reading any other panel.

| Signal | Healthy | If not |
|---|---|---|
| Ingestion freshness | normally under 5 min while traffic flows. The workbook flags stale at 60 min | Check the agent is running, the connection string is set, and the Application Insights resource isn't over its daily cap. |
| Trace completeness | close to 100% | Model and tool spans are missing. Confirm `ENABLE_INSTRUMENTATION=true` and the agent framework version. Tokens, cost and hidden-failure panels are unreliable until fixed. |

## 3. Routines

### Every shift (5 minutes)
1. Set the time range to **last 24 hours**.
2. **At a glance:** runs, success rate, error rate, p95, tokens. Compare with yesterday.
3. **Health status:** every agent should be `Healthy`. Anything else, go to the triage playbook.
4. **Live self-check:** both alerts should show "not firing".
5. **Ingestion freshness** is current.

### Daily (15 minutes)
1. **Hidden failures:** successful runs with a failed model or tool call. Open the listed agents and check the failed dependency names.
2. **Error codes by agent** and **invocation errors by class**: 4xx points to callers or configuration, 5xx to the backend.
3. **Tool calls:** rising tool p95 or error rate usually precedes agent errors.
4. **429 panel:** any sustained throttling means quota or capacity work.
5. **Volume anomaly check:** a change beyond about 50% versus the 7-day average needs an explanation (release, outage, campaign).

### Weekly (30 minutes)
1. Set the range to **7 days**. Review latency and token trends per model.
2. **Cost:** month-to-date versus budget. Reconcile with Azure Cost Management, and add prices for any new model so it isn't silently dropped.
3. **Model inventory:** check versions, SKUs and capacity against what is approved. Watch for models nearing retirement.
4. **Alert rules:** confirm all are enabled and the action groups still route to the right owners.
5. **Quality and safety:** open the Foundry portal **Evaluations** and **Red team** tabs. The workbook section is static text. Run a red-team scan from the portal and record the attack success rate against your agreed limit.
6. Review long conversations in **Sessions**: very high turn counts suggest loops.

### After every release
1. Compare the 24 hours before and after: error rate, p95, tokens per run, tool error rate.
2. Run the red-team v2 gate before promoting. A failing gate blocks the release.
3. Confirm trace completeness didn't drop.

## 4. Triage playbooks

Always start from the symptom, then narrow from agent to operation to trace.

| Symptom | First panel | Likely causes | Action |
|---|---|---|---|
| Health = **No Recent Activity** | Ingestion freshness | Agent stopped, no traffic, telemetry broken | Check agent status in Foundry. If traffic is expected, check the connection string and the app logs. |
| Health = **Down** (errors 20% or more) | Top error codes, tool calls | Bad deployment, model deployment removed or throttled, auth change | Roll back the last release. Check the model inventory and the 429 panel. |
| Health = **Degraded** (errors 5% or more, or p95 over 15 s) | Model-call latency, tool latency | Slow model, slow tool, throttling | Find the slowest layer: model call versus tool call. Scale capacity or fix the slow dependency. |
| Errors mostly **4xx** | Invocation errors by class | Bad request shape, expired credentials, missing permissions | Fix the caller or configuration. Not an outage. |
| Errors mostly **5xx** | Top error codes, external dependency health | Backend or model service failing | Check Azure Service Health and the dependency table. |
| Success rate fine, **users complain** | Hidden failures | A tool failed and the agent answered anyway | Open the failed dependency names, then the trace. Add alerting on child failures. |
| **Latency up**, errors flat | End-to-end vs model vs tool latency | Larger prompts, slow tool, throttling retries | Compare tokens per run, then tool p95, then the 429 rate. |
| **Tokens or cost up**, volume flat | Token totals by model, sessions | Longer prompts, loops, a model change | Check turn counts and recent prompt or model changes. |
| **Volume drop** | Volume anomaly check | Upstream outage, routing change, auth failure | Confirm with the calling team before assuming an agent problem. |
| **Throttling (429)** | HTTP calls vs 429 | Quota or capacity too low | Raise capacity or quota, add retry with backoff, or spread load across deployments. |

### Drill down to one failed run
1. In Application Insights, open **Transaction search** and filter on `operation_Id`, or run the same KQL as the panel with a `where operation_Id == "<id>"` clause.
2. Read the span tree: `invoke_agent` (the run), `chat <model>` (model calls), `execute_tool` (tool calls).
3. Find the first failed span. That is the root cause, not the run's result code.

## 5. Suggested thresholds

These match the "KPI thresholds and ownership" table inside the workbook, so operators see one set of numbers. They are discussion starting points. Agree real values with each team's SLOs.

| KPI | Warn | Page |
|---|---|---|
| Invocation error rate | above 1% for 15 min | above 5% for 10 min with user impact |
| End-to-end latency (p95) | above 10 s for 15 min | above 30 s for 10 min, if the SLO is breached |
| Model-call latency (TTFB proxy, p95) | above 2 s for 15 min | investigate the backend or model |
| Model throttling (429s) | above 1% for 15 min | above 5% for 10 min, if retries fail |
| Tool failure rate | above 1% for 15 min | above 5%, if a critical workflow is blocked |
| Hidden failure rate | above 1% for 15 min | track as a separate recovered-failure group |
| Budget used | 80% | 100%, or a projected overrun |
| Ingestion freshness | lag above 5 min for 15 min | no traces for 10 min while traffic is expected |
| Red-team attack success rate | any new high-severity finding | release gate: no unresolved critical findings. The script's default gate is `--max-asr 0.20`. |

The workbook's **health status** colors use fixed rules: Degraded at 5% errors or p95 above 15 s, Down at 20% errors. They are display rules, not alert thresholds.

Rate and percentile alerts need at least 100 eligible events per window. Below that, alert on raw failed counts. Page only on user impact (errors, latency, down) and send the rest to a ticket queue to avoid alert fatigue.

## 6. Best practices

- **One workbook, many projects.** Use the project picker instead of copying the workbook per project.
- **Look at percentiles.** Use p95 and p99, not averages.
- **Alert on symptoms, investigate with the workbook.** The workbook is for diagnosis. Alerts should fire without anyone watching it.
- **Cover hidden failures.** Add an alert on failed child dependencies of successful runs.
- **Keep sensitive content out of telemetry.** Leave `ENABLE_SENSITIVE_DATA=false`, and apply data-retention and access controls on the Application Insights resource.
- **Treat cost as an estimate.** Reconcile with Cost Management monthly.
- **Own every alert.** Each alert needs an owner, an action group and a runbook link.
- **Version the workbook.** Keep the JSON in source control and review changes like code. Don't edit production copies by hand.
- **Run safety checks on a schedule.** Red-team before each release and at least monthly, with an ASR gate.
- **Review quarterly.** Reset thresholds and SLOs, retire stale panels, and add prices for new models.

## 7. Known limitations to tell operators about

- Cost covers only the models in the price table. Verify the prices for your region and deployment type.
- Latency is end-to-end per model call, not time to first token.
- The "Quality and safety" section isn't live.
- The hidden-failures table counts distinct runs, but any failed dependency counts, not only model and tool calls.
- Sessions are traces. A conversation that spans several traces shows as several sessions.
- When we tested, the red-team portal's **Create** picker listed prompt agents only, not hosted agents. Check whether your portal version lists your hosted agents. If it doesn't, red-team a prompt agent that uses the same model and instructions, or use the Foundry red-teaming SDK to target the hosted agent.

See [Known limitations and gotchas](FOUNDRY_OPERATOR_WORKBOOK.md#known-limitations-and-gotchas) for the full list.
