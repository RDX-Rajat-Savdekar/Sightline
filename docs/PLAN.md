# Sightline — Implementation Plan

**Product:** Self-hostable observability backend (OTLP ingest → store → query → UI → alerts).  
**Bar:** Interview-proof (Compose, demo, Loom, ≥1 measured metric, defendable tradeoffs).  
**Constraint:** Build toward the **final product** continuously. No “phases.” Ship vertical capability one piece at a time; each merge leaves a runnable system closer to the pitch.

Related: [`SHARED_CONTEXT.md`](./SHARED_CONTEXT.md) · [`PROJECT_BRIEF.md`](./PROJECT_BRIEF.md) · [`../journal.md`](../journal.md)

---

## 1. Definition of done (final product)

Sightline is done for resume/demo when **all** of the following are true:

| Capability | Concrete acceptance |
|---|---|
| **Ingest** | OTLP/HTTP (documented subset) for traces + logs; metrics push or scrape for ≥1 signal |
| **Process** | Batching + head or tail sampling with a written policy; drop/sample decisions visible in logs/metrics |
| **Store** | Real persistence (Postgres + write-friendly store if needed); retention policy implemented as a job |
| **Query** | APIs: list traces, get trace (waterfall data), search logs by `trace_id` / text, query one metric series |
| **UI** | Operator app: trace list → waterfall; correlated logs; basic metric chart; alert rule CRUD or config |
| **Alerts** | Evaluator on threshold/error rate → webhook (or email) notification |
| **Demo traffic** | Sample app under Compose emitting real OTel telemetry |
| **Ops** | `docker compose up` brings full stack; README: architecture, how to run, tradeoffs |
| **Proof** | Load test script + **one honest measured number** (write RPS and/or p99 query latency at stated volume) |
| **Tests** | Automated tests on hard core (ingest parse, sampling, storage round-trip, query correctness) — not only UI smoke |

**Explicit non-goals (do not build):** full Datadog surface, RUM/synthetics/security products, ML anomaly theater, Metric-only or Log-only side repos, fake JSON dashboards.

---

## 2. Architecture (target system)

```
┌─────────────┐   OTLP/HTTP    ┌──────────────────┐
│ Sample app  │ ─────────────► │ Ingest service   │
│ (OTel SDK)  │                │ parse · sample   │
└─────────────┘                │ enrich · batch   │
                               └────────┬─────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
              ┌──────────┐       ┌──────────┐        ┌──────────┐
              │ Spans    │       │ Logs     │        │ Metrics  │
              │ store    │       │ store    │        │ store    │
              └────┬─────┘       └────┬─────┘        └────┬─────┘
                   │                  │                   │
                   └──────────────────┼───────────────────┘
                                      ▼
                               ┌──────────────┐
                               │ Query API    │
                               └──────┬───────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                                   ▼
             ┌────────────┐                      ┌────────────┐
             │ React UI   │                      │ Alert      │
             │ waterfall  │                      │ evaluator  │
             │ logs/charts│                      │ → webhook  │
             └────────────┘                      └────────────┘
```

### Locked stack decisions (revise only with a written tradeoff)

| Concern | Choice | Why |
|---|---|---|
| Language (ingest + query) | **TypeScript (Node)** | Aligns with resume / DevStack; fast iteration; strict `tsconfig` |
| Optional later | Go ingest sidecar | Only if Node write path fails load targets — do not dual-language early |
| Primary DB | **Postgres** | Traces metadata, logs index fields, alert rules, tenants/projects |
| High-volume spans/logs | **Postgres first**; add **ClickHouse** (or similar) only if load test proves need | Avoid premature dual-store complexity; document the trigger |
| Buffer / rate limit | **Redis** (optional but planned) | Ingest buffer, sampling counters, alert debounce |
| UI | **React + Vite** (or Next only if SSR is needed — prefer Vite SPA for operator UI) | Operator console, not marketing site |
| Protocol | **OTLP/HTTP** JSON or protobuf (pick one, document) | Industry vocabulary; sample app uses official OTel SDK |
| Deploy | **Docker Compose** | Interview-proof local path |
| Auth (MVP) | Single-tenant API key or none + README note; multi-tenant projects **after** core path works | Depth is telemetry, not Auth0 clone |

### Design principles

1. **One write path, one read path** — ingest never serves UI queries; query never mutates telemetry.
2. **Correlation is a first-class schema** — `trace_id`, `span_id`, `service`, `timestamp` on every signal that can carry them.
3. **Bound cardinality** — attribute keys allowlisted or hashed/truncated; document the guardrail.
4. **Fail closed on overload** — drop or sample with metrics; never unbounded memory queues.
5. **Idempotent ingest where cheap** — span id + trace id uniqueness or upsert; document duplicates.
6. **Observability of Sightline itself** — structured JSON logs from every service; internal metrics for ingest RPS / drop rate.

---

## 3. Repository layout (create as you build)

```
Sightline/
├── AGENTS.md
├── README.md                 # how to run, architecture, metrics (grows with product)
├── journal.md                # session continuity (owner + agents)
├── docs/
│   ├── SHARED_CONTEXT.md
│   ├── PROJECT_BRIEF.md
│   ├── PLAN.md               # this file
│   ├── SCHEMA.md             # span/log/metric + retention (write early, keep true)
│   └── ADRs/                 # short decision records when tradeoffs change
├── docker-compose.yml
├── .env.example
├── packages/ or services/
│   ├── ingest/               # OTLP receiver + processors
│   ├── query/                # read API (may share package with ingest initially)
│   ├── alert/                # evaluator + notifier (can start as module in query)
│   ├── ui/                   # React operator console
│   └── sample-app/           # instrumented demo traffic
├── scripts/
│   ├── load-test.ts          # write path / query latency
│   └── seed.ts               # optional
└── tests/                    # integration against Compose where needed
```

Monorepo with shared types (`@sightline/shared`) is preferred once two packages exist. Start as a single Node service + UI if that ships faster — **split only when process boundaries earn their keep** (ingest scale vs query).

---

## 4. Data model (write `docs/SCHEMA.md` before bulk coding)

### Spans / traces

- Trace: `trace_id`, root service, start/end, span count, error flag, duration
- Span: `trace_id`, `span_id`, `parent_span_id`, name, kind, service, start, duration, status, attributes (bounded JSON), events

### Logs

- `timestamp`, `severity`, `body`, `service`, `trace_id?`, `span_id?`, attributes (bounded)

### Metrics

- Name, labels (bounded set), timestamp, value, type (counter/gauge)
- Start with: `http.server.request.count`, `http.server.errors` (or equivalent from sample app)

### Alerts

- Rule: metric name + threshold + window + notifier URL; last-fired; enabled flag

### Retention

- Hot retention default (e.g. 7 days spans/logs); delete/archive job; document cold path as “delete first, S3 later”

Indexes: `(trace_id)`, `(service, start_time DESC)`, logs `(trace_id)`, `(timestamp DESC)` + optional `tsvector`/ILIKE for MVP search.

---

## 5. Build order (piece by piece → final product)

Work top to bottom. Each item should leave `compose up` (or a documented subset) runnable. **Do not** batch as “Phase A/B/C.” Check boxes in this file or journal as you go.

### Foundations

- [ ] Scaffold monorepo/tooling: package manager (pnpm), TypeScript strict, ESLint/Prettier, shared `tsconfig`
- [ ] `docker-compose.yml`: Postgres (+ Redis when needed), placeholders for services
- [ ] `.env.example` + README “how to run” stub
- [ ] Write `docs/SCHEMA.md` (tables, retention, cardinality rules)
- [ ] Migrations (e.g. Drizzle or Prisma — pick one and stick)

### Ingest core

- [ ] HTTP server: health + OTLP traces endpoint (documented subset)
- [ ] Parse → normalize → persist spans + trace rollup
- [ ] OTLP logs endpoint → persist with `trace_id` correlation
- [ ] Batching writer (flush on size/time)
- [ ] Head sampling (configurable ratio) + structured log of sample decisions
- [ ] Ingest metrics: received / dropped / sampled / write latency

### Query core

- [ ] `GET /traces` (filter: service, time range, error)
- [ ] `GET /traces/:traceId` (ordered spans for waterfall)
- [ ] `GET /logs?trace_id=` and basic text/service filter
- [ ] `GET /metrics/query` for the one/two series used by UI + alerts
- [ ] Integration tests: ingest fixture → query returns correlated data

### Sample app + Compose path

- [ ] Minimal Node/Express (or Fastify) app with OTel SDK exporting to ingest
- [ ] Intentional slow + error routes for demo narrative
- [ ] Compose wires sample-app → ingest → DB → query

### Operator UI

- [ ] Trace list (time range, service, errors)
- [ ] Trace detail waterfall (parent/child, duration bars, status)
- [ ] Side panel or tab: logs for that `trace_id`
- [ ] Metric chart page for request/error rate
- [ ] Alert rule form + list (CRUD against API)

### Alerts

- [ ] Evaluator loop (poll metric query on interval)
- [ ] Webhook notifier + debounce / cooldown
- [ ] Test: breach threshold → webhook received (mock server in test)

### Hardening toward resume bar

- [ ] Retention job (scheduled delete by age)
- [ ] Cardinality guardrails enforced in ingest
- [ ] Tail sampling **or** document why head-only + when you’d add tail
- [ ] Load test script + capture measured number in README
- [ ] Architecture diagram + tradeoffs section in README
- [ ] Loom script: slow request → trace → logs → alert
- [ ] Draft 3–4 resume bullets with **real** numbers only
- [ ] Update `PROJECT_BRIEF.md` Status when demoable

### Optional deepeners (only after bar above)

- Protobuf OTLP, multi-tenant projects + API keys, ClickHouse hot path, cold storage, service map viz, auth UI

---

## 6. Engineering practices (always)

| Practice | How it applies here |
|---|---|
| **Vertical slices** | Prefer “one signal end-to-end” over horizontal layers of stubs |
| **Schema first** | Migrations + `SCHEMA.md` before UI polish |
| **Contract tests** | Golden OTLP payloads in fixtures; ingest → DB → query assertions |
| **ADRs** | Short note when changing store, sampling, or protocol |
| **Secrets** | Never commit `.env`; Compose uses env files |
| **CI** | Lint + unit/integration on PR once repo has tests |
| **Honest metrics** | Load numbers only from scripts you ran; date + hardware noted |
| **README as product** | Runner can demo without reading the brief |
| **Journal** | Update `journal.md` each significant session |

### Testing strategy

1. **Unit:** sampling math, attribute truncation, OTLP normalize
2. **Integration:** Compose Postgres + ingest + query with fixture payloads
3. **Load:** separate script; not a gate on every commit, but required before “shipped”
4. **UI:** light Playwright or manual Loom path — systems tests > pixel tests

### API style

- REST/JSON for query + config; version prefix `/v1`
- Stable error shape: `{ error: { code, message } }`
- Pagination on list endpoints from day one (`limit`/`cursor` or `offset`)

---

## 7. Interview ammo (create while building)

Capture answers in README or `docs/` as you hit them:

- Why sample? Head vs tail tradeoffs for this stack
- How cardinality is bounded on custom attributes
- Trace ↔ log correlation model (`trace_id` propagation)
- What breaks first under burst ingest (memory, Postgres WAL, disk)
- Hot retention vs delete vs cold storage decision

---

## 8. Resume bullets (template — fill metrics when real)

- Built **Sightline**, a self-hostable observability stack ingesting OTLP traces/logs/metrics with sampling, retention, and alert rules.
- Designed storage/query path for trace waterfalls and log search; load-tested write path and documented p99 query latency at N spans/sec.
- Shipped operator UI to debug a request across traces and correlated logs in one workflow.

---

## 9. Current state

| Item | Status |
|---|---|
| Repo seeded (briefs, README, agents entry) | Done |
| Code / Compose / schema | Not started |
| Plan + journal | Done (this session) |

**Next concrete work:** scaffolding (tooling + Compose Postgres) and `docs/SCHEMA.md`, then ingest traces end-to-end.
