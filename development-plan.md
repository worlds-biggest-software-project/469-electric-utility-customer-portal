# Electric Utility Customer Portal — Phased Development Plan

> Project: 469-electric-utility-customer-portal · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into concrete technology decisions, an architecture, and a sequence of buildable phases. The MVP scope follows the "Must-have" list in `features.md`; v1.1 and backlog features map to later phases.

---

## Core Requirements (Synthesis)

- **What it does**: An open-source, AI-enhanced self-service portal that gives residential and commercial electricity customers transparent access to usage data, billing/payments, outage status, net metering, rate analysis, programme enrolment, and EV charging — sitting on top of existing CIS, MDMS, and OMS back-office systems.
- **Who uses it**: (1) Residential/commercial **customers** self-serving via web; (2) **utility staff** viewing engagement and outage analytics; (3) **third-party aggregators** consuming Green Button CMD data with customer authorisation.
- **Key differentiators**: Open standards (Green Button/NAESB ESPI, OpenADR, IEC CIM, OpenEI URDB); AI-native bill explanations, high-bill root-cause analysis, rate optimisation, and outage ETA prediction shipped in-product rather than as paid SaaS add-ons; integrated NEM settlement with transparent true-up; deployable by small co-ops/munis without enterprise licensing.
- **Deployment model**: Self-hosted / cloud / hybrid via Docker + Docker Compose (and Helm for k8s). Must scale the read path independently for 10× storm-event traffic spikes.
- **Integration surface**: CIS (customer/billing), MDMS (interval meter data), OMS (outage), payment processor (PCI-scoped, tokenised), OpenEI URDB rates API, OpenADR VTN, OCPP Central System, SMS/email providers, Keycloak (identity/MFA).
- **Standards compliance**: NAESB REQ.21 ESPI v4.0 (Green Button CMD), OAuth 2.0 + PKCE (RFC 6749/6750), OpenID Connect, OpenAPI 3.1, JSON Schema 2020-12, OpenADR 2.0b (path to 3.0), OCPP 2.1, IEC 61968 CIM semantics, PCI DSS 4.0.1, NIST SP 800-63B AAL2, OWASP Top 10, WCAG 2.1 AA, TLS 1.3.

### Data Model Decision

Adopt **Suggestion 1 (Normalized PostgreSQL)** as the transactional system of record, augmented with **TimescaleDB (from Suggestion 4)** as a hypertable for the high-volume `interval_readings` time-series — this is exactly the hybrid path Suggestion 1's own summary recommends for production. PostGIS handles outage/service-location geospatial queries. CQRS/event-sourcing (Suggestion 2) is explicitly **not** adopted to keep operational complexity tractable; auditability is met via an append-only `audit_log` + `login_audit_log` and TimescaleDB retention. JSONB columns (a nod to Suggestion 3) are used selectively for variable-shape data (rate-plan rules, AI insight payloads, raw integration messages) without abandoning relational integrity.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Backend language | **Python 3.12** | AI/ML workloads (bill explanation, disaggregation, ETA prediction) dominate the differentiators; mature libraries (OpenLEADR for OpenADR, lxml for ESPI Atom/XML, scikit-learn/PyTorch). |
| API framework | **FastAPI** | Native OpenAPI 3.1 + JSON Schema 2020-12 generation (required by standards.md), Pydantic v2 validation, async I/O for high concurrency during storm spikes. |
| ASGI server | **Uvicorn** behind **Gunicorn** workers | Production-grade async serving; horizontal worker scaling. |
| Transactional DB | **PostgreSQL 16 + PostGIS 3.4** | Suggestion 1 baseline: ACID for billing/payments, FK integrity, RLS for customer isolation, GiST spatial index for outage maps. |
| Time-series store | **TimescaleDB 2.x** (PG extension) | Hypertable + native compression + continuous aggregates for 48M+ rows/day AMI interval data (Suggestion 4); avoids a separate datastore. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Typed models, async support, versioned schema migrations. |
| Cache / queue broker | **Redis 7** | Session/rate-limit/outage-map-tile cache; Celery broker. |
| Task queue | **Celery** (Redis broker) + **Celery Beat** | Async webhooks, AI generation, MDMS batch ingest, NEM settlement jobs, notification fan-out, scheduled true-ups. |
| Identity & MFA | **Keycloak** | OIDC provider, TOTP/WebAuthn MFA (NIST AAL2), password-reset flows, brokered SSO; portal is OIDC client + OAuth resource server. |
| LLM access | **Provider-abstracted client** (OpenAI/Anthropic/local Ollama via a `LLMClient` interface) | Bill explanations & high-bill analysis; self-hosters can point at a local model. No PII leaves deployment if local model used. |
| Frontend | **Next.js 15 (React, TypeScript, App Router)** | SSR for fast first paint + SEO of public outage map; mature a11y ecosystem for WCAG 2.1 AA; deploys static + API-backed. |
| UI components | **Radix UI + Tailwind** | Accessible primitives (focus management, ARIA) reduce WCAG risk. |
| Mapping | **MapLibre GL JS** | Open-source (no token lock-in) vector outage map; clustering for storm scale. |
| Charts | **Recharts / visx** | Accessible usage/NEM charts with data-table fallbacks. |
| Payments | **Tokenised processor (Stripe-style hosted fields) via adapter** | Keeps cardholder data out of portal scope per PCI DSS 4.0.1; W3C Payment Request API on mobile. |
| Containerisation | **Docker + Docker Compose** (dev/self-host), **Helm chart** (k8s) | Matches README self-host/cloud/hybrid goal. |
| Backend testing | **pytest + pytest-asyncio + testcontainers** | Real Postgres/Timescale/Redis in integration tests; mocks for external APIs. |
| Frontend testing | **Vitest + Testing Library + Playwright + axe-core** | Unit/component + e2e + automated WCAG 2.1 AA checks. |
| Lint / format / types | **Ruff + Black + mypy** (Python); **ESLint + Prettier + tsc** (TS) | Standard ecosystem quality gates. |
| Package managers | **uv** (Python), **pnpm** (Node) | Fast, reproducible installs. |
| API docs | **OpenAPI 3.1** auto-generated + **Redoc** | Standards.md requirement; enables SDK generation and aggregator integration. |
| Observability | **OpenTelemetry → Prometheus + Grafana**, structured JSON logs | Projection/queue lag, storm-traffic dashboards. |
| Key libraries | `openleadr` (OpenADR 2.0b VEN), `lxml` (ESPI Atom/XML), `authlib` (OAuth/OIDC), `shapely`/`geoalchemy2` (geo), `httpx` (async clients), `tenacity` (retries) | Domain-specific per standards.md. |

### Project Structure

```
electric-utility-customer-portal/
├── README.md
├── LICENSE
├── docker-compose.yml                 # postgres+timescale, redis, keycloak, api, worker, web
├── docker-compose.dev.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── deploy/
│   └── helm/                          # k8s chart
├── backend/
│   ├── pyproject.toml                 # uv-managed; ruff/black/mypy config
│   ├── alembic.ini
│   ├── migrations/                    # Alembic versioned migrations
│   ├── src/
│   │   └── eucp/
│   │       ├── main.py                # FastAPI app factory, OpenAPI metadata
│   │       ├── config.py              # Pydantic Settings (env-driven)
│   │       ├── db/
│   │       │   ├── session.py         # async engine, session, RLS context
│   │       │   ├── base.py
│   │       │   └── models/            # SQLAlchemy models (one module per domain)
│   │       │       ├── customer.py
│   │       │       ├── metering.py
│   │       │       ├── usage.py
│   │       │       ├── billing.py
│   │       │       ├── payment.py
│   │       │       ├── nem.py
│   │       │       ├── outage.py
│   │       │       ├── programme.py
│   │       │       ├── ev.py
│   │       │       ├── greenbutton.py
│   │       │       └── ai_insight.py
│   │       ├── schemas/               # Pydantic request/response models
│   │       ├── api/
│   │       │   ├── deps.py            # auth deps, current_customer, pagination
│   │       │   └── routes/           # one router per domain
│   │       ├── services/              # business logic (rating engine, NEM, etc.)
│   │       ├── integrations/
│   │       │   ├── cis.py             # CIS adapter (abstract + impls)
│   │       │   ├── mdms.py            # meter data ingest
│   │       │   ├── oms.py             # outage feed adapter
│   │       │   ├── payment.py         # tokenised processor adapter
│   │       │   ├── openei.py          # URDB rates client
│   │       │   ├── openadr.py         # OpenLEADR VEN wrapper
│   │       │   ├── ocpp.py            # OCPP Central System client
│   │       │   └── notifier.py        # email/SMS adapters
│   │       ├── ai/
│   │       │   ├── client.py          # LLMClient interface + providers
│   │       │   ├── bill_explainer.py
│   │       │   ├── high_bill.py
│   │       │   ├── rate_optimiser.py
│   │       │   └── outage_eta.py
│   │       ├── greenbutton/           # ESPI Atom/XML serialisation
│   │       ├── workers/               # Celery tasks + beat schedule
│   │       └── security/              # OAuth scopes, RLS helpers, anomaly scoring
│   └── tests/
│       ├── unit/
│       ├── integration/
│       ├── e2e/
│       └── fixtures/                  # sample ESPI XML, OMS feeds, MDMS batches
├── frontend/
│   ├── package.json                   # pnpm
│   ├── app/                           # Next.js App Router pages
│   ├── components/
│   ├── lib/                           # API client (generated from OpenAPI), auth
│   └── tests/                         # vitest + playwright + axe
└── docs/
    ├── api/                           # exported OpenAPI 3.1 spec
    └── integration/                   # CIS/MDMS/OMS adapter contracts
```

The structure is grouped by concern (models, routes, services, integrations, ai) so each phase adds modules without restructuring.

---

## Phase 1: Foundation, Schema & Auth

### Purpose
Establish the runnable skeleton: containerised Postgres+TimescaleDB+PostGIS, Redis, Keycloak, a FastAPI app with health checks and OpenAPI, the core relational schema via Alembic, and OIDC-based authentication with RLS-enforced customer isolation. After this phase, a customer can log in (MFA), and every subsequent phase can rely on `current_customer` and a migrated schema.

### Tasks

#### 1.1 — Containerised dev environment & app skeleton
**What**: `docker-compose` stack plus a FastAPI app factory exposing `/healthz`, `/readyz`, and `/openapi.json`.

**Design**:
- `docker-compose.yml` services: `db` (image `timescale/timescaledb-ha:pg16` with PostGIS), `redis`, `keycloak`, `api`, `worker`, `web`.
- `config.py` using Pydantic `BaseSettings`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://redis:6379/0"
      keycloak_issuer: str
      keycloak_audience: str = "eucp-portal"
      llm_provider: Literal["openai", "anthropic", "ollama", "none"] = "none"
      llm_api_key: str | None = None
      openei_api_key: str | None = None
      environment: Literal["dev", "staging", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="EUCP_")
  ```
- App factory `create_app() -> FastAPI` sets OpenAPI 3.1 metadata (title, version, servers), mounts routers, adds CORS, request-ID middleware, and exception handlers returning RFC 7807 problem+json.
- `/readyz` checks DB and Redis connectivity.

**Testing**:
- `Unit: create_app() returns FastAPI instance with openapi_version == "3.1.0"`.
- `Integration (testcontainers): GET /healthz → 200 {"status":"ok"}`.
- `Integration: /readyz with DB down → 503`.
- `E2E: docker compose up → all services healthy within 90s` (CI smoke).

#### 1.2 — Core schema migrations (customer, location, metering, security domains)
**What**: Alembic migrations creating the customer/auth, service-location, meter, and usage-point tables from Suggestion 1, with PostGIS and TimescaleDB extensions enabled.

**Design**:
- First migration enables extensions: `CREATE EXTENSION IF NOT EXISTS postgis; CREATE EXTENSION IF NOT EXISTS timescaledb;`.
- Implement `customers`, `customer_auth_metadata`, `login_audit_log`, `notification_preferences`, `service_locations` (with `geom GEOMETRY(Point,4326)` + GiST index), `meters`, `usage_points` exactly per Suggestion 1 §1–2.
- SQLAlchemy 2.0 typed models mirror DDL; `geom` via `geoalchemy2.Geometry`.
- Generic `audit_log` table: `(log_id BIGSERIAL, actor_id UUID, action VARCHAR, entity_type VARCHAR, entity_id UUID, before JSONB, after JSONB, created_at TIMESTAMPTZ)` for compliance.
- Enable RLS on `customers` and customer-scoped tables; policy `USING (customer_id = current_setting('eucp.customer_id')::uuid)`.

**Testing**:
- `Integration: alembic upgrade head then downgrade base round-trips cleanly`.
- `Integration: inserting service_location with point geometry; ST_DWithin query returns it`.
- `Integration: RLS — session A (customer_id=X) cannot SELECT customer_id=Y rows`.
- `Unit: SQLAlchemy model column types match migration DDL` (reflection check).

#### 1.3 — OIDC authentication, MFA, and session middleware
**What**: Authenticate browser sessions against Keycloak via OAuth 2.0 authorization-code-with-PKCE; enforce AAL2 MFA; populate `current_customer` and set RLS context per request.

**Design**:
- `authlib` validates the Keycloak-issued JWT (RS256, issuer + audience checks, exp/nbf).
- `deps.current_customer(token) -> Customer`: maps `keycloak_user_id` (token `sub`) → `customer_auth_metadata` → `customers`; sets `SET LOCAL eucp.customer_id` on the DB session.
- MFA enforced via Keycloak required-action; portal verifies `acr`/`amr` claim indicates MFA (`amr` contains `mfa`/`otp`).
- `record_login(event_type, ip, user_agent, anomaly_score)` writes `login_audit_log`.
- Endpoints: `GET /auth/me`, `POST /auth/logout`.
- OAuth scopes defined: `usage:read`, `billing:read`, `billing:write`, `outage:write`, `account:write`, `greenbutton:read`.

**Testing**:
- `Unit: valid JWT with mfa amr → Customer returned`.
- `Unit: token without MFA amr → 403 mfa_required`.
- `Unit: expired/invalid-audience token → 401`.
- `Integration (mock Keycloak JWKS): GET /auth/me with valid token → customer profile`.
- `Integration: request sets eucp.customer_id; RLS blocks cross-customer read`.

---

## Phase 2: Account Management & CIS Integration

### Purpose
Give customers a working account hub — profile, contact details, notification preferences, service locations, and start/stop/transfer service orders — backed by a pluggable CIS adapter so the portal stays in sync with the back-office system of record. This is the table-stakes layer every competitor ships.

### Tasks

#### 2.1 — CIS integration adapter
**What**: Abstract `CISAdapter` interface plus a default REST implementation and a fixture-backed `MockCISAdapter` for self-hosters without a live CIS.

**Design**:
```python
class CISAdapter(Protocol):
    async def get_customer(self, external_cis_id: str) -> CISCustomer: ...
    async def list_service_locations(self, external_cis_id: str) -> list[CISLocation]: ...
    async def submit_service_order(self, order: ServiceOrderRequest) -> CISOrderResult: ...
    async def get_bill(self, external_bill_id: str) -> CISBill: ...
```
- REST impl uses `httpx.AsyncClient` with `tenacity` retry (exp backoff, max 3) and circuit breaker; OAuth2 client-credentials to CIS.
- Sync job (`workers/cis_sync.py`) upserts customer + locations by `external_cis_id`; conflicts resolved CIS-wins, writes `audit_log`.

**Testing**:
- `Unit: REST adapter maps CIS JSON → CISCustomer dataclass`.
- `Unit (mock httpx): 503 from CIS retried 3x then raises CISUnavailable`.
- `Integration: cis_sync upserts new customer, updates changed phone, records audit row`.

#### 2.2 — Profile & notification preference APIs
**What**: CRUD for contact details, paperless billing, and per-channel/per-category notification preferences.

**Design**:
- `GET/PATCH /account/profile` (scope `account:write` for PATCH).
- `GET/PUT /account/notifications` → list of `{channel, category, enabled}` validated against the `notification_preferences` CHECK enums.
- Pydantic `ProfileUpdate` (partial), `NotificationPreference`.
- Email/phone changes that affect billing/contact are forwarded to CIS via adapter; local row updated optimistically with `pending_cis` flag.

**Testing**:
- `Unit: PATCH with invalid email → 422 with field path`.
- `Unit: PUT notifications with unknown category → 422`.
- `Integration: PATCH profile updates row + enqueues CIS sync`.
- `E2E: customer toggles paperless billing; reflected on reload`.

#### 2.3 — Service orders (start/stop/transfer)
**What**: Submit and track service orders, mirrored to CIS.

**Design**:
- `service_orders` table per Suggestion 1 §9.
- State machine: `submitted → scheduled → in_progress → completed` (terminal) and `submitted → rejected | cancelled`.
- `POST /service-orders {order_type, location_id, requested_date}` → validates, persists, calls `CISAdapter.submit_service_order`, stores `external_order_id`.
- `GET /service-orders` (customer-scoped), `GET /service-orders/{id}`.
- Status updates arrive via CIS webhook → `POST /webhooks/cis/service-order` (HMAC-verified) → transition + notification.

**Testing**:
- `Unit: illegal transition completed→submitted rejected`.
- `Unit: transfer_service requires both from/to location`.
- `Integration (mock CIS): POST start_service → external_order_id stored, status scheduled`.
- `Integration: CIS webhook with bad HMAC → 401, no transition`.

---

## Phase 3: Meter Data Ingestion & Usage Dashboard

### Purpose
This is the data backbone and the first headline customer feature. Ingest AMI interval data from MDMS into a TimescaleDB hypertable, roll it up into daily/monthly summaries with TOU bucketing, and expose the usage dashboard (hourly/daily/monthly charts, prior-period and neighbour comparisons). Everything downstream — rating, NEM, AI insights — depends on this data being correct and queryable.

### Tasks

#### 3.1 — Interval reading hypertable & MDMS ingest
**What**: TimescaleDB hypertable for `interval_readings` plus an idempotent batch ingest pipeline from MDMS.

**Design**:
- `interval_readings` per Suggestion 1 §3 but created as a hypertable: `SELECT create_hypertable('interval_readings','interval_start', chunk_time_interval => INTERVAL '7 days');`. Enable compression on chunks older than 30 days; retention policy 5 years.
- `MDMSAdapter.fetch_intervals(since, until) -> AsyncIterator[IntervalBatch]`; ingest task does `COPY`-style bulk insert with `ON CONFLICT (usage_point_id, interval_start) DO UPDATE` keyed by a unique index, so re-delivered batches are idempotent.
- Data-quality: rows tagged `reading_quality` (`actual|estimated|validated|edited`); gaps recorded; portal surfaces a "data current as of" timestamp (research.md 24h-delay requirement).

**Testing**:
- `Integration: ingest 96 readings/day for 7 days → 672 rows; re-ingest same batch → still 672 (idempotent)`.
- `Integration: query last-24h intervals for usage_point < 50ms with index`.
- `Unit: gap detection flags missing 15-min slots`.
- `Integration: compression policy compresses a 31-day-old chunk`.

#### 3.2 — Continuous aggregates (daily/monthly summaries with TOU)
**What**: Daily and monthly summary tables with TOU on/mid/off/super-off-peak buckets, peak demand, and degree-day fields.

**Design**:
- TimescaleDB continuous aggregates over `interval_readings` producing `daily_usage_summaries` and `monthly_usage_summaries` (Suggestion 1 §3), refreshed on a schedule.
- TOU bucketing keyed off the customer's active `rate_schedule_periods` (Phase 4 dependency is soft — default flat bucketing until a TOU plan is enrolled).
- Neighbour benchmark: `neighbour_avg_kwh`/`neighbour_percentile` computed by a Celery Beat job grouping similar `premise_type` + climate zone + `square_footage` band; k-anonymity floor of 15 homes.

**Testing**:
- `Integration: aggregate of known interval set → expected daily kWh and peak_demand`.
- `Unit: TOU bucketing assigns 14:00 weekday reading to on_peak per schedule`.
- `Integration: neighbour benchmark suppressed when cohort < 15 homes`.

#### 3.3 — Usage dashboard API & frontend
**What**: Usage endpoints and the customer usage dashboard UI.

**Design**:
- `GET /usage/intervals?location_id&from&to&resolution=15min|hour` (paginated, capped range).
- `GET /usage/daily?...`, `GET /usage/monthly?...` (from summaries; fast).
- `GET /usage/comparison?period=month` → current vs prior period vs neighbour percentile.
- Frontend `/usage` page: Recharts line/bar with accessible `<table>` fallback, period selector, "data as of" badge, comparison cards.

**Testing**:
- `Unit: range > 90 days at 15min resolution → 422 range_too_large`.
- `Integration: GET /usage/monthly returns 12 points for a year of summaries`.
- `E2E (Playwright + axe): /usage renders chart, passes WCAG 2.1 AA scan, table fallback present`.

---

## Phase 4: Rating Engine, Billing & Payments

### Purpose
Deliver electronic bill presentment with itemised breakdown, online payment (card/ACH), autopay, and budget billing — PCI DSS 4.0.1 compliant via tokenisation. Includes the rate-plan/tariff model and a rating engine that prices interval usage, which is reused by NEM (Phase 6) and rate comparison (Phase 7).

### Tasks

#### 4.1 — Rate plan model & rating engine
**What**: Rate-plan/tariff schema and an engine that prices a usage profile against a plan.

**Design**:
- `rate_plans`, `rate_schedule_periods`, `customer_rate_enrolments` per Suggestion 1 §4; `rate_plans.rules JSONB` holds jurisdiction-specific quirks (Suggestion 3 influence).
- `RatingEngine.price(usage: UsageProfile, plan: RatePlan, period) -> RatedResult` supporting `flat | tiered | tou | demand | nem_tou`. Returns line items: energy (per tier/period), demand, fixed, with quantities/rates.
- Pure, deterministic, no I/O — directly unit-testable.

**Testing**:
- `Unit: tiered plan, 850 kWh → tier1 500@$0.12 + tier2 350.5@$0.18` (matches Suggestion 2 example).
- `Unit: TOU plan buckets on/off-peak kWh to correct rates`.
- `Unit: demand plan applies $/kW to peak_demand_kw`.
- `Unit: zero usage → only fixed charge`.

#### 4.2 — Bill presentment
**What**: Bill + line-item storage and presentment APIs; bills sourced from CIS or generated by the rating engine.

**Design**:
- `bills`, `bill_line_items` per Suggestion 1 §4 (incl. `ai_explanation`, `ai_high_bill_flag` columns, populated in Phase 8).
- `GET /bills?location_id` (paginated, newest first), `GET /bills/{id}` (with line items), `GET /bills/{id}/pdf` (signed URL).
- Bill ingest: CIS adapter `get_bill` → upsert by `external_bill_id`; or internal generation job using RatingEngine over the billing period.

**Testing**:
- `Unit: bill serialiser includes ordered line_items`.
- `Integration: CIS bill upsert idempotent by external_bill_id`.
- `Integration: GET /bills/{id} returns RLS-scoped bill only`.

#### 4.3 — Payments, autopay & budget billing (PCI-scoped)
**What**: Tokenised card/ACH payment, autopay enrolment, and budget-billing/payment plans.

**Design**:
- `payments`, `autopay_enrolments`, `payment_plans` per Suggestion 1 §4 — **no raw PAN stored**; only `payment_token`, `card_last_four`, `card_brand`.
- Payment flow: frontend uses hosted fields → token; `POST /payments {bill_id, amount, token}` → `PaymentAdapter.charge(token, amount)` → record result; processor webhook `POST /webhooks/payment` (HMAC) confirms async settlement, transitions `pending→completed|failed`, updates bill status.
- Autopay: `POST /account/autopay {token, method}`; Celery Beat job charges on due date.
- Budget billing: `POST /account/budget-billing {monthly_amount, start_date}`; annual reconciliation job updates `balance`.
- Idempotency-Key header on `POST /payments` to prevent double-charge.

**Testing**:
- `Unit: amount <= 0 → 422`.
- `Unit: no raw card fields accepted by PaymentRequest schema`.
- `Integration (mock processor): charge success → payment completed, bill paid`.
- `Integration: duplicate Idempotency-Key → single charge`.
- `Integration: payment webhook bad HMAC → 401, status unchanged`.

---

## Phase 5: Outage Reporting, Map & Notifications

### Purpose
Ship outage self-reporting, the real-time public outage map (the highest-traffic surface during storms), OMS integration that exposes only customer-relevant data (NERC CIP boundary), and the email/SMS notification pipeline used across the portal. The read path here must scale independently (Redis-cached map tiles/GeoJSON).

### Tasks

#### 5.1 — OMS integration & outage model
**What**: Read-only OMS adapter feeding `outage_events`; correlation of customer reports to events.

**Design**:
- `outage_events`, `customer_outage_reports`, `outage_notifications` per Suggestion 1 §6 (PostGIS `affected_area_geom`).
- `OMSAdapter.poll_active() -> list[OutageEvent]` (read-only; no SCADA access — standards.md NERC CIP note). Celery Beat polls every 30s; upserts by `external_oms_id`; emits status-change notifications.
- Correlation: customer report point-in-polygon (`ST_Contains`) against active outages → link `customer_outage_reports.outage_id` or create a candidate cluster.

**Testing**:
- `Unit: report inside affected_area_geom links to existing outage`.
- `Unit: report with no matching outage → status received, unlinked`.
- `Integration: OMS poll upserts event, transition active→restored emits restored notification`.
- `Security: OMS adapter exposes no SCADA fields` (contract test on adapter output schema).

#### 5.2 — Outage reporting API & public map
**What**: Customer outage reporting and the cached public outage map.

**Design**:
- `POST /outages/report {location_id, report_type, description}` (scope `outage:write`).
- `GET /outages/active.geojson` — **public**, cached in Redis with 15s TTL, served as GeoJSON FeatureCollection (id, status, cause, customers_affected, ETR, crew_status); clustered server-side at low zoom.
- `GET /outages/summary` → counts for dashboard header (`rm_outage_summary` equivalent computed live + cached).
- Frontend `/outages` page: MapLibre GL map with clustering, report form, status panel.

**Testing**:
- `Integration: GET /outages/active.geojson valid FeatureCollection, served from cache on 2nd call`.
- `Integration: 1000 concurrent map requests served from cache without DB hit` (load smoke).
- `E2E (axe): /outages map page passes WCAG 2.1 AA; report form keyboard-navigable`.

#### 5.3 — Notification pipeline (email/SMS)
**What**: Channel-abstracted notification delivery respecting customer preferences.

**Design**:
- `Notifier` adapters: `EmailNotifier`, `SMSNotifier` (pluggable providers).
- `send_notification(customer_id, category, template, context)` checks `notification_preferences`, renders template, enqueues Celery task, records `outage_notifications`/`audit_log` with delivery status (`pending→sent→delivered|failed|bounced`).
- Outage events, billing events, high-usage alerts all route through this.

**Testing**:
- `Unit: customer with sms outage disabled → no SMS enqueued, email still sent if enabled`.
- `Integration (mock provider): send → status delivered recorded`.
- `Integration: provider failure → status failed, retried via Celery`.

---

## Phase 6: Net Metering & Green Button CMD API

### Purpose
Add the underserved NEM differentiator (generation vs consumption, credit balance, monthly settlement, transparent annual true-up) and the standards-mandated Green Button Connect My Data API (NAESB REQ.21 ESPI v4.0) enabling customer-authorised third-party data sharing. Reuses the rating engine and interval data.

### Tasks

#### 6.1 — NEM accounts, settlement & true-up
**What**: NEM account model plus monthly settlement and annual true-up calculations.

**Design**:
- `nem_accounts`, `nem_monthly_settlements`, `nem_annual_trueups` per Suggestion 1 §5.
- `NemSettlementService.settle_month(nem_account, month)`: sums import/export from `interval_readings` (using `generation_wh`), buckets exports by TOU period, applies `nem_export_rate`, computes `credit_earned`, rolls `credit_balance`.
- `compute_trueup(nem_account, cycle)`: replays the 12-month cycle, applies time-varying rate schedules, produces `net_amount` and `nsc_payout` (negative net → NSC). Celery Beat runs settlement monthly and true-up on `annual_cycle_start` anniversary.
- `GET /nem/dashboard?location_id`, `GET /nem/settlements`, `GET /nem/trueups`.

**Testing**:
- `Unit: month with export > import → credit_earned > 0, net_kwh negative` (matches Suggestion 2 example values).
- `Unit: true-up over surplus year → nsc_payout > 0, net_amount negative`.
- `Integration: settlement reads interval generation_wh correctly`.
- `Unit: rate change mid-cycle applied to correct sub-periods`.

#### 6.2 — Green Button CMD API (ESPI)
**What**: NAESB REQ.21 ESPI v4.0 Connect My Data API with OAuth-authorised third-party access, Atom/XML serialisation.

**Design**:
- `green_button_authorisations` per Suggestion 1 §9.
- ESPI resources via `lxml`: `UsagePoint`, `MeterReading`, `IntervalBlock`, `IntervalReading` as Atom feed entries.
- Endpoints (TLS 1.3, OAuth bearer, scope `greenbutton:read`):
  - `GET /espi/1_1/resource/Subscription/{id}/UsagePoint` → Atom feed.
  - `GET /espi/1_1/resource/UsagePoint/{id}/MeterReading/{mid}/IntervalBlock` → interval data XML.
- Authorisation flow: third party registered as OAuth client; customer grants via `POST /greenbutton/authorisations`; revoke `DELETE /greenbutton/authorisations/{id}`.
- Validate output against Green Button Alliance sandbox / ESPI XSD.

**Testing**:
- `Unit: IntervalBlock serialiser produces ESPI-schema-valid XML` (XSD validation).
- `Integration: authorised third party token → 200 Atom feed; revoked token → 401`.
- `Integration: customer A's token cannot read customer B's UsagePoint`.
- `Fixture: round-trip sample ESPI XML import → re-export matches`.

---

## Phase 7: Rate Comparison & TOU Guidance

### Purpose
Add the rate-comparison tool (OpenEI URDB) projecting cost under alternative tariffs against the customer's actual usage, plus actionable TOU appliance-shift guidance and peak-price alerts — an underserved opportunity per features.md.

### Tasks

#### 7.1 — OpenEI URDB integration & rate comparison
**What**: Pull candidate rate plans from the OpenEI Utility Rate Database and project annual cost for each against the customer's last-12-months usage.

**Design**:
- `OpenEIClient.get_rates(utility, customer_class) -> list[URDBRate]` (REST, API key; note standards.md domain migration to `developer.nlr.gov`). Cache rates in `rate_plans` with `openei_rate_id`.
- `RateComparisonService.compare(location_id)`: load 12 months of `daily_usage_summaries` → `UsageProfile`; run `RatingEngine.price` for current + each eligible plan; rank by `annual_savings`.
- `GET /rates/comparison?location_id` → ranked list with `current_annual_cost`, `compared_annual_cost`, `annual_savings`, `savings_pct`.

**Testing**:
- `Unit (mock OpenEI): URDB JSON → RatePlan + periods`.
- `Unit: comparison ranks cheaper plan first; savings_pct correct`.
- `Integration: customer with no usage history → 409 insufficient_data`.

#### 7.2 — TOU guidance & peak alerts
**What**: Hourly usage-pattern analysis with appliance-shift recommendations and peak-period high-usage alerts.

**Design**:
- Hourly pattern table (`rm_usage_hourly_pattern` shape from Suggestion 2) computed from intervals: avg kWh per hour/day-type, rate period, `shift_savings_potential`.
- `GET /usage/tou-guidance?location_id` → top shift opportunities ("shift ~3 kWh from 18:00 to 22:00 to save ~$X/mo").
- Peak alert: Celery job detects high on-peak usage vs baseline → notification via Phase 5 pipeline.

**Testing**:
- `Unit: hour with high on-peak avg + low off-peak rate → positive shift_savings_potential`.
- `Unit: flat-rate plan → no shift recommendations`.
- `Integration: on-peak spike triggers single alert (debounced per day)`.

---

## Phase 8: AI-Native Features

### Purpose
Deliver the AI-native advantage that incumbents charge for: plain-language bill explanations, high-bill root-cause analysis, AI rate-switch recommendations, and ML outage-ETA prediction. All LLM calls go through the provider-abstracted `LLMClient` so self-hosters can run a local model and keep PII in-house.

### Tasks

#### 8.1 — LLM client abstraction & bill explanation
**What**: `LLMClient` interface with OpenAI/Anthropic/Ollama providers; AI bill explanation generation.

**Design**:
```python
class LLMClient(Protocol):
    async def complete(self, system: str, user: str, *, max_tokens: int, temperature: float) -> str: ...
```
- `bill_explainer.explain(bill, prior_bill, usage, weather) -> str`. System prompt: *"You are a utility billing assistant. Explain this electricity bill in plain language at a 6th-grade reading level. Be factual, cite the numbers provided, never invent charges."* User prompt: structured bill + line items + prior-month delta + degree-day context.
- Output stored in `bills.ai_explanation`; generated by Celery task on bill ingest; guardrail validates the explanation references only provided line items (no hallucinated charges).
- `ai_insights` table (Suggestion 1 §9) stores reusable insights with `confidence_score`.

**Testing**:
- `Unit (mock LLM): explain() builds prompt containing all line items + delta`.
- `Unit: guardrail rejects output mentioning a charge not in line_items`.
- `Integration: bill ingest enqueues + stores ai_explanation`.

#### 8.2 — High-bill root-cause analysis
**What**: Detect high bills and generate an AI root-cause summary.

**Design**:
- `high_bill.analyse(current, history, usage, weather)`: flags `ai_high_bill_flag` when current bill > rolling mean + k·stddev; correlates with weather (degree days), TOU shift, new EV/NEM, or rate change; LLM produces `ai_high_bill_reason`.
- Surfaced on bill detail and as an `ai_insight` of type `high_bill_explanation`.

**Testing**:
- `Unit: bill within normal range → high_bill_flag false`.
- `Unit (mock LLM): summer spike with high CDD → reason cites cooling/weather`.
- `Integration: flagged bill creates ai_insight row`.

#### 8.3 — AI rate optimiser & outage ETA prediction
**What**: Continuous rate-switch recommendations and ML-based outage restoration ETA.

**Design**:
- Rate optimiser: Celery Beat job re-runs Phase 7 comparison monthly; if best plan beats current by > threshold, creates `ai_insight` (`rate_recommendation`) + notification.
- Outage ETA: `outage_eta.predict(outage)` — gradient-boosted model (`scikit-learn`) trained on historical `outage_events` features (cause, customers_affected, weather, time-of-day, crew availability) → predicted restoration minutes; blended with OMS-provided ETR and shown with a confidence band. Training job offline; inference on `OutageDetected`.

**Testing**:
- `Unit: rate optimiser emits recommendation only above savings threshold`.
- `Unit: ETA model loads, predicts a bounded positive duration`.
- `Integration: new outage gets predicted ETA stored on outage_events`.
- `Fixture: model trained on sample dataset achieves MAE < baseline` (regression guard).

---

## Phase 9: Programmes, Demand Response & EV (OpenADR + OCPP)

### Purpose
Add programme enrolment (demand response, rebates, efficiency audits), OpenADR demand-response event handling, and EV smart-charging (rate enrolment, off-peak scheduler, OCPP charger integration) — the v1.1 differentiators tying the portal to grid programmes.

### Tasks

#### 9.1 — Programme catalogue & enrolment
**What**: Programme model and self-service enrolment.

**Design**:
- `programmes`, `programme_enrolments` per Suggestion 1 §7.
- `GET /programmes` (eligible for customer), `POST /programmes/{id}/enrol`, `DELETE` to opt out; capacity enforced via `max_enrolments`/`current_enrolments` with row-lock.
- Rebate applications and audit scheduling as enrolment subtypes.

**Testing**:
- `Unit: enrol when current==max → 409 programme_full`.
- `Unit: duplicate enrolment → 409`.
- `Integration: enrol increments counter atomically under concurrency`.

#### 9.2 — OpenADR demand-response integration
**What**: Receive DR events from a VTN via OpenADR 2.0b (OpenLEADR) and record participation.

**Design**:
- `dr_events`, `dr_event_participation` per Suggestion 1 §7.
- `openleadr` VEN registers with utility VTN; on `EiEvent` → persist `dr_events`, fan-out enrolled-customer notifications, store baseline.
- After event: `compute_participation` calculates `baseline_kwh - actual_kwh = reduction_kwh` from intervals, `incentive_earned`.
- `GET /programmes/dr/events`, `POST /programmes/dr/events/{id}/opt-out`.
- Note (standards.md): baseline on 2.0b with documented 3.0 migration path.

**Testing**:
- `Unit (mock VTN): EiEvent payload → dr_events row with correct window`.
- `Unit: participation reduction = baseline - actual`.
- `Integration: scheduled event notifies opted-in customers`.

#### 9.3 — EV chargers & smart-charging scheduler
**What**: EV charger registration (OCPP 2.1), off-peak charging schedules, session tracking.

**Design**:
- `ev_chargers`, `ev_charging_schedules`, `ev_charging_sessions` per Suggestion 1 §8.
- `OCPPClient` (Central System API) retrieves sessions, sets charging profiles; `prefer_off_peak` schedules align to the active TOU plan's off-peak window.
- EV rate-plan self-enrolment reuses Phase 4 `customer_rate_enrolments` (EV TOU plans).
- DR curtailment: a DR event can flag `was_dr_curtailed` and adjust the active charging profile.
- `GET/POST /ev/chargers`, `GET/PUT /ev/chargers/{id}/schedule`, `GET /ev/sessions`.

**Testing**:
- `Unit: schedule with prefer_off_peak aligns start to off-peak window`.
- `Unit (mock OCPP): set charging profile call formed correctly`.
- `Integration: DR event during session sets was_dr_curtailed`.

---

## Phase 10: Utility Staff Analytics, Hardening & Compliance

### Purpose
Add the utility-staff analytics dashboard (engagement, self-service adoption, outage volumes, programme enrolment) and complete the cross-cutting non-functional work: WCAG 2.1 AA audit, OWASP/security hardening, PCI scoping evidence, load testing for storm spikes, and full OpenAPI documentation. This makes the portal deployable and certifiable.

### Tasks

#### 10.1 — Utility staff analytics dashboard
**What**: Role-gated staff dashboard with engagement and operational metrics.

**Design**:
- New Keycloak role `utility_staff`; staff endpoints require it (separate from customer scopes; RLS bypass via dedicated read role on aggregate views only).
- Metrics via continuous aggregates / materialised views: self-service adoption %, payments via portal, outage report volume, programme enrolment counts, AI-insight engagement.
- `GET /admin/metrics/engagement`, `/admin/metrics/outages`, `/admin/metrics/programmes`.
- Frontend `/admin` area (separate layout).

**Testing**:
- `Unit: customer token → 403 on /admin endpoints`.
- `Integration: staff token → metrics returned; no customer PII in aggregates`.

#### 10.2 — Accessibility, security & PCI hardening
**What**: WCAG 2.1 AA conformance, OWASP Top 10 mitigations, PCI DSS 4.0.1 evidence.

**Design**:
- CI gate: Playwright + axe-core scan on every page must pass WCAG 2.1 AA (screen-reader labels, contrast, keyboard nav).
- OWASP: dependency scanning, parameterised queries (ORM), strict CSP, third-party script controls (PCI 4.0.1 §6.4.3), rate limiting (Redis), security headers, input validation via Pydantic, output via JSON Schema.
- PCI: confirm no PAN storage anywhere; produce data-flow diagram and SAQ-A scope evidence; tokenisation only.
- Login anomaly detection: `anomaly_score` model (geo/velocity/device) stored in `login_audit_log`; high score triggers step-up MFA (NIST AAL2).

**Testing**:
- `E2E (axe): all primary pages pass WCAG 2.1 AA`.
- `Security: ZAP/dependency scan in CI → no high-severity findings`.
- `Unit: anomaly score above threshold forces step-up`.
- `Audit: grep codebase confirms no PAN columns/fields`.

#### 10.3 — Load testing, OpenAPI docs & deployment artefacts
**What**: Storm-spike load tests, published OpenAPI 3.1 spec, Helm chart, runbooks.

**Design**:
- Load test (Locust/k6) simulating 10× baseline on `/outages/active.geojson` and dashboards; verify Redis cache + read replicas hold p95 < 500ms.
- Export OpenAPI 3.1 to `docs/api/openapi.json`; serve Redoc; generate the frontend TS client from it.
- Helm chart with HPA on the API/web read path; `docker-compose` for single-node self-host.
- Runbooks: backup/PITR (pgBackRest), partition/chunk retention, incident response.

**Testing**:
- `Load: 10× outage-map traffic → p95 < 500ms, error rate < 0.1%`.
- `CI: exported OpenAPI validates against OAS 3.1 schema`.
- `E2E: helm install on kind cluster → pods ready, smoke test passes`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Schema & Auth        ─── required by everything
    │
Phase 2: Account Mgmt & CIS               ─── requires 1
    │
Phase 3: Meter Data & Usage Dashboard     ─── requires 1 (uses CIS link from 2)
    │
    ├── Phase 4: Rating, Billing & Payments   ─── requires 3
    │       │
    │       ├── Phase 6: NEM & Green Button CMD   ─── requires 3,4
    │       └── Phase 7: Rate Comparison & TOU    ─── requires 3,4  (parallel with 6)
    │
    └── Phase 5: Outage, Map & Notifications  ─── requires 1,2  (parallel with 4)
            │
            └── (notification pipeline reused by 4,6,7,8,9)

Phase 8: AI-Native Features               ─── requires 4 (bill), 5 (notify), 7 (rate opt), uses 3 data
Phase 9: Programmes, DR & EV              ─── requires 4 (rates), 5 (notify); parallel with 8
Phase 10: Staff Analytics, Hardening      ─── requires all prior (final integration/compliance)
```

**Parallelism opportunities**
- After Phase 3: **Phase 4** and **Phase 5** can be developed concurrently.
- After Phase 4: **Phase 6** and **Phase 7** can be developed concurrently.
- After Phases 4, 5, 7: **Phase 8** and **Phase 9** can be developed concurrently.

**MVP cut-line**: Phases 1–5 + Phase 6.2 (Green Button) deliver the features.md "Must-have (MVP)" set. Phases 6.1, 7 are "Should-have (v1.1)". Phases 8, 9 cover the remaining v1.1/differentiators. Phase 10 is required before any production/certified deployment.

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass; coverage on new code ≥ 80%.
3. Lint and format pass: `ruff` + `black` (backend), `eslint` + `prettier` (frontend).
4. Type checking passes: `mypy` (backend), `tsc --noEmit` (frontend).
5. Alembic migrations created, reversible, and applied cleanly (`upgrade`/`downgrade` round-trip).
6. Docker images for `api`, `worker`, `web` build; `docker compose up` reaches healthy state.
7. The phase's feature works end-to-end (manual + e2e test).
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec with request/response schemas.
9. New config options documented in `config.py` and `.env.example`.
10. Customer-facing pages introduced/changed pass the axe-core WCAG 2.1 AA scan.
11. No new high-severity findings in dependency/security scan; no PAN or secrets in code.
12. RLS verified for any new customer-scoped tables (cross-customer access blocked).
