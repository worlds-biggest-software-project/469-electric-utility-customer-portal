# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Electric Utility Customer Portal · Generated: 2026-05-26

---

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the electric utility customer portal. Instead of storing only the current state, every state change is captured as an immutable event in an append-only event store. Read models (projections) are derived from the event stream and optimised for specific query patterns — billing dashboards, outage maps, NEM settlement views, and usage analytics.

This architecture is particularly well-suited to the utility domain because:

- **Billing is inherently event-driven.** Meter reads arrive, rates change, payments are processed, credits are applied, adjustments are made — each is a discrete event that must be auditable.
- **Regulatory compliance demands audit trails.** Utilities are regulated entities; the ability to reconstruct the exact state of any account at any point in time is not optional.
- **Net metering settlement requires history reconstruction.** NEM true-up calculations must replay an entire year of import/export events against time-varying rate schedules.
- **Outage management is temporal.** Outages have a lifecycle (detected → crew assigned → restoring → restored) that is naturally expressed as event sequences.
- **Multiple read patterns coexist.** The customer portal, the utility operations dashboard, the billing engine, and AI analytics all need different views of the same underlying data.

---

## Architecture

```
                    ┌─────────────────────────────────┐
                    │      Command Side (Write)        │
                    │                                  │
    API Gateway ──► │  Command Handlers / Aggregates   │
                    │  (validate, emit events)         │
                    │                                  │
                    └──────────┬──────────────────────┘
                               │
                               ▼
                    ┌─────────────────────────────────┐
                    │       Event Store (PostgreSQL)    │
                    │  Append-only, immutable events    │
                    │  Partitioned by stream_id hash    │
                    └──────────┬──────────────────────┘
                               │
                    ┌──────────▼──────────────────────┐
                    │      Event Bus / Projectors      │
                    │  (async event processing)        │
                    └──┬───────┬───────┬──────┬───────┘
                       │       │       │      │
                       ▼       ▼       ▼      ▼
                    ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
                    │Bill │ │Usage│ │Outage│ │NEM  │
                    │View │ │View │ │View  │ │View │
                    └─────┘ └─────┘ └─────┘ └─────┘
                    (Read Models / Projections)
                               │
                    ┌──────────▼──────────────────────┐
                    │      Query Side (Read)            │
                    │  Query Handlers / API endpoints   │
    API Gateway ◄── │  (serve from read models)        │
                    └─────────────────────────────────┘
```

---

## Event Store Schema

```sql
-- =============================================================
-- EVENT STORE — the single source of truth
-- =============================================================

CREATE TABLE event_store (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id           VARCHAR(255) NOT NULL,                  -- aggregate identity (e.g., 'customer:abc-123', 'bill:xyz-456')
    stream_type         VARCHAR(50) NOT NULL,                   -- aggregate type ('Customer', 'Bill', 'Outage', etc.)
    event_type          VARCHAR(100) NOT NULL,                  -- e.g., 'CustomerCreated', 'MeterReadingReceived'
    event_version       INTEGER NOT NULL,                       -- monotonic within stream
    payload             JSONB NOT NULL,                         -- event data
    metadata            JSONB NOT NULL DEFAULT '{}',            -- correlation_id, causation_id, user_id, source system
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),     -- when the event actually happened
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),     -- when it was recorded in the store
    UNIQUE (stream_id, event_version)                           -- optimistic concurrency control
);

-- Partition by stream_type for efficient aggregate replay
CREATE INDEX idx_event_store_stream ON event_store (stream_id, event_version);
CREATE INDEX idx_event_store_type ON event_store (event_type, recorded_at);
CREATE INDEX idx_event_store_time ON event_store (recorded_at);
CREATE INDEX idx_event_store_correlation ON event_store USING GIN ((metadata->'correlation_id'));

-- =============================================================
-- EVENT STORE SNAPSHOTS — periodic aggregate state snapshots
-- to avoid replaying entire event history for long-lived aggregates
-- =============================================================

CREATE TABLE event_snapshots (
    snapshot_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id           VARCHAR(255) NOT NULL,
    stream_type         VARCHAR(50) NOT NULL,
    snapshot_version    INTEGER NOT NULL,                       -- event_version at snapshot time
    state               JSONB NOT NULL,                         -- serialised aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshots_stream ON event_snapshots (stream_id, snapshot_version DESC);

-- =============================================================
-- PROJECTION CHECKPOINTS — track each projection's position in the event stream
-- =============================================================

CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(100) PRIMARY KEY,
    last_event_id       UUID REFERENCES event_store(event_id),
    last_recorded_at    TIMESTAMPTZ,
    events_processed    BIGINT DEFAULT 0,
    status              VARCHAR(20) DEFAULT 'running' CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message       TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- DEAD LETTER QUEUE — events that failed projection processing
-- =============================================================

CREATE TABLE dead_letter_events (
    id                  BIGSERIAL PRIMARY KEY,
    event_id            UUID NOT NULL,
    projection_name     VARCHAR(100) NOT NULL,
    error_message       TEXT NOT NULL,
    retry_count         INTEGER DEFAULT 0,
    max_retries         INTEGER DEFAULT 5,
    next_retry_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Event Type Catalogue

### Customer Domain Events

```sql
-- Example events stored as JSONB payloads in the event_store

-- CustomerCreated
-- stream_type: 'Customer', stream_id: 'customer:{customer_id}'
{
    "customer_id": "uuid",
    "first_name": "Jane",
    "last_name": "Smith",
    "email": "jane@example.com",
    "customer_type": "residential",
    "external_cis_id": "CIS-12345"
}

-- CustomerContactUpdated
{
    "customer_id": "uuid",
    "field": "phone_primary",
    "old_value": "555-0100",
    "new_value": "555-0200"
}

-- PaperlessBillingEnabled
{
    "customer_id": "uuid"
}

-- NotificationPreferenceChanged
{
    "customer_id": "uuid",
    "channel": "sms",
    "category": "outage",
    "enabled": true
}

-- MfaEnrolled
{
    "customer_id": "uuid",
    "mfa_method": "totp"
}

-- LoginAttempted
{
    "customer_id": "uuid",
    "success": true,
    "ip_address": "192.168.1.1",
    "anomaly_score": 0.12
}
```

### Metering Domain Events

```sql
-- MeterInstalled
-- stream_type: 'Meter', stream_id: 'meter:{meter_id}'
{
    "meter_id": "uuid",
    "location_id": "uuid",
    "meter_serial": "MTR-2026-0001",
    "meter_type": "ami",
    "is_bidirectional": true,
    "install_date": "2026-01-15"
}

-- IntervalReadingsReceived (batch event for efficiency)
-- stream_type: 'UsagePoint', stream_id: 'usage_point:{usage_point_id}'
{
    "usage_point_id": "uuid",
    "meter_id": "uuid",
    "source": "mdms",
    "readings": [
        {
            "interval_start": "2026-05-25T00:00:00Z",
            "duration_seconds": 900,
            "consumption_wh": 1250,
            "generation_wh": 0,
            "voltage_v": 121.3,
            "quality": "actual"
        },
        {
            "interval_start": "2026-05-25T00:15:00Z",
            "duration_seconds": 900,
            "consumption_wh": 1180,
            "generation_wh": 0,
            "voltage_v": 121.1,
            "quality": "actual"
        }
    ]
}

-- DailyUsageSummarised (derived event emitted by aggregation process)
{
    "usage_point_id": "uuid",
    "date": "2026-05-25",
    "total_consumption_kwh": 28.4,
    "total_generation_kwh": 12.1,
    "peak_demand_kw": 4.2,
    "on_peak_kwh": 8.1,
    "off_peak_kwh": 20.3
}
```

### Billing Domain Events

```sql
-- BillGenerated
-- stream_type: 'Bill', stream_id: 'bill:{bill_id}'
{
    "bill_id": "uuid",
    "customer_id": "uuid",
    "location_id": "uuid",
    "rate_plan_id": "uuid",
    "billing_period_start": "2026-04-01",
    "billing_period_end": "2026-04-30",
    "statement_date": "2026-05-01",
    "due_date": "2026-05-25",
    "total_kwh": 850.5,
    "line_items": [
        {"category": "energy", "description": "Tier 1 (0-500 kWh)", "quantity": 500, "rate": 0.12, "amount": 60.00},
        {"category": "energy", "description": "Tier 2 (501+ kWh)", "quantity": 350.5, "rate": 0.18, "amount": 63.09},
        {"category": "fixed", "description": "Customer Charge", "amount": 12.00},
        {"category": "tax", "description": "State Tax", "amount": 5.42}
    ],
    "total_amount_due": 140.51
}

-- BillAiExplanationGenerated
{
    "bill_id": "uuid",
    "explanation": "Your bill this month is $140.51, which is $22 higher than last month...",
    "high_bill_flag": true,
    "high_bill_reason": "Increased HVAC usage due to unusually warm weather (avg 92°F vs 85°F last year)"
}

-- PaymentReceived
-- stream_type: 'Payment', stream_id: 'payment:{payment_id}'
{
    "payment_id": "uuid",
    "customer_id": "uuid",
    "bill_id": "uuid",
    "amount": 140.51,
    "payment_method": "card",
    "card_last_four": "4242",
    "card_brand": "visa",
    "processor_txn_id": "pi_3abc123"
}

-- PaymentFailed
{
    "payment_id": "uuid",
    "customer_id": "uuid",
    "amount": 140.51,
    "failure_reason": "insufficient_funds"
}

-- PaymentRefunded
{
    "payment_id": "uuid",
    "refund_amount": 140.51,
    "reason": "duplicate_payment"
}

-- AutopayEnrolled
{
    "customer_id": "uuid",
    "payment_method": "ach",
    "payment_token": "tok_masked",
    "ach_routing_last_four": "1234"
}

-- BudgetBillingEnrolled
{
    "customer_id": "uuid",
    "monthly_amount": 125.00,
    "start_date": "2026-06-01",
    "reconciliation_date": "2027-06-01"
}
```

### Net Metering Domain Events

```sql
-- NemAccountCreated
-- stream_type: 'NemAccount', stream_id: 'nem:{nem_account_id}'
{
    "nem_account_id": "uuid",
    "customer_id": "uuid",
    "location_id": "uuid",
    "nem_programme": "nem_2",
    "system_size_kw": 7.5,
    "interconnection_date": "2025-03-15",
    "annual_cycle_start": "2025-04-01"
}

-- NemMonthlySettled
{
    "nem_account_id": "uuid",
    "settlement_month": "2026-05",
    "total_import_kwh": 420.3,
    "total_export_kwh": 580.1,
    "net_kwh": -159.8,
    "on_peak_export_kwh": 210.5,
    "off_peak_export_kwh": 369.6,
    "credit_earned": 38.72,
    "credit_used": 0,
    "credit_balance": 142.56
}

-- NemAnnualTrueUpCalculated
{
    "nem_account_id": "uuid",
    "cycle_start": "2025-04-01",
    "cycle_end": "2026-03-31",
    "total_import_kwh": 5200.4,
    "total_export_kwh": 7800.2,
    "total_credits_earned": 465.30,
    "total_charges": 312.80,
    "net_amount": -152.50,
    "nsc_payout": 152.50
}
```

### Outage Domain Events

```sql
-- OutageDetected
-- stream_type: 'Outage', stream_id: 'outage:{outage_id}'
{
    "outage_id": "uuid",
    "external_oms_id": "OMS-2026-5432",
    "outage_type": "unplanned",
    "cause": "weather",
    "start_time": "2026-05-25T14:30:00Z",
    "affected_area": {"type": "MultiPolygon", "coordinates": [...]},
    "customers_affected": 2340
}

-- OutageCrewAssigned
{
    "outage_id": "uuid",
    "crew_id": "CREW-42",
    "crew_status": "en_route",
    "estimated_restoration": "2026-05-25T18:00:00Z"
}

-- OutageEtaUpdated
{
    "outage_id": "uuid",
    "previous_eta": "2026-05-25T18:00:00Z",
    "new_eta": "2026-05-25T20:00:00Z",
    "reason": "Additional damage found requiring transformer replacement"
}

-- OutageRestored
{
    "outage_id": "uuid",
    "actual_restoration": "2026-05-25T19:45:00Z",
    "customers_restored": 2340,
    "total_duration_minutes": 315
}

-- CustomerOutageReported
{
    "report_id": "uuid",
    "customer_id": "uuid",
    "location_id": "uuid",
    "report_type": "no_power",
    "reported_via": "portal"
}

-- OutageNotificationSent
{
    "outage_id": "uuid",
    "customer_id": "uuid",
    "channel": "sms",
    "notification_type": "outage_detected",
    "message": "We have detected a power outage in your area..."
}
```

### Programme and DR Domain Events

```sql
-- ProgrammeEnrolled
-- stream_type: 'ProgrammeEnrolment', stream_id: 'enrolment:{enrolment_id}'
{
    "enrolment_id": "uuid",
    "programme_id": "uuid",
    "customer_id": "uuid",
    "location_id": "uuid",
    "programme_type": "demand_response",
    "enrolled_via": "portal"
}

-- DrEventScheduled
-- stream_type: 'DrEvent', stream_id: 'dr_event:{event_id}'
{
    "event_id": "uuid",
    "programme_id": "uuid",
    "openadr_event_id": "ADR-2026-0042",
    "event_type": "load_shed",
    "signal_type": "level",
    "signal_value": 2,
    "start_time": "2026-07-15T14:00:00Z",
    "end_time": "2026-07-15T18:00:00Z"
}

-- DrEventParticipationRecorded
{
    "event_id": "uuid",
    "customer_id": "uuid",
    "baseline_kwh": 12.5,
    "actual_kwh": 8.2,
    "reduction_kwh": 4.3,
    "incentive_earned": 8.60
}
```

### EV Domain Events

```sql
-- EvChargerRegistered
-- stream_type: 'EvCharger', stream_id: 'charger:{charger_id}'
{
    "charger_id": "uuid",
    "customer_id": "uuid",
    "location_id": "uuid",
    "charger_make": "ChargePoint",
    "charger_model": "Home Flex",
    "max_power_kw": 48.0,
    "connector_type": "nacs"
}

-- EvChargingSessionStarted
{
    "session_id": "uuid",
    "charger_id": "uuid",
    "start_time": "2026-05-25T22:00:00Z",
    "rate_period": "off_peak"
}

-- EvChargingSessionCompleted
{
    "session_id": "uuid",
    "charger_id": "uuid",
    "end_time": "2026-05-26T06:00:00Z",
    "energy_kwh": 38.4,
    "peak_power_kw": 7.2,
    "cost_estimate": 4.61
}

-- EvChargingScheduleSet
{
    "charger_id": "uuid",
    "schedule_name": "Weeknight Off-Peak",
    "day_of_week": [1,2,3,4,5],
    "start_time": "22:00",
    "end_time": "06:00",
    "prefer_off_peak": true
}
```

### Service Order and Green Button Events

```sql
-- ServiceOrderSubmitted
-- stream_type: 'ServiceOrder', stream_id: 'order:{order_id}'
{
    "order_id": "uuid",
    "customer_id": "uuid",
    "location_id": "uuid",
    "order_type": "start_service",
    "requested_date": "2026-06-01"
}

-- ServiceOrderCompleted
{
    "order_id": "uuid",
    "completed_date": "2026-06-01",
    "external_order_id": "WO-54321"
}

-- GreenButtonAuthorisationGranted
{
    "auth_id": "uuid",
    "customer_id": "uuid",
    "usage_point_id": "uuid",
    "third_party_name": "EnergyStar Portfolio Manager",
    "third_party_id": "client_abc123",
    "scope": "usage_data",
    "expires_at": "2027-05-25T00:00:00Z"
}

-- GreenButtonAuthorisationRevoked
{
    "auth_id": "uuid",
    "customer_id": "uuid",
    "third_party_id": "client_abc123",
    "reason": "customer_requested"
}
```

---

## Read Model Projections

### Customer Account Projection

```sql
-- Materialised view of current customer state, built from event replay

CREATE TABLE rm_customer_accounts (
    customer_id         UUID PRIMARY KEY,
    external_cis_id     VARCHAR(50),
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    email               VARCHAR(255),
    phone_primary       VARCHAR(20),
    customer_type       VARCHAR(20),
    status              VARCHAR(20),
    paperless_billing   BOOLEAN,
    mfa_enabled         BOOLEAN,
    mfa_method          VARCHAR(20),
    last_login_at       TIMESTAMPTZ,
    active_locations     INTEGER DEFAULT 0,
    active_programmes    INTEGER DEFAULT 0,
    has_nem_account     BOOLEAN DEFAULT FALSE,
    has_ev_charger      BOOLEAN DEFAULT FALSE,
    last_event_version  INTEGER,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_customers_email ON rm_customer_accounts (email);
```

### Billing Dashboard Projection

```sql
-- Optimised for the customer billing page: current bill, recent history, payment status

CREATE TABLE rm_billing_dashboard (
    customer_id         UUID NOT NULL,
    location_id         UUID NOT NULL,
    -- Current bill
    current_bill_id     UUID,
    current_bill_amount NUMERIC(10,2),
    current_bill_due    DATE,
    current_bill_status VARCHAR(20),
    current_bill_period VARCHAR(30),                            -- '2026-04-01 to 2026-04-30'
    ai_explanation      TEXT,
    high_bill_flag      BOOLEAN DEFAULT FALSE,
    -- Payment status
    autopay_enabled     BOOLEAN DEFAULT FALSE,
    last_payment_date   TIMESTAMPTZ,
    last_payment_amount NUMERIC(10,2),
    -- Budget billing
    budget_billing_active BOOLEAN DEFAULT FALSE,
    budget_monthly_amount NUMERIC(10,2),
    budget_balance       NUMERIC(10,2),
    -- Totals
    bills_last_12_months INTEGER,
    total_paid_12_months NUMERIC(12,2),
    avg_monthly_bill     NUMERIC(10,2),
    projected_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (customer_id, location_id)
);
```

### Usage Analytics Projection

```sql
-- Optimised for the usage dashboard: daily summaries, comparisons, TOU breakdown

CREATE TABLE rm_usage_daily (
    usage_point_id      UUID NOT NULL,
    usage_date          DATE NOT NULL,
    total_consumption_kwh NUMERIC(10,3),
    total_generation_kwh  NUMERIC(10,3),
    net_kwh             NUMERIC(10,3),
    peak_demand_kw      NUMERIC(10,3),
    on_peak_kwh         NUMERIC(10,3),
    mid_peak_kwh        NUMERIC(10,3),
    off_peak_kwh        NUMERIC(10,3),
    cost_estimate       NUMERIC(10,2),
    temperature_avg_f   NUMERIC(5,1),
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (usage_point_id, usage_date)
);

CREATE TABLE rm_usage_monthly (
    usage_point_id      UUID NOT NULL,
    billing_month       DATE NOT NULL,
    total_consumption_kwh NUMERIC(12,3),
    total_generation_kwh  NUMERIC(12,3),
    net_kwh             NUMERIC(12,3),
    peak_demand_kw      NUMERIC(10,3),
    avg_daily_kwh       NUMERIC(10,3),
    neighbour_avg_kwh   NUMERIC(12,3),
    neighbour_percentile INTEGER,
    cost_estimate       NUMERIC(10,2),
    year_over_year_pct  NUMERIC(6,2),
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (usage_point_id, billing_month)
);

-- Pre-computed hourly pattern for TOU guidance
CREATE TABLE rm_usage_hourly_pattern (
    usage_point_id      UUID NOT NULL,
    day_type            VARCHAR(10) NOT NULL,                   -- 'weekday', 'weekend'
    hour_of_day         INTEGER NOT NULL CHECK (hour_of_day BETWEEN 0 AND 23),
    avg_kwh             NUMERIC(8,3),
    rate_period         VARCHAR(20),
    avg_cost_per_kwh    NUMERIC(8,5),
    shift_savings_potential NUMERIC(8,2),                       -- estimated savings from shifting this hour
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (usage_point_id, day_type, hour_of_day)
);
```

### Outage Map Projection

```sql
-- Optimised for the real-time outage map — denormalised for fast rendering

CREATE TABLE rm_active_outages (
    outage_id           UUID PRIMARY KEY,
    external_oms_id     VARCHAR(100),
    outage_type         VARCHAR(30),
    cause               VARCHAR(50),
    cause_detail        TEXT,
    status              VARCHAR(20),
    affected_area_geojson TEXT,                                 -- GeoJSON string for map rendering
    center_lat          NUMERIC(10,7),
    center_lng          NUMERIC(10,7),
    customers_affected  INTEGER,
    start_time          TIMESTAMPTZ,
    estimated_restoration TIMESTAMPTZ,
    crew_status         VARCHAR(100),
    last_update_at      TIMESTAMPTZ,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Summary statistics for the outage dashboard header
CREATE TABLE rm_outage_summary (
    region              VARCHAR(100) PRIMARY KEY DEFAULT 'all',
    active_outages      INTEGER DEFAULT 0,
    total_customers_affected INTEGER DEFAULT 0,
    major_events        INTEGER DEFAULT 0,
    avg_restoration_minutes NUMERIC(8,1),
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### NEM Settlement Projection

```sql
-- Optimised for the net metering account page

CREATE TABLE rm_nem_dashboard (
    nem_account_id      UUID PRIMARY KEY,
    customer_id         UUID NOT NULL,
    location_id         UUID NOT NULL,
    nem_programme       VARCHAR(30),
    system_size_kw      NUMERIC(10,3),
    interconnection_date DATE,
    -- Current cycle
    cycle_start         DATE,
    cycle_months_elapsed INTEGER,
    -- Current month
    current_month_import_kwh NUMERIC(12,3),
    current_month_export_kwh NUMERIC(12,3),
    current_month_net_kwh    NUMERIC(12,3),
    current_month_credit     NUMERIC(10,2),
    -- Rolling balance
    rolling_credit_balance   NUMERIC(12,2),
    -- YTD totals
    ytd_import_kwh      NUMERIC(14,3),
    ytd_export_kwh      NUMERIC(14,3),
    ytd_credits_earned  NUMERIC(12,2),
    ytd_charges         NUMERIC(12,2),
    -- Last true-up
    last_trueup_date    DATE,
    last_trueup_amount  NUMERIC(12,2),
    next_trueup_date    DATE,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Rate Comparison Projection

```sql
-- Pre-computed rate comparison results updated when usage data or rates change

CREATE TABLE rm_rate_comparisons (
    comparison_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL,
    location_id         UUID NOT NULL,
    current_rate_plan_id UUID,
    current_rate_name   VARCHAR(200),
    current_annual_cost NUMERIC(12,2),
    compared_rate_plan_id UUID,
    compared_rate_name  VARCHAR(200),
    compared_annual_cost NUMERIC(12,2),
    annual_savings      NUMERIC(12,2),
    savings_pct         NUMERIC(6,2),
    recommendation_rank INTEGER,
    analysis_period     VARCHAR(30),                            -- '2025-06 to 2026-05'
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rm_rate_comparisons_customer ON rm_rate_comparisons (customer_id, location_id);
```

### Programme Participation Projection

```sql
CREATE TABLE rm_customer_programmes (
    customer_id         UUID NOT NULL,
    programme_id        UUID NOT NULL,
    programme_name      VARCHAR(200),
    programme_type      VARCHAR(30),
    enrolled_at         TIMESTAMPTZ,
    status              VARCHAR(20),
    -- DR-specific
    total_dr_events     INTEGER DEFAULT 0,
    total_reduction_kwh NUMERIC(12,3) DEFAULT 0,
    total_incentives    NUMERIC(10,2) DEFAULT 0,
    next_event_time     TIMESTAMPTZ,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (customer_id, programme_id)
);
```

---

## Event Processing Pipeline

### Projection Handlers (Pseudocode)

```python
# Example: Billing Dashboard Projector

class BillingDashboardProjector:
    """Listens to billing events and maintains the rm_billing_dashboard read model."""

    def handle(self, event):
        match event.event_type:

            case 'BillGenerated':
                upsert_billing_dashboard(
                    customer_id=event.payload['customer_id'],
                    location_id=event.payload['location_id'],
                    current_bill_id=event.payload['bill_id'],
                    current_bill_amount=event.payload['total_amount_due'],
                    current_bill_due=event.payload['due_date'],
                    current_bill_status='issued',
                    current_bill_period=f"{event.payload['billing_period_start']} to {event.payload['billing_period_end']}"
                )
                recalculate_12_month_totals(event.payload['customer_id'])

            case 'BillAiExplanationGenerated':
                update_billing_dashboard(
                    bill_id=event.payload['bill_id'],
                    ai_explanation=event.payload['explanation'],
                    high_bill_flag=event.payload['high_bill_flag']
                )

            case 'PaymentReceived':
                update_billing_dashboard(
                    customer_id=event.payload['customer_id'],
                    current_bill_status='paid',
                    last_payment_date=event.occurred_at,
                    last_payment_amount=event.payload['amount']
                )

            case 'AutopayEnrolled':
                update_billing_dashboard(
                    customer_id=event.payload['customer_id'],
                    autopay_enabled=True
                )

            case 'BudgetBillingEnrolled':
                update_billing_dashboard(
                    customer_id=event.payload['customer_id'],
                    budget_billing_active=True,
                    budget_monthly_amount=event.payload['monthly_amount']
                )
```

```python
# Example: Outage Map Projector

class OutageMapProjector:
    """Maintains the real-time outage map read model."""

    def handle(self, event):
        match event.event_type:

            case 'OutageDetected':
                insert_active_outage(
                    outage_id=event.payload['outage_id'],
                    outage_type=event.payload['outage_type'],
                    cause=event.payload['cause'],
                    status='active',
                    affected_area_geojson=json.dumps(event.payload['affected_area']),
                    customers_affected=event.payload['customers_affected'],
                    start_time=event.payload['start_time']
                )
                increment_outage_summary()

            case 'OutageCrewAssigned':
                update_active_outage(
                    outage_id=event.payload['outage_id'],
                    status='crew_assigned',
                    crew_status=event.payload['crew_status'],
                    estimated_restoration=event.payload['estimated_restoration']
                )

            case 'OutageRestored':
                remove_active_outage(event.payload['outage_id'])
                decrement_outage_summary()
                # Outage moves to historical outage projection
                insert_historical_outage(event.payload)
```

---

## Pros and Cons

### Pros

1. **Complete audit trail.** Every state change is permanently recorded. Regulators, auditors, and customer service agents can reconstruct the exact state of any account, bill, or NEM settlement at any past moment. This is not a bolt-on audit log — it is the system of record.

2. **Temporal queries are trivial.** "What was this customer's NEM credit balance on March 15?" requires replaying events up to that date. No temporal columns or SCD (slowly changing dimension) patterns needed.

3. **Flexible read models.** New dashboard requirements (e.g., a C&I key-accounts view, an AI training data export, or a regulatory report) can be built by creating a new projection from the existing event stream without modifying the write side.

4. **Natural fit for NEM true-up.** Annual NEM settlement is inherently a replay of 12 months of import/export events against rate schedules that may have changed mid-year. Event sourcing makes this calculation a projection rather than a complex SQL query.

5. **Resilient to bugs.** If a projection has a bug, fix the projector code and rebuild the read model from the event stream. The authoritative data (events) is never corrupted by a projection bug.

6. **Excellent for AI/ML pipelines.** The event stream is a structured, time-ordered dataset ready for ML training (usage pattern recognition, high-bill prediction, outage ETA estimation). No ETL needed — subscribe to the event stream.

7. **Decoupled scaling.** Write side (event ingestion) and read side (projections/queries) scale independently. During a major outage, the read side (outage map queries) can scale horizontally while the write side handles event ingestion at normal throughput.

### Cons

1. **Increased complexity.** Event sourcing + CQRS adds significant architectural complexity. Developers must understand event design, projection handlers, eventual consistency, and idempotency. This is a larger learning curve than a conventional CRUD application.

2. **Eventual consistency for read models.** After a payment is processed, the billing dashboard projection may take milliseconds to seconds to update. Customers might see a stale bill status briefly. This can be mitigated with synchronous projections for critical paths but adds complexity.

3. **Event schema evolution is hard.** Changing the structure of an event type after production deployment requires versioning strategies (upcasting, lazy migration). A poorly planned event schema can become a maintenance burden.

4. **High storage volume for interval data.** Meter reading events at 15-minute intervals for hundreds of thousands of meters produce massive event streams. Snapshots and stream compaction are essential, but the raw event volume is higher than a state-based model.

5. **Projection rebuild time.** If a projection needs to be rebuilt from scratch (e.g., after a bug fix), replaying months or years of events can take hours to days. Snapshots mitigate this but add another subsystem to maintain.

6. **Testing complexity.** Testing requires setting up event streams, triggering projections, and verifying read models. Integration tests are more involved than simple CRUD tests.

7. **Operational overhead.** Monitoring projection lag, managing dead letter queues, handling out-of-order events, and maintaining snapshot schedules all add operational burden.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event store** | PostgreSQL 16+ (event_store table with JSONB payloads). Alternatively, EventStoreDB for a purpose-built event store with built-in projections. |
| **Event bus** | Apache Kafka or NATS JetStream for durable, ordered event distribution to projectors |
| **Projection store** | PostgreSQL for relational read models; Redis for low-latency outage map data |
| **Snapshot store** | PostgreSQL (same instance as event store) |
| **Framework** | Axon Framework (Java/Kotlin), Marten (C#/.NET), or custom Python with asyncio |
| **Serialisation** | JSON (JSONB in PostgreSQL) with schema registry for event versioning |
| **Monitoring** | Projection lag dashboards, dead letter queue alerts, event throughput metrics |
| **Idempotency** | Deduplication by event_id at each projector |

---

## Migration and Scaling Considerations

### Migrating from a CRUD System

1. **Dual-write period.** Run the existing CRUD system alongside the event store. Every state change writes to both. The event store gradually becomes the authoritative source.
2. **Event generation from existing data.** Generate synthetic "initial state" events from the current database to bootstrap the event stream. E.g., for each existing customer, emit a `CustomerCreated` event backdated to their account creation date.
3. **Projection parity testing.** Verify that projections rebuilt from the event stream produce identical results to the existing database. This is the acceptance criterion for migration.

### Scaling the Event Store

- **Partitioning.** Partition the event_store table by recorded_at (monthly) for time-range queries and by stream_type for aggregate-specific replays.
- **Archival.** Move events older than the retention window to cold storage (S3/Glacier) while keeping the projection state current. Snapshots enable aggregate reconstruction without full replay.
- **Kafka for distribution.** Use Kafka topics partitioned by stream_id to distribute event processing across multiple projector instances. Each projector consumer group handles a subset of aggregates.

### Handling Interval Data Volume

The highest-volume event type is `IntervalReadingsReceived`. Strategies to manage volume:

- **Batch events.** Group 96 readings (24 hours at 15-minute intervals) into a single event per meter per day, reducing event count by 96x.
- **Separate stream.** Use a dedicated Kafka topic for meter reading events with higher throughput settings and dedicated projector instances.
- **Hybrid approach.** Store raw interval readings in a specialised time-series store (TimescaleDB) and emit summarised events (daily/monthly aggregates) into the main event stream. This gives time-series query performance while maintaining event sourcing semantics for business events.

---

## Summary

The event-sourced / CQRS model is the most architecturally ambitious option. It delivers unmatched auditability, temporal query capability, and flexibility for evolving read patterns — all critical for a regulated utility portal. The trade-off is significant implementation and operational complexity. This model is best suited for teams with event sourcing experience, deployments requiring rigorous regulatory compliance, or portals where the AI/ML pipeline benefits from a structured event stream. For teams without this experience, consider the normalized relational model (Suggestion 1) with a separate audit log table as a simpler alternative that still meets basic compliance needs.
