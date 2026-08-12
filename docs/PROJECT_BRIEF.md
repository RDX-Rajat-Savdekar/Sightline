> Repo: **Sightline**. Companion to `docs/SHARED_CONTEXT.md`.

# Sightline — Observability field brief

**Read first:** [`SHARED_CONTEXT.md`](./SHARED_CONTEXT.md)  
**Field:** Observability (traces + logs + metrics combined into one product)  
**Working name:** Sightline  
**Say it:** SITE-line  
**Status:** Not started  
**Priority:** First recommended *new* project (not DevStack)

---

## One-sentence pitch

Self-hostable observability backend: ingest telemetry, store it, query it, visualize it, alert on it — a credible free/open slice of Datadog/Grafana Cloud, not three separate toy apps.

---

## Paid analog / why it exists

Companies pay for Datadog, New Relic, Grafana Cloud because ingest + storage + query + UI + alerts is hard. Sightline proves Rajat can own that vertical: cardinality, sampling, write path, query path, operator UX.

---

## Depth requirement (what “done enough for resume” means)

Must combine what used to be three overlapping ideas (traces, logs, metrics) into **one** coherent system:

| Pillar | Minimum real capability |
|---|---|
| **Ingest** | OTLP-compatible (or clearly documented subset) for traces; structured log intake; metrics scrape or push |
| **Storage** | Purpose-built or well-modeled store (e.g. Postgres + column/analytics store, or ClickHouse/QuestDB/etc.) with retention tiers |
| **Sampling / volume control** | Adaptive or head/tail sampling story you can explain under load |
| **Query + UI** | Trace waterfall, log search, basic metric charts — enough to debug a real request |
| **Alerts** | Rule-based alerts on metric thresholds or error rates (email/webhook OK) |
| **Proof** | Load test: write RPS and/or p99 query latency on a stated span/log volume |

**Shallow fail:** pretty Grafana-looking UI with fake JSON, or “we store spans in a table” with no ingest protocol, no load story, no sampling.

---

## Suggested architecture (agent starting point — revise with tradeoffs)

```
Apps / SDKs → OTLP / HTTP ingest → API gateway-ish receiver
        → processors (sample, enrich, batch)
        → storage (hot + cold optional)
        → query API → React UI (waterfall, logs, metrics)
        → alert evaluator → notifier
```

**Stack bias (align with resume):** TypeScript and/or Go for ingest path; Postgres + something write-heavy friendly; Redis optional for buffers/rate limits; React UI; Docker Compose for whole stack; OpenTelemetry vocabulary in README.

---

## MVP vertical slice (ship this before expanding)

1. Generate traffic from a sample app instrumented with OTel.
2. Ingest spans → persist → list traces → open waterfall for one trace.
3. Ingest structured logs correlated by `trace_id`.
4. One metric (e.g. request count / error rate) + one alert rule.
5. Compose up + Loom walking a slow request end-to-end.
6. Document one measured number.

Then deepen: sampling strategies, retention jobs, cardinality guardrails, auth for multi-tenant projects.

---

## Resume bullets (draft — replace metrics when real)

- Built **Sightline**, a self-hostable observability stack ingesting OTLP traces/logs/metrics with sampling, retention, and alert rules.
- Designed storage/query path for trace waterfalls and log search; load-tested write path and documented p99 query latency at N spans/sec.
- Shipped operator UI to debug a request across traces and correlated logs in one workflow.

**Best resume variants:** Backend / platform, Full-stack.

---

## Interview ammo to create while building

- Why sample? Head vs tail sampling tradeoffs.
- How do you bound cardinality from custom attributes?
- Trace → log correlation model.
- What breaks first under burst ingest?
- Hot vs cold storage decision.

---

## Explicit non-goals

- Rebuilding all of Datadog (APM, RUM, synthetics, security…)
- ML anomaly theater without a working ingest/query core
- Duplicating DevStack’s “observability service” as a rename — this must be a **standalone product** with real telemetry depth

---

## Overlap warnings

- Do not start Metric-only or Log-only side repos — fold into Sightline.
- Flowpad/ArchMap-style service maps are optional later visualizations *on top of* Sightline data, not a substitute.

---

## Agent kickoff checklist

- [ ] Confirm repo name/location with user
- [ ] Scaffold Compose: ingest + DB + UI + sample app
- [ ] Define span/log schema and retention policy in writing
- [ ] Implement MVP slice above
- [ ] Load test + README tradeoffs
- [ ] Update Status + draft bullets when demoable
