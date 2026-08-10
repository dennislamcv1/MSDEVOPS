# Observability, Tracing, and Automated Rollback Design
### Meridian Systems — Platform Engineering
**Author:** Lead Platform Engineer
**Audience:** Platform team (implementation) and engineering leadership (design review)
**Status:** For review

---

## 1. Executive Summary
*(145 words)*

Meridian Systems runs four product lines on a shared platform with no common observability layer. Pipeline metrics sit in disconnected tools, flaky tests bury real failures in alert noise, and no production request can be traced back to the pipeline run or commit that changed it. Latency regressions reach us as user complaints, and every post-incident review becomes an archaeology exercise.

This design specifies one stack: OpenTelemetry instrumentation, Azure Monitor and Application Insights collection, Log Analytics as the query surface, and Grafana Cloud as the presentation and alerting layer. It adds DORA dashboards, flaky test quarantine, 95% trace coverage, and automated rollback.

P99 latency is the experience of the slowest one percent of requests — on a shared platform, usually the largest tenant running the heaviest job. Alerting on P99 converts silent degradation into a paged signal and moves detection from hours of complaints to minutes.

---

## 2. Observability Stack Architecture
*(279 words, excluding diagram)*

**Instrumentation.** Every service is instrumented with the OpenTelemetry SDK for traces, metrics, and logs. The Azure Pipelines build stage stamps three resource attributes into the container image environment — `service.version`, `deployment.commit_sha`, and `deployment.run_id` — so telemetry is bound to its pipeline run before the container ever starts.

**Collection.** Pods export OTLP to a node-local OpenTelemetry Collector agent, which forwards to a gateway Collector deployment. The gateway handles tail sampling, attribute scrubbing, and fan-out. Its Azure Monitor exporter writes traces, requests, dependencies, and exceptions to Application Insights.

**Storage.** Application Insights is workspace-based, so all telemetry lands in a single Log Analytics workspace as `requests`, `dependencies`, `traces`, `exceptions`, and `customMetrics`. Pipeline events are pushed separately: an Azure DevOps service hook fires on build and release completion into a Logic App, which posts through the Logs Ingestion API and a data collection rule into the custom table `DeploymentEvents_CL`. Test results land in `TestRuns_CL` the same way. Because deployment records and runtime telemetry share one workspace and one correlation key (`deployment.run_id` plus `deployment.commit_sha`), a single KQL query can join a production request to the commit that shipped it.

**Presentation.** Grafana Cloud connects through the Azure Monitor data source, authenticated by an Entra ID service principal holding Monitoring Reader on the subscription and Log Analytics Reader on the workspace. That one data source exposes three query modes: Azure Monitor Metrics for platform-level metrics, Log Analytics for KQL across both product tables and custom tables, and Application Insights Traces for span-level drill-down. Dashboard panels issue KQL directly against the workspace; Grafana alert rules evaluate the same queries and route to the rollback webhook.

### 2.1 Architecture diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│ (1) BUILD & RELEASE — Azure Pipelines                                      │
│     • injects OTEL_RESOURCE_ATTRIBUTES into the image:                     │
│         service.version | deployment.run_id | deployment.commit_sha        │
│     • service hook on build/release completion ──────────────┐             │
└───────────────────────┬──────────────────────────────────────│─────────────┘
                        │ immutable image (provenance baked in)│ JSON event
                        ▼                                      ▼
┌───────────────────────────────────────────┐      ┌────────────────────────┐
│ (2) RUNTIME — services + OTel SDK         │      │ Logic App →            │
│     traces / metrics / logs (OTLP)        │      │ Logs Ingestion API     │
│     W3C traceparent propagation           │      │ + Data Collection Rule │
└───────────────────────┬───────────────────┘      └───────────┬────────────┘
                        │ OTLP                                 │
                        ▼                                      │
┌───────────────────────────────────────────┐                  │
│ (3) OTel Collector: agent → gateway       │                  │
│     tail sampling · scrubbing · fan-out   │                  │
│     Azure Monitor exporter                │                  │
└───────────────────────┬───────────────────┘                  │
                        ▼                                      │
┌───────────────────────────────────────────┐                  │
│ (4) Application Insights (workspace-based) │                 │
│     fixed-rate ingestion sampling @ 95%   │                  │
│     100% retention: failures / 5xx / >P99 │                  │
└───────────────────────┬───────────────────┘                  │
                        ▼                                      ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ (5) LOG ANALYTICS WORKSPACE  — single query surface                        │
│     requests · dependencies · traces · exceptions · customMetrics          │
│     DeploymentEvents_CL · TestRuns_CL · IncidentEvents_CL                  │
│     join key: deployment.run_id + deployment.commit_sha                    │
└───────────────────────┬────────────────────────────────────────────────────┘
                        │ Azure Monitor data source (Entra ID service principal:
                        │ Monitoring Reader + Log Analytics Reader)
                        ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ (6) GRAFANA CLOUD                                                          │
│     DORA dashboard panels (KQL) · trace drill-down · alert rules           │
└───────────────────────┬────────────────────────────────────────────────────┘
                        │ P99 breach > 10%
                        ▼
        Azure Monitor action group → on-call page + Teams post
                        │
                        └──► HMAC-signed webhook ──► Rollback pipeline (§5.2)
```

---

## 3. KQL Metrics and Dashboard Design
*(191 words, excluding queries)*

Every panel answers one operational question and carries one DORA label.

**Panel 1 — Deployment Frequency.** *Question: how often does each product line reach production, and is that rate improving?* `DeploymentEvents_CL` is filtered to successful production releases and summarised into daily bins by product line.

**Panel 2 — Lead Time for Changes.** *Question: how long does a merged commit wait before it serves traffic?* The query subtracts commit timestamp from deployment completion time and reports the 50th and 95th percentiles.

**Panel 3 — Change Failure Rate.** *Question: what share of deployments degrade production?* Deployments are joined to rollback and incident events inside a 60-minute window and expressed as a percentage.

**Panel 4 — Mean Time to Recovery (MTTR).** *Question: once degradation is detected, how fast do we restore service?* The query averages the interval between detection and confirmed recovery.

**Panel 5 — P99 latency by release (supports MTTR).** *Question: which release moved the tail?* Requests are bucketed by `deployment.run_id` and P99 duration is compared to the prior release.

**Panel 6 — Flaky test rate (supports Change Failure Rate).** *Question: how much of our pipeline signal is noise?* Flagged tests are divided by total tests.

### 3.1 Panel queries

```kql
// Panel 1 — Deployment Frequency
DeploymentEvents_CL
| where Environment_s == "production" and Status_s == "succeeded"
| summarize Deployments = count() by bin(TimeGenerated, 1d), ProductLine_s
```

```kql
// Panel 2 — Lead Time for Changes
DeploymentEvents_CL
| where Environment_s == "production" and Status_s == "succeeded"
| extend LeadTimeMin = datetime_diff('minute', DeployCompletedAt_t, CommitTimestamp_t)
| summarize p50 = percentile(LeadTimeMin, 50),
            p95 = percentile(LeadTimeMin, 95)
          by bin(TimeGenerated, 1d), ProductLine_s
```

```kql
// Panel 3 — Change Failure Rate
let failureWindow = 60m;
let deploys =
    DeploymentEvents_CL
    | where Environment_s == "production" and Status_s == "succeeded"
    | project RunId_s, ProductLine_s, DeployedAt = TimeGenerated;
let failures =
    DeploymentEvents_CL
    | where Outcome_s in ("rolled_back", "incident_declared")
    | project RunId_s, FailedAt = TimeGenerated;
deploys
| join kind=leftouter failures on RunId_s
| extend Failed = iff(isnotempty(FailedAt) and FailedAt - DeployedAt <= failureWindow, 1, 0)
| summarize ChangeFailureRatePct = round(100.0 * sum(Failed) / count(), 2)
          by bin(DeployedAt, 7d), ProductLine_s
```

```kql
// Panel 4 — Mean Time to Recovery
IncidentEvents_CL
| where Severity_s in ("Sev1", "Sev2")
| extend RecoveryMin = datetime_diff('minute', RecoveredAt_t, DetectedAt_t)
| summarize MTTR_Minutes = avg(RecoveryMin),
            p90_Minutes  = percentile(RecoveryMin, 90)
          by bin(TimeGenerated, 7d), ProductLine_s
```

```kql
// Panel 5 — P99 latency by release
requests
| where timestamp > ago(24h)
| extend RunId     = tostring(customDimensions["deployment.run_id"]),
         CommitSha = tostring(customDimensions["deployment.commit_sha"])
| summarize P99Ms = percentile(duration, 99)
          by bin(timestamp, 5m), cloud_RoleName, RunId, CommitSha
```

```kql
// Panel 6 — Flaky test rate
TestRuns_CL
| where TimeGenerated > ago(30d)
| summarize Runs = count(), Failures = countif(Result_s == "failed") by TestId_s
| where Runs >= 50
| extend FailureRatePct = 100.0 * Failures / Runs
| summarize FlaggedFlaky = countif(FailureRatePct between (5.0 .. 50.0)),
            TotalTests   = count()
| extend FlakyRatePct = round(100.0 * FlaggedFlaky / TotalTests, 2)
```

---

## 4. Flaky Test Detection and Quarantine Logic
*(188 words)*

**Detection threshold.** A test is flagged flaky when, over a rolling window of the last 50 pipeline runs on `main`, its failure rate falls between 5% and 50% while the test file and the code paths it covers are unchanged. Below 5% is treated as noise not yet worth acting on; above 50% is treated as a genuine, deterministic failure and routed to the normal bug flow rather than to quarantine. The evaluation runs nightly against `TestRuns_CL`.

**Quarantine action.** A bot opens a pull request adding the `[Quarantined]` trait to the flagged test and registers it in `quarantine-registry.yaml` with the owning team, detection date, and observed failure rate. Quarantined tests stop gating merges but still execute in a nightly non-blocking job, so their behaviour keeps being recorded. Each entry creates a work item with a 10-business-day remediation SLA.

**Exit and budget.** A quarantined test auto-restores after 50 consecutive passes. The registry carries a hard budget of 1% of total test count; when quarantine exceeds that, the pipeline blocks new quarantine pull requests and the platform team burns the list down before feature work resumes.

---

## 5. Distributed Tracing and Rollback Design
*(248 words)*

### 5.1 Build-time trace context and sampling
*(106 words)*

The build stage exports `OTEL_RESOURCE_ATTRIBUTES` carrying `service.version`, `deployment.run_id`, and `deployment.commit_sha` into the image, so every span the service emits already knows its provenance. W3C `traceparent` headers propagate that trace ID across service boundaries. Adaptive sampling in Application Insights is disabled and replaced with fixed-rate ingestion sampling at 95%, so the sampled fraction stays constant instead of throttling under load. A telemetry processor forces 100% retention for failures, 5xx responses, and requests above the current P99 threshold. A daily KQL check verifies that at least 95% of production requests resolve to both a run ID and a commit SHA.

```kql
// Daily traceability verification — target ≥ 95%
requests
| where timestamp > ago(24h)
| extend HasRun = isnotempty(tostring(customDimensions["deployment.run_id"])),
         HasSha = isnotempty(tostring(customDimensions["deployment.commit_sha"]))
| summarize TraceablePct = round(100.0 * countif(HasRun and HasSha) / count(), 2)
          by cloud_RoleName
```

### 5.2 Automated rollback mechanism
*(142 words)*

**Detection.** A Grafana alert rule compares the new release's rolling five-minute P99 against its 24-hour pre-deployment baseline and fires when P99 exceeds that baseline by more than 10% across three consecutive one-minute evaluations within 30 minutes of release.

**Alert routing.** The rule notifies an Azure Monitor action group that pages on-call, posts failing trace exemplars to the platform Teams channel, and calls an HMAC-signed webhook on the rollback pipeline.

**Pipeline action.** The rollback stage redeploys the previous known-good image digest, shifts all traffic back to it, and freezes the release branch.

**Confirmation.** The pipeline holds until P99 returns within 5% of baseline for ten consecutive minutes and error rate matches pre-deployment levels, then annotates Grafana and writes `outcome=rolled_back` to `DeploymentEvents_CL`. If confirmation fails, the deployment was not the cause and the alert escalates to an incident commander.

```kql
// Rollback detection — P99 degradation > 10% vs pre-deployment baseline
let candidateRun = "<deployment.run_id>";
let baseline =
    requests
    | where timestamp between (ago(25h) .. ago(1h))
      and tostring(customDimensions["deployment.run_id"]) != candidateRun
    | summarize BaselineP99 = percentile(duration, 99) by cloud_RoleName;
requests
| where timestamp > ago(5m)
  and tostring(customDimensions["deployment.run_id"]) == candidateRun
| summarize CurrentP99 = percentile(duration, 99) by cloud_RoleName
| join kind=inner baseline on cloud_RoleName
| extend DegradationPct = round(100.0 * (CurrentP99 - BaselineP99) / BaselineP99, 2)
| where DegradationPct > 10.0
```

---

## 6. Reflection
*(144 words)*

The hardest call was how much rollback authority to automate. The safe option was a paged human approval gate: an engineer reviews the P99 comparison and clicks rollback. It is auditable, and it never reverts a healthy release because of a noisy neighbour or a traffic spike. But it reintroduces the exact delay this design exists to remove — median page-to-acknowledgement on this team is around eight minutes, which is roughly the whole blast radius of a bad deploy on a shared platform.

I chose full automation, bounded rather than unlimited. The trigger only arms for 30 minutes after a release, requires three consecutive breaches rather than one, and reverts to an image digest already proven in production. The cost I accepted is occasional unnecessary rollbacks; on this platform a needless revert is cheap and a slow one is not.

---

## Appendix A — Terminology and conventions

| Term | Definition as used in this document |
|---|---|
| P99 latency | Response time at the 99th percentile; the slowest 1% of requests |
| `deployment.run_id` | Azure Pipelines build ID, injected as an OTel resource attribute |
| `deployment.commit_sha` | Git commit SHA, injected as an OTel resource attribute |
| Traceability | Share of production requests resolvable to both a run ID and a commit SHA; target ≥ 95% |
| Flaky test | Test with a 5–50% failure rate over the last 50 `main` pipeline runs, with no change to the test or covered code |
| Baseline | 24-hour pre-deployment P99 for the same service |
| Known-good digest | Immutable image digest of the last release that passed post-deploy confirmation |
| DORA metrics | Deployment Frequency, Lead Time for Changes, Change Failure Rate, Mean Time to Recovery (MTTR) |
