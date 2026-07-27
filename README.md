# Fraud Detection Event Processor

**Enterprise-grade real-time financial fraud detection platform** — 12 backend microservices, a Python ML inference engine, and a full observability stack, wired together over Kafka.

![Java](https://img.shields.io/badge/Java-21-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3-brightgreen)
![Python](https://img.shields.io/badge/Python-3.12-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-0.115-teal)
![Kafka](https://img.shields.io/badge/Apache%20Kafka-7.6-black)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## Overview

A transaction hits the API Gateway, gets validated and deduplicated, and flows through Kafka: **8 fraud rules evaluate it → a risk score gets computed → scores ≥70 open a case → notifications fan out by severity.** In parallel, an aggregator service stitches every event for that transaction into one unified case record, and an analytics service rolls it into live daily metrics — all off the same topics, with no extra service-to-service calls.

Every item from the original build spec has been implemented: 12 backend services, a React/TypeScript dashboard, an API Gateway, Prometheus/Grafana, ELK logging, a transaction simulator, CSV/Excel/PDF reporting, and CI covering all services plus a full-stack smoke test.

**What "implemented" means here:** the Python ML service, the frontend, and the transaction simulator were actually installed, built, and run in this environment — not just written. The 11 Java services were written carefully and reviewed line-by-line but couldn't be compiled here (no Maven Central access in this sandbox — see [Known Limitations](#known-limitations)). Read that section before treating this as "production ready."

---

## Services

| Service | Port | Responsibility | Tests |
|---|---|---|---|
| API Gateway | 8080 | Single entry point — JWT validation at the edge, routing, circuit breakers, CORS | 4 |
| Auth Service | 8081 | Signup/login/refresh/logout, JWT + RBAC (Admin/Analyst/Viewer), Redis rate limiting | 7 |
| Transaction Service | 8082 | Validates + persists transactions, idempotency, publishes `transactions.created` | 5 |
| Rules Engine Service | 8083 | 8 fraud rules (velocity, card testing, impossible travel, blacklist, etc.) | 7 |
| Risk Scoring Service | 8084 | Combines rule + ML signal into an explainable 0–100 score | 5 |
| ML Inference Service (Python) | 8000 | IsolationForest + RandomForest, trained on a locally-generated synthetic dataset | 6 |
| Alert Service | 8085 | Opens fraud cases (score ≥70), full lifecycle: assign / escalate / resolve / false-positive | 7 |
| Notification Service | 8086 | Simulated Email/Slack/Webhook/SMS delivery, fanned out by risk level | 5 |
| User Service | 8087 | Profile, preferences, per-channel notification settings | 6 |
| Fraud Detection Service | 8088 | Master aggregator — merges all events into one case per transaction | 4 |
| Analytics Service | 8089 | Real-time volume + alert-rate aggregates, incrementally rolled up from Kafka | 4 |
| Audit Service | 8091 | Centralized, searchable audit log | 4 |

**64 tests across the platform.** Only the ML service's 6 were actually run (6/6 passing) — see [Known Limitations](#known-limitations) for why the Java tests are unverified rather than failing.

---

## Quick start

```bash
docker compose up --build
```

| Service | URL |
|---|---|
| API Gateway | http://localhost:8080 |
| Auth / Transaction / Rules / Risk / Alert / Notification / User / Fraud Detection / Analytics / Audit | Swagger UI on each service's port — e.g. `http://localhost:8081/swagger-ui.html` |
| ML Inference Service | http://localhost:8000/docs |
| Dashboard | http://localhost:5173 |
| Kafka UI | http://localhost:8090 |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3001 (`admin` / see `GRAFANA_ADMIN_PASSWORD` in `docker-compose.yml`) |
| Kibana | http://localhost:5601 |

### Try the full flow

```bash
# 1. Submit a transaction
curl -X POST localhost:8082/api/v1/transactions -H "Content-Type: application/json" -d '{
  "idempotencyKey": "demo-1", "userId": "11111111-1111-1111-1111-111111111111",
  "cardId": "card-1", "merchantId": "merchant-1", "currency": "USD",
  "amount": 7500.00, "countryCode": "US"
}'

# 2. Fetch its risk score (rules + risk scorer run automatically via Kafka)
curl localhost:8084/api/v1/risk-scores/<transactionId>

# 3. If the score is >= 70, check the alert that opened
curl localhost:8085/api/v1/alerts

# 4. See the simulated notifications sent for it
curl localhost:8086/api/v1/notifications/alert/<alertId>

# 5. See it as one merged case (transaction + rules + risk + alert)
curl localhost:8088/api/v1/cases/<transactionId>

# 6. Check today's aggregate numbers
curl "localhost:8089/api/v1/analytics/summary?from=$(date +%F)&to=$(date +%F)"
```

Call the ML service directly:

```bash
curl -X POST localhost:8000/api/v1/ml/predict -H "Content-Type: application/json" -d '{
  "transaction_id": "txn-1", "user_id": "user-1", "card_id": "card-1", "merchant_id": "merchant-1",
  "amount": 8500, "country_code": "KP", "user_avg_amount_30d": 45, "user_txn_count_24h": 9,
  "is_new_device": true, "is_new_merchant": true
}'
```

Download reports (from analytics-service, also reachable through the gateway):

```bash
curl "localhost:8089/api/v1/analytics/reports/volume.csv?from=2026-06-01&to=2026-06-30" -o volume.csv
curl "localhost:8089/api/v1/analytics/reports/volume.xlsx?from=2026-06-01&to=2026-06-30" -o volume.xlsx
curl "localhost:8089/api/v1/analytics/reports/summary.pdf?from=2026-06-01&to=2026-06-30" -o summary.pdf
```

### Running tests locally

```bash
# Any Java service (requires JDK 21 + Maven)
cd services/<service-name>
mvn test

# ML service (requires Python 3.12)
cd services/ml-inference-service
pip install -r requirements.txt
pytest tests/ -v
```

> This sandbox has no access to Maven Central, so the Java builds haven't been compiled/tested here — only reviewed for correctness. `docker compose up --build` will resolve dependencies from Maven Central the first time you run it on your machine. PyPI *is* reachable here, so the ML service has been genuinely installed and tested. If a Java service fails to compile, share the error and it'll get fixed immediately.

---

## Roadmap

- [x] **Phase 1** — Infra (Postgres/Redis/Kafka) + Auth Service
- [x] **Phase 2** — Transaction, Rules Engine, Risk Scoring services
- [x] **Phase 3** — ML Inference, Alert, Notification services
- [x] **Phase 4** — Dashboard, API Gateway, Prometheus/Grafana, ELK — only remaining item: wiring Risk Scoring → ML Inference over REST
- [x] **Phase 5** — Analytics, Audit, User, Fraud Detection services; transaction simulator; CSV/Excel/PDF reports; CI expanded to all 12 services + frontend + smoke test

---

## Project structure

```
fraud-detection-platform/
├── docker-compose.yml
├── services/
│   ├── auth-service/            ← complete
│   ├── transaction-service/     ← complete
│   ├── rules-engine-service/    ← complete
│   ├── risk-scoring-service/    ← complete
│   ├── ml-inference-service/    ← complete (Python/FastAPI — actually tested)
│   ├── alert-service/           ← complete
│   ├── notification-service/    ← complete
│   ├── user-service/            ← complete
│   ├── fraud-detection-service/ ← complete (master aggregator)
│   ├── analytics-service/       ← complete
│   ├── audit-service/           ← complete (consumer + API; producers not yet wired elsewhere)
│   ├── api-gateway/             ← complete
│   └── shared-lib/              ← not built — each service's DTOs are self-contained, no shared code needed yet
├── frontend/                    ← complete (React/TS/Vite/Tailwind — built + typechecked)
├── infrastructure/
│   ├── docker/postgres/init-multi-db.sh
│   ├── monitoring/              ← Prometheus scrape config, Grafana provisioning + 2 dashboards
│   └── logging/                 ← Logstash pipeline config for the ELK stack
├── docs/
└── scripts/
    └── transaction-simulator/   ← complete (run + verified with --dry-run)
```

Reports (CSV/Excel/PDF) live inside `analytics-service` at `/api/v1/analytics/reports/*` rather than as a separate service — the aggregate data they render already lives there, so a dedicated reports-service would just be a thin proxy.

---

## Known limitations

Being upfront about the gap between this repo and a production system:

- **The smoke-test CI job wasn't run against a live GitHub Actions runner.** It brings up all 17 containers and hits every health endpoint, but was only written and YAML-validated — marked `continue-on-error: true` for exactly this reason.
- **The dashboard, transaction simulator, and Python logging were actually run and verified** (`npm run build` with zero TS errors, `simulate.py --dry-run`, `python-json-logger` executed for real). The report generation code (Apache POI / PDFBox) and Java logging/Kafka wiring were written carefully but not compiler-verified.
- **The ML service logs structured JSON to stdout but doesn't ship to Logstash directly**, unlike the Java services (which use `LogstashTcpSocketAppender`). Wiring it in needs either a Python TCP log handler or a Filebeat sidecar — not set up here.
- **The API Gateway's correlation-ID filter uses MDC inside a WebFlux reactive chain**, a known-imperfect pattern since MDC doesn't reliably survive thread hops across Netty's event loop. The correlation ID is still generated, forwarded via `X-Correlation-Id`, and echoed on the response — it's specifically the gateway's own log lines (not downstream services', which use plain MVC MDC and are fine) that may occasionally log without it attached.
- **Grafana dashboards use standard Micrometer/Spring Boot and kafka-exporter metric names**, not validated against a live Prometheus instance in this sandbox — double-check panel queries once you spin up the real stack.
- **Audit Service consumes `audit.events` and exposes a search API, but nothing publishes to that topic yet.** Wiring a `KafkaTemplate`-based `AuditPublisher` into the other services is a mechanical follow-up — the message contract (`AuditEventMessage`) already exists in `audit-service/src/main/java/.../dto/`.
- **Analytics can't compute a true false-positive rate.** Alert resolution/false-positive marking happens via REST only, not published to Kafka — so `alertRatePercent` is (alerts opened / transactions), not a precision/recall metric.
- **The API Gateway duplicates auth-service's JWT signing logic** (same secret, same HS256 claims shape) rather than calling auth-service per-request — a deliberate latency tradeoff, but the two must stay in sync if the token shape changes.
- **Risk Scoring → ML Inference isn't wired together yet.** The ML service works standalone; Risk Scoring currently scores off the rule engine only — the integration point is marked `TODO(Phase 3)` in `RiskScoringService.computeMlContribution()`.
- **The Java services weren't compiler-verified** in the environment that generated them (no Maven Central access). The Python ML service was installed and run for real, with passing tests. A failed Java build is a straightforward fix — open an issue with the compiler error.
- **This is a portfolio/learning project**, not something that's been security-audited or load-tested at the "1M+ transactions" scale from the original goals. Treat secrets in `docker-compose.yml` as dev-only — rotate everything before any real deployment.

---

## Consolidation notes

This repo was produced by diffing 9 overlapping checkpoint exports (`phase1` → `phase3` → `github-ready` / `github-ready__1_` → `checkpoint-3` → `checkpoint-4` → `FINAL`) file by file:

- `FINAL` and `checkpoint-4` were byte-identical except the README; both were the most complete checkpoint. This repo is `FINAL` unchanged, plus restoring a `.gitignore` that was accidentally dropped between `checkpoint-3` and `checkpoint-4`/`FINAL`.
- `checkpoint-3` was a clean subset of `FINAL` — missing correlation-ID filters, logback configs, and the analytics report module, all already forward-ported.
- `github-ready` / `github-ready__1_` turned out to be an earlier, abandoned parallel session — despite the name, it only has git history for 6 of 12 services and stubs the rest with placeholder READMEs. It wasn't used as a base; its one real improvement (a `BlacklistController` null-check) was already present in `FINAL` in equal or better form.
- Kafka topic names, event DTO shapes, gateway routes, and `docker-compose.yml` ports were all cross-checked service-by-service — every producer/consumer pair matches, every route matches its downstream controller, every port is consistent.
- Actually re-verified in this pass: `pip install -r requirements.txt && pytest tests/` for `ml-inference-service` (6/6 passing), and `npm ci && npx tsc -b && npm run build` for the frontend (clean build, zero TS errors). `mvn clean compile` on `auth-service` confirmed no Maven Central access (403 on `repo.maven.apache.org`) — consistent with the disclosure above, no new gap introduced.
- `docker compose config` and a live `docker compose up` weren't run (no Docker daemon here) — compose syntax and cross-references were checked by hand instead.

---

## Tech stack

Java 21 · Spring Boot 3 · Spring Cloud Gateway · PostgreSQL · Redis · Apache Kafka · React / TypeScript · Python (scikit-learn / FastAPI) · Docker Compose · Prometheus / Grafana · ELK

## License

MIT
