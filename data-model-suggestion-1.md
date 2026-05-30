# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Electric Utility Customer Portal · Generated: 2026-05-26

---

## Overview

This model uses a fully normalized (3NF) relational schema in PostgreSQL, mapping every domain entity to its own table with explicit foreign-key relationships, check constraints, and indexes. The design aligns with the IEC 61968 Common Information Model (CIM) entity semantics where applicable, mapping CIM concepts like UsagePoint, MeterReading, and ServiceLocation to concrete relational tables.

PostgreSQL is chosen for its mature ecosystem, strong ACID compliance, advanced indexing (B-tree, GIN, GiST for geospatial), native partitioning, row-level security, and broad adoption in utility IT environments.

---

## Schema Design

### 1. Customer and Account Domain

```sql
-- =============================================================
-- CUSTOMER IDENTITY
-- =============================================================

CREATE TABLE customers (
    customer_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_cis_id     VARCHAR(50) UNIQUE,                     -- ID in upstream CIS system
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    email               VARCHAR(255) UNIQUE NOT NULL,
    phone_primary       VARCHAR(20),
    phone_secondary     VARCHAR(20),
    preferred_language  VARCHAR(10) DEFAULT 'en',
    customer_type       VARCHAR(20) NOT NULL CHECK (customer_type IN ('residential', 'commercial', 'industrial', 'agricultural')),
    status              VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'closed', 'suspended')),
    paperless_billing   BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_email ON customers (email);
CREATE INDEX idx_customers_cis_id ON customers (external_cis_id);
CREATE INDEX idx_customers_status ON customers (status);

-- =============================================================
-- AUTHENTICATION AND SECURITY
-- (Keycloak handles primary auth; this table stores portal-specific security metadata)
-- =============================================================

CREATE TABLE customer_auth_metadata (
    customer_id         UUID PRIMARY KEY REFERENCES customers(customer_id),
    keycloak_user_id    UUID UNIQUE NOT NULL,                   -- Keycloak subject identifier
    mfa_enabled         BOOLEAN DEFAULT FALSE,
    mfa_method          VARCHAR(20) CHECK (mfa_method IN ('totp', 'sms', 'email', 'webauthn')),
    last_login_at       TIMESTAMPTZ,
    last_login_ip       INET,
    failed_login_count  INTEGER DEFAULT 0,
    locked_until        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE login_audit_log (
    log_id              BIGSERIAL PRIMARY KEY,
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    event_type          VARCHAR(30) NOT NULL CHECK (event_type IN ('login_success', 'login_failure', 'logout', 'mfa_challenge', 'mfa_success', 'mfa_failure', 'password_reset', 'account_locked')),
    ip_address          INET,
    user_agent          TEXT,
    geo_location        VARCHAR(100),
    anomaly_score       NUMERIC(5,4),                           -- ML anomaly detection score
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_login_audit_customer ON login_audit_log (customer_id, created_at DESC);

-- =============================================================
-- NOTIFICATION PREFERENCES
-- =============================================================

CREATE TABLE notification_preferences (
    preference_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    channel             VARCHAR(10) NOT NULL CHECK (channel IN ('email', 'sms', 'push')),
    category            VARCHAR(30) NOT NULL CHECK (category IN ('billing', 'payment', 'outage', 'usage_alert', 'programme', 'security', 'marketing')),
    enabled             BOOLEAN DEFAULT TRUE,
    UNIQUE (customer_id, channel, category)
);
```

### 2. Service Location and Metering Domain

```sql
-- =============================================================
-- SERVICE LOCATIONS (CIM: ServiceLocation)
-- =============================================================

CREATE TABLE service_locations (
    location_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    address_line1       VARCHAR(255) NOT NULL,
    address_line2       VARCHAR(255),
    city                VARCHAR(100) NOT NULL,
    state_province      VARCHAR(50) NOT NULL,
    postal_code         VARCHAR(20) NOT NULL,
    country             VARCHAR(3) DEFAULT 'USA',
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    geom                GEOMETRY(Point, 4326),                  -- PostGIS geometry for spatial queries
    service_type        VARCHAR(20) DEFAULT 'electric' CHECK (service_type IN ('electric', 'gas', 'water', 'multi')),
    status              VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'pending_connect', 'pending_disconnect')),
    premise_type        VARCHAR(30) CHECK (premise_type IN ('single_family', 'multi_family', 'apartment', 'commercial', 'industrial', 'agricultural')),
    square_footage      INTEGER,
    year_built          INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_service_locations_customer ON service_locations (customer_id);
CREATE INDEX idx_service_locations_geom ON service_locations USING GIST (geom);
CREATE INDEX idx_service_locations_status ON service_locations (status);

-- =============================================================
-- METERS (CIM: Meter / EndDevice)
-- =============================================================

CREATE TABLE meters (
    meter_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    meter_serial        VARCHAR(50) UNIQUE NOT NULL,
    meter_type          VARCHAR(20) NOT NULL CHECK (meter_type IN ('ami', 'amr', 'manual', 'net_meter')),
    manufacturer        VARCHAR(100),
    model               VARCHAR(100),
    firmware_version    VARCHAR(50),
    install_date        DATE,
    multiplier          NUMERIC(10,4) DEFAULT 1.0,              -- CT/PT multiplier
    is_bidirectional    BOOLEAN DEFAULT FALSE,                   -- supports net metering
    communication_type  VARCHAR(30) CHECK (communication_type IN ('rf_mesh', 'cellular', 'plc', 'wifi', 'manual')),
    status              VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'retired', 'maintenance')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_meters_location ON meters (location_id);
CREATE INDEX idx_meters_serial ON meters (meter_serial);

-- =============================================================
-- USAGE POINTS (CIM: UsagePoint — logical metering point)
-- =============================================================

CREATE TABLE usage_points (
    usage_point_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meter_id            UUID NOT NULL REFERENCES meters(meter_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    service_category    VARCHAR(20) DEFAULT 'electricity',
    phase_code          VARCHAR(10) CHECK (phase_code IN ('A', 'B', 'C', 'AB', 'BC', 'AC', 'ABC')),
    rated_power_kw      NUMERIC(10,2),
    green_button_id     VARCHAR(100) UNIQUE,                    -- ESPI resource identifier
    status              VARCHAR(20) DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_usage_points_meter ON usage_points (meter_id);
CREATE INDEX idx_usage_points_location ON usage_points (location_id);
```

### 3. Meter Data and Usage Domain

```sql
-- =============================================================
-- INTERVAL READINGS (CIM: IntervalBlock / IntervalReading)
-- Partitioned by month for performance on high-volume AMI data
-- =============================================================

CREATE TABLE interval_readings (
    reading_id          BIGSERIAL,
    usage_point_id      UUID NOT NULL REFERENCES usage_points(usage_point_id),
    interval_start      TIMESTAMPTZ NOT NULL,
    interval_duration   INTEGER NOT NULL DEFAULT 900,           -- seconds (900 = 15 min)
    consumption_wh      NUMERIC(12,2) NOT NULL,                 -- watt-hours consumed
    generation_wh       NUMERIC(12,2) DEFAULT 0,                -- watt-hours generated (solar/DER)
    net_wh              NUMERIC(12,2) GENERATED ALWAYS AS (consumption_wh - generation_wh) STORED,
    demand_w            NUMERIC(10,2),                          -- instantaneous demand in watts
    voltage_v           NUMERIC(7,2),
    power_factor        NUMERIC(5,4),
    reading_quality     VARCHAR(20) DEFAULT 'actual' CHECK (reading_quality IN ('actual', 'estimated', 'validated', 'edited')),
    source              VARCHAR(20) DEFAULT 'mdms' CHECK (source IN ('mdms', 'green_button', 'manual', 'estimated')),
    received_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (reading_id, interval_start)
) PARTITION BY RANGE (interval_start);

-- Create partitions for 2 years of data
-- Example: monthly partitions
CREATE TABLE interval_readings_2026_01 PARTITION OF interval_readings
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE interval_readings_2026_02 PARTITION OF interval_readings
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... (continue for each month)
CREATE TABLE interval_readings_2026_06 PARTITION OF interval_readings
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_interval_readings_usage_point ON interval_readings (usage_point_id, interval_start DESC);
CREATE INDEX idx_interval_readings_quality ON interval_readings (reading_quality) WHERE reading_quality != 'actual';

-- =============================================================
-- DAILY USAGE SUMMARIES (pre-aggregated from interval readings)
-- =============================================================

CREATE TABLE daily_usage_summaries (
    summary_id          BIGSERIAL PRIMARY KEY,
    usage_point_id      UUID NOT NULL REFERENCES usage_points(usage_point_id),
    usage_date          DATE NOT NULL,
    total_consumption_kwh   NUMERIC(10,3) NOT NULL,
    total_generation_kwh    NUMERIC(10,3) DEFAULT 0,
    net_consumption_kwh     NUMERIC(10,3),
    peak_demand_kw          NUMERIC(10,3),
    peak_demand_time        TIMESTAMPTZ,
    on_peak_kwh             NUMERIC(10,3),                      -- TOU on-peak consumption
    mid_peak_kwh            NUMERIC(10,3),                      -- TOU mid-peak
    off_peak_kwh            NUMERIC(10,3),                      -- TOU off-peak
    super_off_peak_kwh      NUMERIC(10,3),                      -- TOU super off-peak
    avg_temperature_f       NUMERIC(5,1),                       -- weather correlation
    reading_count           INTEGER,
    estimated_count         INTEGER DEFAULT 0,
    UNIQUE (usage_point_id, usage_date)
);

CREATE INDEX idx_daily_summaries_date ON daily_usage_summaries (usage_point_id, usage_date DESC);

-- =============================================================
-- MONTHLY USAGE SUMMARIES
-- =============================================================

CREATE TABLE monthly_usage_summaries (
    summary_id          BIGSERIAL PRIMARY KEY,
    usage_point_id      UUID NOT NULL REFERENCES usage_points(usage_point_id),
    billing_month       DATE NOT NULL,                          -- first day of month
    total_consumption_kwh   NUMERIC(12,3) NOT NULL,
    total_generation_kwh    NUMERIC(12,3) DEFAULT 0,
    net_consumption_kwh     NUMERIC(12,3),
    peak_demand_kw          NUMERIC(10,3),
    on_peak_kwh             NUMERIC(10,3),
    off_peak_kwh            NUMERIC(10,3),
    avg_daily_kwh           NUMERIC(10,3),
    neighbour_avg_kwh       NUMERIC(12,3),                      -- peer benchmark
    neighbour_percentile    INTEGER,                             -- 1-100 percentile ranking
    degree_days_heating     NUMERIC(6,1),
    degree_days_cooling     NUMERIC(6,1),
    UNIQUE (usage_point_id, billing_month)
);
```

### 4. Billing and Payment Domain

```sql
-- =============================================================
-- RATE PLANS / TARIFFS
-- =============================================================

CREATE TABLE rate_plans (
    rate_plan_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    utility_id          UUID,                                   -- for multi-utility deployments
    plan_code           VARCHAR(50) UNIQUE NOT NULL,
    plan_name           VARCHAR(200) NOT NULL,
    plan_description    TEXT,
    rate_type           VARCHAR(30) NOT NULL CHECK (rate_type IN ('flat', 'tiered', 'tou', 'real_time', 'demand', 'ev_tou', 'nem_tou', 'agricultural')),
    commodity           VARCHAR(20) DEFAULT 'electricity',
    customer_class      VARCHAR(20) CHECK (customer_class IN ('residential', 'commercial', 'industrial', 'agricultural')),
    openei_rate_id      VARCHAR(50),                            -- link to OpenEI URDB
    effective_date      DATE NOT NULL,
    expiration_date     DATE,
    is_default          BOOLEAN DEFAULT FALSE,
    is_nem_eligible     BOOLEAN DEFAULT FALSE,
    is_ev_eligible      BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- RATE SCHEDULE DETAILS
-- =============================================================

CREATE TABLE rate_schedule_periods (
    period_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rate_plan_id        UUID NOT NULL REFERENCES rate_plans(rate_plan_id),
    period_name         VARCHAR(50) NOT NULL,                   -- 'on_peak', 'off_peak', 'super_off_peak'
    season              VARCHAR(20) CHECK (season IN ('summer', 'winter', 'spring', 'fall', 'all')),
    day_type            VARCHAR(20) CHECK (day_type IN ('weekday', 'weekend', 'holiday', 'all')),
    start_hour          TIME,
    end_hour            TIME,
    rate_per_kwh        NUMERIC(8,5) NOT NULL,                  -- $/kWh
    demand_charge_per_kw NUMERIC(8,4),                          -- $/kW (demand charges)
    tier_min_kwh        NUMERIC(10,2),                          -- tier floor
    tier_max_kwh        NUMERIC(10,2),                          -- tier ceiling (NULL = unlimited)
    baseline_pct        NUMERIC(5,2),                           -- % of baseline allocation
    nem_export_rate     NUMERIC(8,5),                           -- $/kWh for NEM export credit
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_periods_plan ON rate_schedule_periods (rate_plan_id);

-- =============================================================
-- CUSTOMER RATE PLAN ENROLMENT
-- =============================================================

CREATE TABLE customer_rate_enrolments (
    enrolment_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    rate_plan_id        UUID NOT NULL REFERENCES rate_plans(rate_plan_id),
    effective_date      DATE NOT NULL,
    end_date            DATE,
    enrolment_source    VARCHAR(20) CHECK (enrolment_source IN ('portal', 'cis', 'phone', 'auto')),
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'pending', 'ended', 'cancelled')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_enrolments_customer ON customer_rate_enrolments (customer_id, status);

-- =============================================================
-- BILLS
-- =============================================================

CREATE TABLE bills (
    bill_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    rate_plan_id        UUID REFERENCES rate_plans(rate_plan_id),
    external_bill_id    VARCHAR(50),                            -- CIS bill number
    billing_period_start DATE NOT NULL,
    billing_period_end   DATE NOT NULL,
    statement_date      DATE NOT NULL,
    due_date            DATE NOT NULL,
    total_kwh           NUMERIC(12,3),
    total_generation_kwh NUMERIC(12,3),
    net_kwh             NUMERIC(12,3),
    peak_demand_kw      NUMERIC(10,3),
    subtotal_energy     NUMERIC(10,2),                          -- energy charges
    subtotal_demand     NUMERIC(10,2),                          -- demand charges
    subtotal_fixed      NUMERIC(10,2),                          -- fixed charges (customer charge, minimum bill)
    taxes_and_fees      NUMERIC(10,2),
    nem_credit_applied  NUMERIC(10,2) DEFAULT 0,               -- NEM credits applied to this bill
    previous_balance    NUMERIC(10,2),
    payments_received   NUMERIC(10,2),
    adjustments         NUMERIC(10,2) DEFAULT 0,
    total_amount_due    NUMERIC(10,2) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'issued' CHECK (status IN ('draft', 'issued', 'paid', 'partial', 'overdue', 'cancelled', 'disputed')),
    pdf_url             VARCHAR(500),
    ai_explanation      TEXT,                                   -- AI-generated plain-language explanation
    ai_high_bill_flag   BOOLEAN DEFAULT FALSE,
    ai_high_bill_reason TEXT,                                   -- AI root-cause analysis for high bills
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bills_customer ON bills (customer_id, statement_date DESC);
CREATE INDEX idx_bills_status ON bills (status) WHERE status IN ('issued', 'overdue');
CREATE INDEX idx_bills_due_date ON bills (due_date) WHERE status NOT IN ('paid', 'cancelled');

-- =============================================================
-- BILL LINE ITEMS (itemised breakdown)
-- =============================================================

CREATE TABLE bill_line_items (
    line_item_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bill_id             UUID NOT NULL REFERENCES bills(bill_id) ON DELETE CASCADE,
    line_order          INTEGER NOT NULL,
    category            VARCHAR(30) NOT NULL CHECK (category IN ('energy', 'demand', 'fixed', 'tax', 'fee', 'credit', 'adjustment', 'nem_credit', 'programme')),
    description         VARCHAR(255) NOT NULL,
    quantity            NUMERIC(12,3),
    unit                VARCHAR(10),                            -- 'kWh', 'kW', 'days', etc.
    rate                NUMERIC(10,6),
    amount              NUMERIC(10,2) NOT NULL,
    rate_period_name    VARCHAR(50),                            -- 'on_peak', 'tier_1', etc.
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bill_line_items_bill ON bill_line_items (bill_id);

-- =============================================================
-- PAYMENTS (PCI DSS compliant — no raw card data stored)
-- =============================================================

CREATE TABLE payments (
    payment_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    bill_id             UUID REFERENCES bills(bill_id),
    amount              NUMERIC(10,2) NOT NULL CHECK (amount > 0),
    payment_method      VARCHAR(20) NOT NULL CHECK (payment_method IN ('card', 'ach', 'autopay_card', 'autopay_ach', 'cash_kiosk', 'check', 'wallet')),
    payment_token       VARCHAR(255),                           -- tokenised payment reference from processor
    card_last_four      VARCHAR(4),
    card_brand          VARCHAR(20),
    ach_routing_last_four VARCHAR(4),
    processor_txn_id    VARCHAR(100),                           -- payment processor transaction ID
    status              VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded', 'reversed')),
    failure_reason      VARCHAR(255),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_customer ON payments (customer_id, created_at DESC);
CREATE INDEX idx_payments_bill ON payments (bill_id);
CREATE INDEX idx_payments_status ON payments (status) WHERE status IN ('pending', 'processing');

-- =============================================================
-- AUTOPAY ENROLMENT
-- =============================================================

CREATE TABLE autopay_enrolments (
    enrolment_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    payment_method      VARCHAR(20) NOT NULL CHECK (payment_method IN ('card', 'ach')),
    payment_token       VARCHAR(255) NOT NULL,                  -- tokenised payment credential
    card_last_four      VARCHAR(4),
    card_brand          VARCHAR(20),
    card_expiry         VARCHAR(7),                             -- MM/YYYY
    ach_routing_last_four VARCHAR(4),
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'paused', 'cancelled')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_autopay_customer ON autopay_enrolments (customer_id);

-- =============================================================
-- BUDGET BILLING / PAYMENT PLANS
-- =============================================================

CREATE TABLE payment_plans (
    plan_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    plan_type           VARCHAR(20) NOT NULL CHECK (plan_type IN ('budget_billing', 'payment_arrangement', 'low_income_assistance')),
    monthly_amount      NUMERIC(10,2) NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE,
    reconciliation_date DATE,                                   -- annual true-up date for budget billing
    balance             NUMERIC(10,2) DEFAULT 0,                -- running balance (over/under actual)
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'completed', 'defaulted', 'cancelled')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 5. Net Metering Domain

```sql
-- =============================================================
-- NET METERING ACCOUNTS
-- =============================================================

CREATE TABLE nem_accounts (
    nem_account_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    usage_point_id      UUID NOT NULL REFERENCES usage_points(usage_point_id),
    nem_programme       VARCHAR(30) NOT NULL CHECK (nem_programme IN ('nem_1', 'nem_2', 'nem_3', 'nem_t', 'vnem', 'nbia')),
    interconnection_date DATE NOT NULL,
    system_size_kw      NUMERIC(10,3) NOT NULL,
    panel_count         INTEGER,
    inverter_type       VARCHAR(100),
    battery_capacity_kwh NUMERIC(10,3),
    annual_cycle_start  DATE NOT NULL,                          -- true-up anniversary
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'pending', 'suspended', 'closed')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_nem_accounts_customer ON nem_accounts (customer_id);

-- =============================================================
-- NEM MONTHLY SETTLEMENT
-- =============================================================

CREATE TABLE nem_monthly_settlements (
    settlement_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nem_account_id      UUID NOT NULL REFERENCES nem_accounts(nem_account_id),
    settlement_month    DATE NOT NULL,
    total_import_kwh    NUMERIC(12,3) NOT NULL,                 -- consumed from grid
    total_export_kwh    NUMERIC(12,3) NOT NULL,                 -- exported to grid
    net_kwh             NUMERIC(12,3) NOT NULL,                 -- import - export
    on_peak_export_kwh  NUMERIC(12,3),
    off_peak_export_kwh NUMERIC(12,3),
    credit_earned       NUMERIC(10,2) NOT NULL,                 -- monetary credit for exports
    credit_used         NUMERIC(10,2) DEFAULT 0,                -- credits applied to charges
    credit_balance      NUMERIC(10,2) NOT NULL,                 -- rolling credit balance
    nsc_eligible_kwh    NUMERIC(12,3),                          -- Net Surplus Compensation eligible
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (nem_account_id, settlement_month)
);

-- =============================================================
-- NEM ANNUAL TRUE-UP
-- =============================================================

CREATE TABLE nem_annual_trueups (
    trueup_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nem_account_id      UUID NOT NULL REFERENCES nem_accounts(nem_account_id),
    cycle_start         DATE NOT NULL,
    cycle_end           DATE NOT NULL,
    total_import_kwh    NUMERIC(14,3),
    total_export_kwh    NUMERIC(14,3),
    total_credits_earned NUMERIC(12,2),
    total_charges       NUMERIC(12,2),
    net_amount          NUMERIC(12,2) NOT NULL,                 -- positive = customer owes, negative = NSC payout
    nsc_payout          NUMERIC(12,2) DEFAULT 0,
    status              VARCHAR(20) DEFAULT 'calculated' CHECK (status IN ('calculated', 'issued', 'paid', 'disputed')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6. Outage Management Domain

```sql
-- =============================================================
-- OUTAGE EVENTS (sourced from OMS integration)
-- =============================================================

CREATE TABLE outage_events (
    outage_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_oms_id     VARCHAR(100) UNIQUE,                    -- OMS ticket/event ID
    outage_type         VARCHAR(30) NOT NULL CHECK (outage_type IN ('unplanned', 'planned', 'momentary', 'rolling_blackout')),
    cause               VARCHAR(50) CHECK (cause IN ('weather', 'equipment_failure', 'vehicle_accident', 'animal', 'vegetation', 'overload', 'planned_maintenance', 'unknown', 'other')),
    cause_detail        TEXT,
    severity            VARCHAR(20) CHECK (severity IN ('major', 'significant', 'minor', 'momentary')),
    status              VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'crew_assigned', 'crew_en_route', 'crew_on_site', 'assessing', 'restoring', 'restored', 'cancelled')),
    affected_area_geom  GEOMETRY(MultiPolygon, 4326),           -- PostGIS polygon of affected area
    customers_affected  INTEGER,
    start_time          TIMESTAMPTZ NOT NULL,
    estimated_restoration TIMESTAMPTZ,
    actual_restoration  TIMESTAMPTZ,
    crew_status         VARCHAR(100),
    source              VARCHAR(20) DEFAULT 'oms' CHECK (source IN ('oms', 'customer_report', 'scada', 'ami')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outage_events_status ON outage_events (status) WHERE status != 'restored';
CREATE INDEX idx_outage_events_geom ON outage_events USING GIST (affected_area_geom);
CREATE INDEX idx_outage_events_time ON outage_events (start_time DESC);

-- =============================================================
-- CUSTOMER OUTAGE REPORTS
-- =============================================================

CREATE TABLE customer_outage_reports (
    report_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    outage_id           UUID REFERENCES outage_events(outage_id),  -- linked to OMS event once correlated
    report_type         VARCHAR(30) DEFAULT 'no_power' CHECK (report_type IN ('no_power', 'partial_power', 'flickering', 'low_voltage', 'streetlight', 'other')),
    description         TEXT,
    reported_via        VARCHAR(20) DEFAULT 'portal' CHECK (reported_via IN ('portal', 'mobile', 'ivr', 'sms', 'phone')),
    status              VARCHAR(20) DEFAULT 'received' CHECK (status IN ('received', 'confirmed', 'merged', 'resolved', 'duplicate')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at         TIMESTAMPTZ
);

CREATE INDEX idx_outage_reports_customer ON customer_outage_reports (customer_id, created_at DESC);
CREATE INDEX idx_outage_reports_location ON customer_outage_reports (location_id);

-- =============================================================
-- OUTAGE NOTIFICATIONS
-- =============================================================

CREATE TABLE outage_notifications (
    notification_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    outage_id           UUID NOT NULL REFERENCES outage_events(outage_id),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    channel             VARCHAR(10) NOT NULL CHECK (channel IN ('email', 'sms', 'push')),
    notification_type   VARCHAR(30) NOT NULL CHECK (notification_type IN ('outage_detected', 'crew_assigned', 'eta_update', 'power_restored', 'planned_outage')),
    message_content     TEXT,
    status              VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'sent', 'delivered', 'failed', 'bounced')),
    sent_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outage_notif_outage ON outage_notifications (outage_id);
CREATE INDEX idx_outage_notif_customer ON outage_notifications (customer_id);
```

### 7. Programme and Demand Response Domain

```sql
-- =============================================================
-- UTILITY PROGRAMMES
-- =============================================================

CREATE TABLE programmes (
    programme_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_code      VARCHAR(50) UNIQUE NOT NULL,
    programme_name      VARCHAR(200) NOT NULL,
    programme_type      VARCHAR(30) NOT NULL CHECK (programme_type IN ('demand_response', 'energy_efficiency', 'rebate', 'low_income', 'ev_managed_charging', 'solar_incentive', 'battery_storage', 'audit')),
    description         TEXT,
    eligibility_criteria TEXT,
    incentive_type      VARCHAR(20) CHECK (incentive_type IN ('bill_credit', 'rebate', 'rate_discount', 'device', 'none')),
    incentive_amount    NUMERIC(10,2),
    start_date          DATE,
    end_date            DATE,
    max_enrolments      INTEGER,
    current_enrolments  INTEGER DEFAULT 0,
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'upcoming', 'closed', 'archived')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- PROGRAMME ENROLMENTS
-- =============================================================

CREATE TABLE programme_enrolments (
    enrolment_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id        UUID NOT NULL REFERENCES programmes(programme_id),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID REFERENCES service_locations(location_id),
    enrolled_via        VARCHAR(20) DEFAULT 'portal' CHECK (enrolled_via IN ('portal', 'phone', 'mail', 'auto')),
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN ('pending', 'active', 'opted_out', 'completed', 'cancelled')),
    enrolled_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    opted_out_at        TIMESTAMPTZ,
    UNIQUE (programme_id, customer_id, location_id)
);

CREATE INDEX idx_programme_enrolments_customer ON programme_enrolments (customer_id);

-- =============================================================
-- DEMAND RESPONSE EVENTS (OpenADR EiEvent)
-- =============================================================

CREATE TABLE dr_events (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id        UUID NOT NULL REFERENCES programmes(programme_id),
    openadr_event_id    VARCHAR(100),                           -- OpenADR event identifier
    event_name          VARCHAR(200),
    event_type          VARCHAR(30) CHECK (event_type IN ('load_shed', 'load_shift', 'critical_peak', 'real_time_pricing', 'ev_curtailment')),
    signal_type         VARCHAR(30) CHECK (signal_type IN ('simple', 'level', 'price', 'price_relative', 'setpoint')),
    signal_value        NUMERIC(10,4),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    notification_time   TIMESTAMPTZ,                            -- when advance notice was sent
    status              VARCHAR(20) DEFAULT 'scheduled' CHECK (status IN ('scheduled', 'active', 'completed', 'cancelled')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dr_events_programme ON dr_events (programme_id);
CREATE INDEX idx_dr_events_time ON dr_events (start_time, end_time);

-- =============================================================
-- DR EVENT PARTICIPATION
-- =============================================================

CREATE TABLE dr_event_participation (
    participation_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id            UUID NOT NULL REFERENCES dr_events(event_id),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    opt_status          VARCHAR(20) DEFAULT 'enrolled' CHECK (opt_status IN ('enrolled', 'opted_in', 'opted_out')),
    baseline_kwh        NUMERIC(10,3),                          -- expected consumption without DR
    actual_kwh          NUMERIC(10,3),                          -- actual consumption during event
    reduction_kwh       NUMERIC(10,3),                          -- baseline - actual
    incentive_earned    NUMERIC(10,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dr_participation_event ON dr_event_participation (event_id);
CREATE INDEX idx_dr_participation_customer ON dr_event_participation (customer_id);
```

### 8. EV Integration Domain

```sql
-- =============================================================
-- EV CHARGERS
-- =============================================================

CREATE TABLE ev_chargers (
    charger_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    charger_make        VARCHAR(100),
    charger_model       VARCHAR(100),
    ocpp_charge_point_id VARCHAR(100),                          -- OCPP 2.1 identifier
    max_power_kw        NUMERIC(6,2),
    connector_type      VARCHAR(30) CHECK (connector_type IN ('j1772', 'ccs1', 'ccs2', 'chademo', 'nacs', 'type2')),
    is_v2g_capable      BOOLEAN DEFAULT FALSE,
    communication_type  VARCHAR(20) CHECK (communication_type IN ('ocpp', 'proprietary_api', 'manual')),
    status              VARCHAR(20) DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ev_chargers_customer ON ev_chargers (customer_id);

-- =============================================================
-- EV CHARGING SCHEDULES
-- =============================================================

CREATE TABLE ev_charging_schedules (
    schedule_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    charger_id          UUID NOT NULL REFERENCES ev_chargers(charger_id),
    schedule_name       VARCHAR(100),
    day_of_week         INTEGER[] NOT NULL,                     -- {0,1,2,3,4,5,6} = Sun-Sat
    start_time          TIME NOT NULL,
    end_time            TIME NOT NULL,
    max_charge_rate_kw  NUMERIC(6,2),
    target_soc_pct      INTEGER CHECK (target_soc_pct BETWEEN 0 AND 100),
    is_active           BOOLEAN DEFAULT TRUE,
    prefer_off_peak     BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- EV CHARGING SESSIONS
-- =============================================================

CREATE TABLE ev_charging_sessions (
    session_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    charger_id          UUID NOT NULL REFERENCES ev_chargers(charger_id),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ,
    energy_kwh          NUMERIC(8,3),
    peak_power_kw       NUMERIC(6,2),
    cost_estimate       NUMERIC(8,2),
    rate_period         VARCHAR(30),                            -- on_peak, off_peak etc.
    was_dr_curtailed    BOOLEAN DEFAULT FALSE,
    ocpp_transaction_id VARCHAR(100),
    status              VARCHAR(20) DEFAULT 'in_progress' CHECK (status IN ('in_progress', 'completed', 'interrupted', 'failed')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ev_sessions_charger ON ev_charging_sessions (charger_id, start_time DESC);
```

### 9. Service Orders and Green Button Domain

```sql
-- =============================================================
-- SERVICE ORDERS (start/stop/transfer)
-- =============================================================

CREATE TABLE service_orders (
    order_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID REFERENCES service_locations(location_id),
    external_order_id   VARCHAR(50),                            -- CIS work order number
    order_type          VARCHAR(30) NOT NULL CHECK (order_type IN ('start_service', 'stop_service', 'transfer_service', 'meter_test', 'reconnect', 'disconnect')),
    requested_date      DATE NOT NULL,
    scheduled_date      DATE,
    completed_date      DATE,
    status              VARCHAR(20) DEFAULT 'submitted' CHECK (status IN ('submitted', 'scheduled', 'in_progress', 'completed', 'cancelled', 'rejected')),
    notes               TEXT,
    submitted_via       VARCHAR(20) DEFAULT 'portal',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_service_orders_customer ON service_orders (customer_id, created_at DESC);

-- =============================================================
-- GREEN BUTTON AUTHORISATIONS (CMD third-party data sharing)
-- =============================================================

CREATE TABLE green_button_authorisations (
    auth_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    usage_point_id      UUID NOT NULL REFERENCES usage_points(usage_point_id),
    third_party_name    VARCHAR(200) NOT NULL,
    third_party_id      VARCHAR(100) NOT NULL,                  -- OAuth client_id of third party
    scope               VARCHAR(50) NOT NULL DEFAULT 'usage_data',
    oauth_token_hash    VARCHAR(64),                            -- SHA-256 of issued access token
    authorised_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ,
    revoked_at          TIMESTAMPTZ,
    status              VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'expired', 'revoked')),
    UNIQUE (customer_id, usage_point_id, third_party_id)
);

CREATE INDEX idx_gb_auth_customer ON green_button_authorisations (customer_id);

-- =============================================================
-- AI-GENERATED INSIGHTS
-- =============================================================

CREATE TABLE ai_insights (
    insight_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID REFERENCES service_locations(location_id),
    insight_type        VARCHAR(30) NOT NULL CHECK (insight_type IN ('high_bill_explanation', 'rate_recommendation', 'efficiency_tip', 'usage_anomaly', 'appliance_insight', 'outage_eta', 'dr_recommendation')),
    title               VARCHAR(200) NOT NULL,
    body                TEXT NOT NULL,
    confidence_score    NUMERIC(5,4),
    data_window_start   DATE,
    data_window_end     DATE,
    is_read             BOOLEAN DEFAULT FALSE,
    is_dismissed        BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ai_insights_customer ON ai_insights (customer_id, created_at DESC);
CREATE INDEX idx_ai_insights_unread ON ai_insights (customer_id) WHERE NOT is_read AND NOT is_dismissed;
```

---

## Pros and Cons

### Pros

1. **Strong data integrity.** Foreign keys, check constraints, and unique constraints enforce business rules at the database level. Invalid states (e.g. a bill referencing a non-existent customer, a NEM settlement without an account) are structurally impossible.

2. **Mature tooling and expertise.** PostgreSQL is the most widely adopted open-source RDBMS. Utility IT teams are likely to have in-house PostgreSQL skills. Ecosystem tools (pgAdmin, pg_dump, logical replication, Patroni for HA) are production-proven.

3. **ACID transactions across domains.** A payment that updates the bill status and creates an audit log entry can occur in a single transaction. This is critical for billing accuracy and regulatory compliance.

4. **PostGIS for geospatial.** Native spatial indexing (GiST) enables efficient outage-map queries (e.g., "find all customers within this outage polygon") without a separate GIS database.

5. **Row-level security.** PostgreSQL RLS policies can enforce customer data isolation at the database layer, preventing one customer from accessing another's data even if application logic has a bug.

6. **Partitioning for time-series.** Native declarative partitioning on interval_readings by month enables efficient data lifecycle management (drop old partitions, attach new ones) and query pruning.

7. **Standards alignment.** Table structures map directly to IEC 61968 CIM concepts (UsagePoint, MeterReading), Green Button ESPI resources, and OpenADR event models, making integration straightforward.

### Cons

1. **High-volume interval data pressure.** A utility with 500,000 meters producing 15-minute reads generates ~48 million rows/day in interval_readings. Monthly partitioning helps, but sustained write throughput at this scale requires careful tuning (batch COPY, WAL configuration, connection pooling) and may push PostgreSQL's single-writer architecture.

2. **Schema rigidity.** Adding a new rate plan type, tariff structure, or programme attribute requires DDL changes and potentially application-layer migrations. Rate plans in particular vary enormously across jurisdictions and change frequently.

3. **Denormalisation pressure for dashboards.** The normalized model requires multi-table joins for common dashboard queries (e.g., "show me this customer's latest bill with line items, NEM credits, and daily usage chart"). Pre-computed summary tables (daily_usage_summaries, monthly_usage_summaries) partially address this but must be maintained.

4. **No native time-series optimisation.** Unlike TimescaleDB, vanilla PostgreSQL partitioning does not provide automatic chunk management, compression, or continuous aggregate functions. This must be implemented in application code or by adding TimescaleDB as an extension.

5. **Outage map scalability.** During a major storm, thousands of concurrent users querying PostGIS spatial data against active outage polygons can create significant read load. Caching layers (Redis, CDN) are essential but add architectural complexity.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with PostGIS 3.4+ extension |
| **Connection pooling** | PgBouncer (transaction-mode) for high-concurrency portal traffic |
| **High availability** | Patroni + etcd for automatic failover with streaming replication |
| **Backup** | pgBackRest for PITR (point-in-time recovery) with WAL archival to S3 |
| **Monitoring** | pg_stat_statements, Prometheus postgres_exporter, Grafana dashboards |
| **Migration tool** | Flyway or Alembic for versioned schema migrations |
| **Read replicas** | Streaming replicas for dashboard/analytics queries; portal reads from replica |
| **Caching** | Redis for session data, outage map tiles, and frequently-accessed bill summaries |
| **Search** | pg_trgm extension for customer name search; optional Elasticsearch for full-text |

---

## Migration and Scaling Considerations

### Data Migration

- **From CIS systems:** ETL pipeline to load customer, account, and historical billing data from Oracle CC&B, SAP IS-U, or other CIS via CSV/API. Map CIS account IDs to `external_cis_id` for ongoing synchronisation.
- **From MDMS:** Bulk load historical interval data via COPY INTO partitioned interval_readings table. Use `received_at` to track data freshness. For ongoing sync, use CDC (change data capture) from MDMS or scheduled batch loads.
- **Green Button backfill:** Import historical Green Button XML files by parsing ESPI UsagePoint/IntervalBlock resources into the interval_readings and usage_points tables.

### Scaling Strategy

1. **Vertical first.** A well-tuned PostgreSQL instance on modern hardware (64+ cores, 256GB+ RAM, NVMe SSDs) can handle millions of customers with proper indexing and partitioning.
2. **Read replicas for analytics.** Route dashboard queries, neighbour benchmarking calculations, and AI insight generation to streaming replicas.
3. **Partition lifecycle.** Drop interval_readings partitions older than the retention window (typically 3-5 years for regulatory compliance) or move to cold storage.
4. **Connection pooling.** PgBouncer in transaction mode to support thousands of concurrent portal users without exhausting PostgreSQL connection limits.
5. **Horizontal sharding (future).** If a single PostgreSQL instance reaches its limits, Citus can distribute tables by customer_id. This is typically needed only at scale beyond 5 million customers.

### Performance Benchmarks (Estimated)

| Metric | Expected Performance |
|--------|---------------------|
| Concurrent portal users | 10,000+ with PgBouncer |
| Interval reading ingestion | 100,000+ rows/sec with COPY |
| Bill lookup by customer | <5ms with index |
| Outage map query (active events) | <50ms with spatial index and caching |
| Dashboard page load (bill + usage + NEM) | <200ms with summary tables |

---

## Summary

The fully normalized PostgreSQL model is the safest starting point for a utility customer portal. It provides maximum data integrity, aligns with CIM standards, leverages the broadest pool of PostgreSQL expertise, and handles the transactional requirements of billing and payment. The main trade-off is managing high-volume interval data at scale and accommodating the inherent variability of rate plan structures across different utility jurisdictions. For utilities with fewer than 1 million meters, this model can serve as the sole database. For larger deployments, it serves as the transactional core alongside a dedicated time-series store (see Suggestion 4).
