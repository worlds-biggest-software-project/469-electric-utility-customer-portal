# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

> Project: Electric Utility Customer Portal · Generated: 2026-05-26

---

## Overview

This model uses PostgreSQL as a single database engine but strategically mixes normalized relational columns with JSONB document columns. Stable, frequently-queried, and relationship-heavy data (customers, meters, bills, payments) lives in traditional typed columns with foreign keys and constraints. Variable, jurisdiction-specific, or rapidly-evolving data (rate plan structures, programme eligibility rules, outage metadata, AI insights, Green Button ESPI data) lives in JSONB columns with GIN indexes.

The rationale is pragmatic: electric utility data has a stable core (customers have accounts, accounts have meters, meters produce readings, readings generate bills) but enormous variability at the edges. Rate plan structures differ radically across utilities, states, and tariff types. Programme eligibility rules change with each regulatory filing. Outage metadata from different OMS vendors has different schemas. Net metering settlement rules vary by state and NEM programme version. Forcing all this variability into rigid relational columns produces either an explosion of nullable columns, EAV (entity-attribute-value) anti-patterns, or constant DDL migrations. JSONB absorbs this variability while keeping the stable core strictly typed and constrained.

---

## Design Principles

1. **Relational columns for identity, relationships, status, and dates.** These are the columns used in WHERE clauses, JOINs, foreign keys, and unique constraints. They must be strongly typed.
2. **JSONB columns for variable attributes, configuration, and rich metadata.** These are the columns whose structure varies across deployments, changes frequently, or contains nested/array data that does not participate in relational integrity constraints.
3. **GIN indexes on frequently-queried JSONB paths.** Not every JSONB column needs indexing, but high-traffic query paths (e.g., rate plan type, outage cause, programme eligibility) get expression indexes.
4. **CHECK constraints on JSONB where critical.** PostgreSQL supports CHECK constraints that validate JSONB structure, providing a safety net for critical document fields.
5. **No EAV.** JSONB replaces the need for EAV patterns entirely.

---

## Schema Design

### 1. Customer Domain (Mostly Relational)

```sql
-- =============================================================
-- CUSTOMERS — stable identity data in columns, preferences in JSONB
-- =============================================================

CREATE TABLE customers (
    customer_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_cis_id     VARCHAR(50) UNIQUE,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    email               VARCHAR(255) UNIQUE NOT NULL,
    phone_primary       VARCHAR(20),
    customer_type       VARCHAR(20) NOT NULL CHECK (customer_type IN ('residential', 'commercial', 'industrial', 'agricultural')),
    status              VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'closed', 'suspended')),

    -- JSONB: notification preferences, communication settings, accessibility needs
    -- Varies by deployment and evolves rapidly
    preferences         JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    Example preferences structure:
    {
        "paperless_billing": true,
        "preferred_language": "en",
        "notifications": {
            "billing": {"email": true, "sms": false, "push": true},
            "outage": {"email": true, "sms": true, "push": true},
            "usage_alert": {"email": true, "sms": false, "push": false},
            "programme": {"email": true, "sms": false, "push": false},
            "security": {"email": true, "sms": true, "push": true}
        },
        "accessibility": {
            "screen_reader_optimised": false,
            "high_contrast": false,
            "large_text": false
        },
        "marketing_opt_in": false,
        "communication_language": "en"
    }
    */

    -- JSONB: security metadata from Keycloak + portal anomaly detection
    security_metadata   JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    {
        "keycloak_user_id": "uuid",
        "mfa_enabled": true,
        "mfa_method": "totp",
        "last_login_at": "2026-05-25T10:00:00Z",
        "last_login_ip": "192.168.1.1",
        "failed_login_count": 0,
        "trusted_devices": [
            {"device_id": "abc", "name": "iPhone 16", "last_used": "2026-05-25"}
        ],
        "login_anomaly_history": [
            {"timestamp": "2026-05-20T03:00:00Z", "ip": "45.33.32.156", "score": 0.87, "action": "mfa_challenged"}
        ]
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_email ON customers (email);
CREATE INDEX idx_customers_cis_id ON customers (external_cis_id);
CREATE INDEX idx_customers_status ON customers (status);
-- GIN index on preferences for notification queries
CREATE INDEX idx_customers_preferences ON customers USING GIN (preferences jsonb_path_ops);
```

### 2. Service Location and Metering Domain

```sql
-- =============================================================
-- SERVICE LOCATIONS — relational core with JSONB for building characteristics
-- =============================================================

CREATE TABLE service_locations (
    location_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    address_line1       VARCHAR(255) NOT NULL,
    city                VARCHAR(100) NOT NULL,
    state_province      VARCHAR(50) NOT NULL,
    postal_code         VARCHAR(20) NOT NULL,
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    geom                GEOMETRY(Point, 4326),
    service_type        VARCHAR(20) DEFAULT 'electric',
    status              VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'pending_connect', 'pending_disconnect')),

    -- JSONB: building/premise characteristics used for neighbour benchmarking and AI insights
    -- Varies enormously: residential vs commercial vs agricultural properties
    premise_attributes  JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    Residential example:
    {
        "premise_type": "single_family",
        "square_footage": 2200,
        "year_built": 1998,
        "bedrooms": 4,
        "heating_type": "electric_heat_pump",
        "cooling_type": "central_ac",
        "pool": true,
        "pool_pump_hp": 1.5,
        "ev_charger": true,
        "solar_installed": true,
        "insulation_rating": "moderate",
        "climate_zone": "3B"
    }

    Commercial example:
    {
        "premise_type": "office",
        "square_footage": 45000,
        "floors": 3,
        "operating_hours": {"weekday": "07:00-19:00", "weekend": "closed"},
        "hvac_type": "commercial_rooftop",
        "lighting_type": "led",
        "data_center": false,
        "refrigeration": false,
        "naics_code": "541512"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_locations_customer ON service_locations (customer_id);
CREATE INDEX idx_locations_geom ON service_locations USING GIST (geom);

-- =============================================================
-- METERS — relational core with JSONB for vendor-specific configuration
-- =============================================================

CREATE TABLE meters (
    meter_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    meter_serial        VARCHAR(50) UNIQUE NOT NULL,
    meter_type          VARCHAR(20) NOT NULL CHECK (meter_type IN ('ami', 'amr', 'manual', 'net_meter')),
    is_bidirectional    BOOLEAN DEFAULT FALSE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    install_date        DATE,

    -- JSONB: vendor-specific meter configuration
    -- Different AMI vendors (Itron, Landis+Gyr, Honeywell) have completely different config schemas
    meter_config        JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    {
        "manufacturer": "Landis+Gyr",
        "model": "Gridstream RF",
        "firmware_version": "4.2.1",
        "multiplier": 1.0,
        "communication": {
            "type": "rf_mesh",
            "network_id": "NET-042",
            "signal_strength_dbm": -65,
            "last_communication": "2026-05-25T23:45:00Z"
        },
        "registers": [
            {"register_id": "1.0.1.8.0", "description": "Total Import Active Energy", "unit": "kWh"},
            {"register_id": "1.0.2.8.0", "description": "Total Export Active Energy", "unit": "kWh"}
        ],
        "dlms_cosem": {
            "obis_codes": ["1.0.1.8.0", "1.0.2.8.0", "1.0.1.7.0"],
            "security_suite": "suite_0"
        }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_meters_location ON meters (location_id);
CREATE INDEX idx_meters_serial ON meters (meter_serial);

-- =============================================================
-- USAGE POINTS
-- =============================================================

CREATE TABLE usage_points (
    usage_point_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meter_id            UUID NOT NULL REFERENCES meters(meter_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    green_button_id     VARCHAR(100) UNIQUE,
    status              VARCHAR(20) DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 3. Interval Readings (Relational, Partitioned)

```sql
-- =============================================================
-- INTERVAL READINGS — fully relational for query performance
-- No JSONB here: this is high-volume time-series data where every column matters
-- =============================================================

CREATE TABLE interval_readings (
    reading_id          BIGSERIAL,
    usage_point_id      UUID NOT NULL,
    interval_start      TIMESTAMPTZ NOT NULL,
    interval_duration   INTEGER NOT NULL DEFAULT 900,
    consumption_wh      NUMERIC(12,2) NOT NULL,
    generation_wh       NUMERIC(12,2) DEFAULT 0,
    net_wh              NUMERIC(12,2) GENERATED ALWAYS AS (consumption_wh - generation_wh) STORED,
    demand_w            NUMERIC(10,2),
    voltage_v           NUMERIC(7,2),
    power_factor        NUMERIC(5,4),
    reading_quality     VARCHAR(20) DEFAULT 'actual',
    source              VARCHAR(20) DEFAULT 'mdms',
    received_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (reading_id, interval_start)
) PARTITION BY RANGE (interval_start);

-- Monthly partitions (create programmatically for each month)
CREATE TABLE interval_readings_2026_01 PARTITION OF interval_readings
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... etc.

CREATE INDEX idx_readings_usage_point ON interval_readings (usage_point_id, interval_start DESC);

-- =============================================================
-- DAILY AND MONTHLY SUMMARIES — relational with JSONB for TOU breakdown
-- TOU periods differ by rate plan, making fixed columns impractical
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

    -- JSONB: TOU breakdown varies by rate plan (some have 2 periods, some have 5)
    tou_breakdown       JSONB,
    /*
    {
        "on_peak": {"kwh": 8.1, "hours": 6, "cost": 1.46},
        "mid_peak": {"kwh": 5.3, "hours": 4, "cost": 0.64},
        "off_peak": {"kwh": 12.8, "hours": 10, "cost": 1.02},
        "super_off_peak": {"kwh": 2.2, "hours": 4, "cost": 0.13}
    }
    */

    -- JSONB: weather data for correlation
    weather_data        JSONB,
    /*
    {
        "avg_temperature_f": 92.3,
        "max_temperature_f": 101.2,
        "min_temperature_f": 78.5,
        "cooling_degree_days": 27.3,
        "heating_degree_days": 0,
        "precipitation_inches": 0,
        "wind_speed_mph": 8.2
    }
    */

    UNIQUE (usage_point_id, usage_date)
);

CREATE INDEX idx_daily_summaries_date ON daily_usage_summaries (usage_point_id, usage_date DESC);
```

### 4. Rate Plans and Tariffs (Heavily JSONB)

```sql
-- =============================================================
-- RATE PLANS — the domain with the most structural variability
-- Rate structures differ enormously: flat, tiered, TOU, demand, seasonal, NEM variants
-- JSONB absorbs this variability without schema changes per utility/jurisdiction
-- =============================================================

CREATE TABLE rate_plans (
    rate_plan_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    utility_id          UUID,
    plan_code           VARCHAR(50) UNIQUE NOT NULL,
    plan_name           VARCHAR(200) NOT NULL,
    rate_type           VARCHAR(30) NOT NULL CHECK (rate_type IN ('flat', 'tiered', 'tou', 'real_time', 'demand', 'ev_tou', 'nem_tou', 'agricultural')),
    customer_class      VARCHAR(20),
    commodity           VARCHAR(20) DEFAULT 'electricity',
    effective_date      DATE NOT NULL,
    expiration_date     DATE,
    is_default          BOOLEAN DEFAULT FALSE,
    is_nem_eligible     BOOLEAN DEFAULT FALSE,
    is_ev_eligible      BOOLEAN DEFAULT FALSE,
    openei_rate_id      VARCHAR(50),

    -- JSONB: the complete rate structure — this is where the variability lives
    rate_structure      JSONB NOT NULL,
    /*
    FLAT rate example:
    {
        "fixed_charges": [
            {"name": "Customer Charge", "amount": 12.00, "unit": "month"}
        ],
        "energy_charges": [
            {"rate_per_kwh": 0.14, "description": "All Usage"}
        ]
    }

    TIERED rate example:
    {
        "fixed_charges": [
            {"name": "Customer Charge", "amount": 10.50, "unit": "month"},
            {"name": "Minimum Bill", "amount": 5.00, "unit": "month", "type": "minimum"}
        ],
        "energy_charges": [
            {"tier": 1, "min_kwh": 0, "max_kwh": 500, "rate_per_kwh": 0.11, "description": "Baseline (0-500 kWh)"},
            {"tier": 2, "min_kwh": 500, "max_kwh": 1000, "rate_per_kwh": 0.15, "description": "101-200% Baseline"},
            {"tier": 3, "min_kwh": 1000, "max_kwh": null, "rate_per_kwh": 0.22, "description": "201%+ Baseline"}
        ],
        "baseline_allocation": {
            "summer_kwh_per_day": 16.5,
            "winter_kwh_per_day": 12.0,
            "all_electric_adder_pct": 40
        }
    }

    TOU rate example:
    {
        "fixed_charges": [
            {"name": "Customer Charge", "amount": 12.00, "unit": "month"}
        ],
        "seasons": {
            "summer": {"months": [6,7,8,9], "label": "Summer (Jun-Sep)"},
            "winter": {"months": [1,2,3,4,5,10,11,12], "label": "Winter (Oct-May)"}
        },
        "periods": [
            {
                "name": "on_peak",
                "label": "On-Peak",
                "schedules": {
                    "summer": {"weekday": [{"start": "16:00", "end": "21:00"}], "weekend": []},
                    "winter": {"weekday": [{"start": "16:00", "end": "21:00"}], "weekend": []}
                },
                "rates": {
                    "summer": {"energy_per_kwh": 0.38, "demand_per_kw": null},
                    "winter": {"energy_per_kwh": 0.28, "demand_per_kw": null}
                }
            },
            {
                "name": "off_peak",
                "label": "Off-Peak",
                "schedules": {
                    "summer": {"weekday": [{"start": "00:00", "end": "16:00"}, {"start": "21:00", "end": "00:00"}], "weekend": [{"start": "00:00", "end": "00:00"}]},
                    "winter": {"weekday": [{"start": "00:00", "end": "16:00"}, {"start": "21:00", "end": "00:00"}], "weekend": [{"start": "00:00", "end": "00:00"}]}
                },
                "rates": {
                    "summer": {"energy_per_kwh": 0.14, "demand_per_kw": null},
                    "winter": {"energy_per_kwh": 0.13, "demand_per_kw": null}
                }
            },
            {
                "name": "super_off_peak",
                "label": "Super Off-Peak",
                "schedules": {
                    "summer": {"weekday": [], "weekend": []},
                    "winter": {"weekday": [{"start": "00:00", "end": "06:00"}], "weekend": [{"start": "00:00", "end": "06:00"}]}
                },
                "rates": {
                    "summer": null,
                    "winter": {"energy_per_kwh": 0.08, "demand_per_kw": null}
                }
            }
        ],
        "nem_export_rates": {
            "on_peak": {"summer": 0.08, "winter": 0.06},
            "off_peak": {"summer": 0.04, "winter": 0.03},
            "super_off_peak": {"winter": 0.02}
        },
        "demand_charges": null
    }

    DEMAND rate example (C&I):
    {
        "fixed_charges": [
            {"name": "Customer Charge", "amount": 45.00, "unit": "month"},
            {"name": "Facility Charge", "amount": 0.50, "unit": "kW", "basis": "contract_demand"}
        ],
        "energy_charges": [
            {"rate_per_kwh": 0.085, "description": "All kWh"}
        ],
        "demand_charges": [
            {"name": "On-Peak Demand", "rate_per_kw": 18.50, "measurement_window": "15_min", "period": "on_peak", "ratchet_months": 11},
            {"name": "Non-Coincident Demand", "rate_per_kw": 6.25, "measurement_window": "15_min", "period": "all"}
        ],
        "reactive_charges": {
            "power_factor_threshold": 0.90,
            "penalty_per_kvar": 0.50
        }
    }
    */

    -- JSONB: metadata from OpenEI URDB import
    openei_metadata     JSONB,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_plans_type ON rate_plans (rate_type);
CREATE INDEX idx_rate_plans_class ON rate_plans (customer_class);
CREATE INDEX idx_rate_plans_effective ON rate_plans (effective_date, expiration_date);
-- GIN index for querying rate structure properties
CREATE INDEX idx_rate_plans_structure ON rate_plans USING GIN (rate_structure jsonb_path_ops);

-- =============================================================
-- CUSTOMER RATE ENROLMENTS
-- =============================================================

CREATE TABLE customer_rate_enrolments (
    enrolment_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    rate_plan_id        UUID NOT NULL REFERENCES rate_plans(rate_plan_id),
    effective_date      DATE NOT NULL,
    end_date            DATE,
    status              VARCHAR(20) DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_enrolments_customer ON customer_rate_enrolments (customer_id, status);
```

### 5. Billing and Payment Domain

```sql
-- =============================================================
-- BILLS — relational core with JSONB for line items and AI analysis
-- =============================================================

CREATE TABLE bills (
    bill_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    rate_plan_id        UUID REFERENCES rate_plans(rate_plan_id),
    external_bill_id    VARCHAR(50),
    billing_period_start DATE NOT NULL,
    billing_period_end   DATE NOT NULL,
    statement_date      DATE NOT NULL,
    due_date            DATE NOT NULL,
    total_kwh           NUMERIC(12,3),
    total_generation_kwh NUMERIC(12,3),
    net_kwh             NUMERIC(12,3),
    peak_demand_kw      NUMERIC(10,3),
    total_amount_due    NUMERIC(10,2) NOT NULL,
    previous_balance    NUMERIC(10,2),
    payments_received   NUMERIC(10,2),
    status              VARCHAR(20) NOT NULL DEFAULT 'issued' CHECK (status IN ('draft', 'issued', 'paid', 'partial', 'overdue', 'cancelled', 'disputed')),

    -- JSONB: itemised line items (variable structure per rate plan)
    line_items          JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*
    [
        {"order": 1, "category": "energy", "description": "On-Peak Energy (250 kWh x $0.38)", "quantity": 250, "unit": "kWh", "rate": 0.38, "amount": 95.00, "period": "on_peak"},
        {"order": 2, "category": "energy", "description": "Off-Peak Energy (500 kWh x $0.14)", "quantity": 500, "unit": "kWh", "rate": 0.14, "amount": 70.00, "period": "off_peak"},
        {"order": 3, "category": "fixed", "description": "Customer Charge", "amount": 12.00},
        {"order": 4, "category": "nem_credit", "description": "NEM Export Credit", "amount": -18.50},
        {"order": 5, "category": "tax", "description": "State Tax (2.5%)", "rate": 0.025, "amount": 3.96},
        {"order": 6, "category": "fee", "description": "Public Purpose Programs Charge", "amount": 4.20}
    ]
    */

    -- JSONB: AI-generated bill analysis
    ai_analysis         JSONB,
    /*
    {
        "explanation": "Your bill this month is $166.66. The main driver is higher on-peak usage...",
        "high_bill_flag": true,
        "high_bill_reasons": [
            {"factor": "temperature", "description": "Average temperature was 95°F vs 88°F last month", "impact_pct": 35},
            {"factor": "on_peak_usage", "description": "On-peak usage increased 22%", "impact_pct": 40},
            {"factor": "rate_change", "description": "Summer rates took effect June 1", "impact_pct": 25}
        ],
        "savings_tips": [
            "Shift laundry and dishwasher to off-peak hours (after 9 PM) to save ~$12/month",
            "Your pool pump runs during on-peak hours. Switching to off-peak could save ~$8/month"
        ],
        "model_version": "bill-explainer-v2.1",
        "confidence": 0.89
    }
    */

    -- JSONB: rate comparison snapshot at bill time
    rate_comparison     JSONB,
    /*
    {
        "current_plan": {"code": "TOU-D-A", "cost": 166.66},
        "alternatives": [
            {"code": "TOU-D-B", "cost": 152.30, "savings": 14.36, "savings_pct": 8.6},
            {"code": "EV-TOU-5", "cost": 148.90, "savings": 17.76, "savings_pct": 10.7}
        ],
        "recommendation": "EV-TOU-5",
        "analysis_date": "2026-06-01"
    }
    */

    pdf_url             VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bills_customer ON bills (customer_id, statement_date DESC);
CREATE INDEX idx_bills_status ON bills (status) WHERE status IN ('issued', 'overdue');
CREATE INDEX idx_bills_due_date ON bills (due_date) WHERE status NOT IN ('paid', 'cancelled');
-- Expression index for high-bill AI flag
CREATE INDEX idx_bills_high_bill ON bills ((ai_analysis->>'high_bill_flag')) WHERE ai_analysis->>'high_bill_flag' = 'true';

-- =============================================================
-- PAYMENTS — mostly relational (PCI compliance demands explicit structure)
-- =============================================================

CREATE TABLE payments (
    payment_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    bill_id             UUID REFERENCES bills(bill_id),
    amount              NUMERIC(10,2) NOT NULL CHECK (amount > 0),
    payment_method      VARCHAR(20) NOT NULL CHECK (payment_method IN ('card', 'ach', 'autopay_card', 'autopay_ach', 'cash_kiosk', 'check', 'wallet')),
    payment_token       VARCHAR(255),
    card_last_four      VARCHAR(4),
    card_brand          VARCHAR(20),
    processor_txn_id    VARCHAR(100),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'processing', 'completed', 'failed', 'refunded', 'reversed')),
    failure_reason      VARCHAR(255),
    processed_at        TIMESTAMPTZ,

    -- JSONB: processor-specific response metadata (varies by payment processor)
    processor_metadata  JSONB,
    /*
    {
        "processor": "stripe",
        "charge_id": "ch_3abc123",
        "receipt_url": "https://pay.stripe.com/receipts/...",
        "risk_score": 12,
        "risk_level": "normal",
        "authorization_code": "A12345"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_customer ON payments (customer_id, created_at DESC);
CREATE INDEX idx_payments_bill ON payments (bill_id);

-- =============================================================
-- AUTOPAY AND PAYMENT PLANS
-- =============================================================

CREATE TABLE autopay_enrolments (
    enrolment_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    payment_method      VARCHAR(20) NOT NULL,
    payment_token       VARCHAR(255) NOT NULL,
    card_last_four      VARCHAR(4),
    card_brand          VARCHAR(20),
    card_expiry         VARCHAR(7),
    status              VARCHAR(20) DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE payment_plans (
    plan_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    plan_type           VARCHAR(20) NOT NULL,
    monthly_amount      NUMERIC(10,2) NOT NULL,
    start_date          DATE NOT NULL,
    end_date            DATE,
    balance             NUMERIC(10,2) DEFAULT 0,
    status              VARCHAR(20) DEFAULT 'active',

    -- JSONB: plan-specific terms (varies by plan type and assistance programme)
    plan_terms          JSONB,
    /*
    Budget billing:
    {
        "reconciliation_date": "2027-06-01",
        "calculation_method": "12_month_rolling_average",
        "adjustment_threshold_pct": 25
    }

    Low-income assistance:
    {
        "programme_name": "CARE",
        "discount_pct": 30,
        "income_verification_date": "2026-01-15",
        "household_size": 4,
        "renewal_date": "2027-01-15"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6. Net Metering Domain

```sql
-- =============================================================
-- NET METERING ACCOUNTS — relational core with JSONB for system details
-- =============================================================

CREATE TABLE nem_accounts (
    nem_account_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    usage_point_id      UUID NOT NULL REFERENCES usage_points(usage_point_id),
    nem_programme       VARCHAR(30) NOT NULL,
    interconnection_date DATE NOT NULL,
    system_size_kw      NUMERIC(10,3) NOT NULL,
    annual_cycle_start  DATE NOT NULL,
    status              VARCHAR(20) DEFAULT 'active',

    -- JSONB: solar system and DER details (varies by installation)
    system_details      JSONB,
    /*
    {
        "panels": {
            "count": 20,
            "manufacturer": "SunPower",
            "model": "Maxeon 6",
            "watt_per_panel": 440,
            "orientation": "south",
            "tilt_degrees": 25
        },
        "inverter": {
            "manufacturer": "Enphase",
            "model": "IQ8+",
            "count": 20,
            "type": "micro",
            "max_output_w": 300
        },
        "battery": {
            "installed": true,
            "manufacturer": "Tesla",
            "model": "Powerwall 3",
            "capacity_kwh": 13.5,
            "count": 2,
            "total_capacity_kwh": 27.0,
            "max_discharge_kw": 11.5
        },
        "expected_annual_generation_kwh": 11500,
        "installer": "SunRun",
        "permit_number": "PV-2025-1234",
        "interconnection_agreement_id": "IA-2025-5678"
    }
    */

    -- JSONB: jurisdiction-specific NEM rules (vary by state/utility/programme version)
    nem_rules           JSONB,
    /*
    {
        "programme_version": "NEM 2.0",
        "jurisdiction": "CPUC",
        "utility": "PG&E",
        "export_compensation": "retail_rate_with_tou",
        "nbc_charges_apply": true,
        "nbc_rate_per_kwh": 0.023,
        "minimum_bill_monthly": 10.00,
        "true_up_frequency": "annual",
        "nsc_rate_per_kwh": 0.04,
        "capacity_limit_kw": null,
        "grandfathering_expiry": "2046-04-15"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_nem_customer ON nem_accounts (customer_id);

-- =============================================================
-- NEM MONTHLY SETTLEMENTS
-- =============================================================

CREATE TABLE nem_monthly_settlements (
    settlement_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    nem_account_id      UUID NOT NULL REFERENCES nem_accounts(nem_account_id),
    settlement_month    DATE NOT NULL,
    total_import_kwh    NUMERIC(12,3) NOT NULL,
    total_export_kwh    NUMERIC(12,3) NOT NULL,
    net_kwh             NUMERIC(12,3) NOT NULL,
    credit_earned       NUMERIC(10,2) NOT NULL,
    credit_used         NUMERIC(10,2) DEFAULT 0,
    credit_balance      NUMERIC(10,2) NOT NULL,

    -- JSONB: detailed TOU export breakdown (varies by rate plan)
    export_breakdown    JSONB,
    /*
    {
        "on_peak": {"kwh": 210.5, "rate": 0.08, "credit": 16.84},
        "off_peak": {"kwh": 280.3, "rate": 0.04, "credit": 11.21},
        "super_off_peak": {"kwh": 89.3, "rate": 0.02, "credit": 1.79},
        "nbc_charges": {"kwh": 420.3, "rate": 0.023, "amount": 9.67}
    }
    */

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
    net_amount          NUMERIC(12,2) NOT NULL,
    nsc_payout          NUMERIC(12,2) DEFAULT 0,
    status              VARCHAR(20) DEFAULT 'calculated',

    -- JSONB: detailed true-up calculation (for transparency and dispute resolution)
    calculation_detail  JSONB,
    /*
    {
        "monthly_summaries": [
            {"month": "2025-04", "import_kwh": 380, "export_kwh": 520, "charges": 28.50, "credits": 35.20, "net": -6.70},
            ...
        ],
        "rate_changes_during_cycle": [
            {"effective_date": "2025-10-01", "change_description": "Winter rates applied"}
        ],
        "nbc_total": 115.40,
        "minimum_bill_total": 120.00,
        "methodology": "NEM 2.0 CPUC D.16-01-044"
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 7. Outage Domain

```sql
-- =============================================================
-- OUTAGE EVENTS — relational status tracking, JSONB for OMS-specific data
-- =============================================================

CREATE TABLE outage_events (
    outage_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_oms_id     VARCHAR(100) UNIQUE,
    outage_type         VARCHAR(30) NOT NULL,
    cause               VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    customers_affected  INTEGER,
    start_time          TIMESTAMPTZ NOT NULL,
    estimated_restoration TIMESTAMPTZ,
    actual_restoration  TIMESTAMPTZ,

    -- PostGIS geometry for spatial queries
    affected_area_geom  GEOMETRY(MultiPolygon, 4326),

    -- JSONB: OMS-vendor-specific metadata, crew details, damage assessment
    -- Structure varies dramatically across OMS vendors (Schneider, ABB, GE, Oracle)
    oms_metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    /*
    {
        "cause_detail": "60-foot pine tree fell across 12kV primary line",
        "severity": "major",
        "weather_conditions": {
            "wind_speed_mph": 45,
            "precipitation": "heavy_rain",
            "lightning": true
        },
        "crews": [
            {"crew_id": "CREW-42", "status": "on_site", "type": "line_crew", "eta_minutes": 0},
            {"crew_id": "CREW-57", "status": "en_route", "type": "tree_crew", "eta_minutes": 25}
        ],
        "equipment_damaged": [
            {"type": "transformer", "asset_id": "TX-5432", "description": "25kVA pole-mount transformer"},
            {"type": "conductor", "description": "200ft 4/0 ACSR primary conductor"}
        ],
        "switching_operations": [
            {"time": "2026-05-25T15:10:00Z", "operation": "open", "device": "SW-1234", "purpose": "isolate fault"},
            {"time": "2026-05-25T15:15:00Z", "operation": "close", "device": "TIE-5678", "purpose": "restore upstream section"}
        ],
        "customer_callbacks_requested": 42,
        "ai_eta_prediction": {
            "model": "outage-eta-v3",
            "predicted_minutes": 240,
            "confidence": 0.72,
            "factors": ["transformer_replacement", "tree_removal", "crew_availability"]
        }
    }
    */

    -- JSONB: timeline of status updates (append-only within the JSON)
    status_timeline     JSONB NOT NULL DEFAULT '[]'::jsonb,
    /*
    [
        {"time": "2026-05-25T14:30:00Z", "status": "active", "note": "Outage detected via AMI last-gasp"},
        {"time": "2026-05-25T14:45:00Z", "status": "crew_assigned", "note": "Line crew CREW-42 dispatched"},
        {"time": "2026-05-25T15:30:00Z", "status": "crew_on_site", "note": "Crew arrived, assessing damage"},
        {"time": "2026-05-25T16:00:00Z", "status": "assessing", "note": "Transformer replacement required, ETA extended to 20:00"},
        {"time": "2026-05-25T19:45:00Z", "status": "restored", "note": "All customers restored"}
    ]
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outages_status ON outage_events (status) WHERE status != 'restored';
CREATE INDEX idx_outages_geom ON outage_events USING GIST (affected_area_geom);
CREATE INDEX idx_outages_time ON outage_events (start_time DESC);
-- Expression index for outage cause from OMS metadata
CREATE INDEX idx_outages_severity ON outage_events USING GIN ((oms_metadata->'severity'));

-- =============================================================
-- CUSTOMER OUTAGE REPORTS
-- =============================================================

CREATE TABLE customer_outage_reports (
    report_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    outage_id           UUID REFERENCES outage_events(outage_id),
    report_type         VARCHAR(30) DEFAULT 'no_power',
    reported_via        VARCHAR(20) DEFAULT 'portal',
    status              VARCHAR(20) DEFAULT 'received',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at         TIMESTAMPTZ
);
```

### 8. Programme and DR Domain

```sql
-- =============================================================
-- PROGRAMMES — relational identity, JSONB for eligibility rules and terms
-- =============================================================

CREATE TABLE programmes (
    programme_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_code      VARCHAR(50) UNIQUE NOT NULL,
    programme_name      VARCHAR(200) NOT NULL,
    programme_type      VARCHAR(30) NOT NULL,
    status              VARCHAR(20) DEFAULT 'active',
    start_date          DATE,
    end_date            DATE,
    max_enrolments      INTEGER,
    current_enrolments  INTEGER DEFAULT 0,

    -- JSONB: programme details, eligibility rules, incentives
    -- Every programme has different rules, different incentive structures
    programme_details   JSONB NOT NULL,
    /*
    Demand Response programme example:
    {
        "description": "Reduce load during peak events to earn bill credits",
        "eligibility": {
            "customer_types": ["residential"],
            "min_demand_kw": null,
            "requires_smart_thermostat": true,
            "compatible_devices": ["Nest", "Ecobee", "Honeywell T10"],
            "service_territories": ["zone_a", "zone_b"]
        },
        "incentives": {
            "type": "bill_credit",
            "sign_up_bonus": 50.00,
            "per_event_credit": 5.00,
            "annual_cap": 100.00
        },
        "event_parameters": {
            "max_events_per_year": 15,
            "max_duration_hours": 4,
            "min_advance_notice_hours": 24,
            "temperature_override_threshold_f": 78
        },
        "opt_out_rules": {
            "max_opt_outs_per_year": 3,
            "opt_out_penalty": "none"
        }
    }

    Rebate programme example:
    {
        "description": "Rebates for ENERGY STAR appliance purchases",
        "eligibility": {
            "customer_types": ["residential", "commercial"],
            "active_account_required": true,
            "income_qualified_only": false
        },
        "rebate_items": [
            {"item": "Heat Pump Water Heater", "rebate": 500, "max_per_customer": 1, "certification": "ENERGY STAR"},
            {"item": "Smart Thermostat", "rebate": 75, "max_per_customer": 2, "certification": "ENERGY STAR"},
            {"item": "Heat Pump HVAC", "rebate": 1500, "max_per_customer": 1, "certification": "ENERGY STAR", "requires_inspection": true}
        ],
        "budget": {"total": 2000000, "remaining": 1450000, "fiscal_year": "2026"}
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_programmes_type ON programmes (programme_type, status);

-- =============================================================
-- PROGRAMME ENROLMENTS
-- =============================================================

CREATE TABLE programme_enrolments (
    enrolment_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id        UUID NOT NULL REFERENCES programmes(programme_id),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID REFERENCES service_locations(location_id),
    enrolled_via        VARCHAR(20) DEFAULT 'portal',
    status              VARCHAR(20) DEFAULT 'active',
    enrolled_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- JSONB: enrolment-specific data (devices registered, application details)
    enrolment_data      JSONB,
    /*
    DR enrolment:
    {
        "devices": [
            {"type": "thermostat", "make": "Nest", "model": "Learning 4th Gen", "device_id": "NEST-abc123"}
        ]
    }

    Rebate application:
    {
        "items_claimed": [
            {"item": "Heat Pump Water Heater", "make": "Rheem", "model": "ProTerra", "purchase_date": "2026-04-10", "receipt_url": "s3://receipts/abc.pdf", "rebate_amount": 500}
        ],
        "application_status": "approved",
        "payment_method": "bill_credit",
        "approved_at": "2026-04-25"
    }
    */

    UNIQUE (programme_id, customer_id, location_id)
);

-- =============================================================
-- DR EVENTS AND PARTICIPATION
-- =============================================================

CREATE TABLE dr_events (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id        UUID NOT NULL REFERENCES programmes(programme_id),
    openadr_event_id    VARCHAR(100),
    start_time          TIMESTAMPTZ NOT NULL,
    end_time            TIMESTAMPTZ NOT NULL,
    status              VARCHAR(20) DEFAULT 'scheduled',

    -- JSONB: OpenADR signal details and event parameters
    event_details       JSONB NOT NULL,
    /*
    {
        "event_type": "load_shed",
        "signal_type": "level",
        "signal_value": 2,
        "notification_time": "2026-07-15T10:00:00Z",
        "reason": "Excessive heat forecast; grid stress expected 14:00-18:00",
        "temperature_forecast_f": 108,
        "grid_load_forecast_mw": 45200,
        "openadr_payload": { ... }
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE dr_event_participation (
    participation_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id            UUID NOT NULL REFERENCES dr_events(event_id),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    opt_status          VARCHAR(20) DEFAULT 'enrolled',
    baseline_kwh        NUMERIC(10,3),
    actual_kwh          NUMERIC(10,3),
    reduction_kwh       NUMERIC(10,3),
    incentive_earned    NUMERIC(10,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 9. EV, Service Orders, and Green Button

```sql
-- =============================================================
-- EV CHARGERS — relational identity, JSONB for charger capabilities
-- =============================================================

CREATE TABLE ev_chargers (
    charger_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    ocpp_charge_point_id VARCHAR(100),
    max_power_kw        NUMERIC(6,2),
    status              VARCHAR(20) DEFAULT 'active',

    -- JSONB: charger-specific details (varies by manufacturer and protocol)
    charger_details     JSONB,
    /*
    {
        "make": "ChargePoint",
        "model": "Home Flex",
        "firmware": "3.1.2",
        "connector_type": "nacs",
        "is_v2g_capable": false,
        "communication": "ocpp_2.1",
        "ocpp_config": {
            "heartbeat_interval_sec": 300,
            "meter_values_interval_sec": 60,
            "max_current_amps": 50
        },
        "schedules": [
            {"name": "Weeknight Off-Peak", "days": [1,2,3,4,5], "start": "22:00", "end": "06:00", "max_rate_kw": 7.2, "prefer_off_peak": true}
        ]
    }
    */

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
    rate_period         VARCHAR(30),
    was_dr_curtailed    BOOLEAN DEFAULT FALSE,
    status              VARCHAR(20) DEFAULT 'in_progress',

    -- JSONB: OCPP session metadata
    session_metadata    JSONB,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- SERVICE ORDERS
-- =============================================================

CREATE TABLE service_orders (
    order_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID REFERENCES service_locations(location_id),
    external_order_id   VARCHAR(50),
    order_type          VARCHAR(30) NOT NULL,
    requested_date      DATE NOT NULL,
    scheduled_date      DATE,
    completed_date      DATE,
    status              VARCHAR(20) DEFAULT 'submitted',
    submitted_via       VARCHAR(20) DEFAULT 'portal',

    -- JSONB: order-specific details (varies by order type)
    order_details       JSONB,
    /*
    Transfer of service:
    {
        "new_address": {"line1": "456 Oak Ave", "city": "Sacramento", "state": "CA", "zip": "95814"},
        "current_service_end_date": "2026-06-15",
        "new_service_start_date": "2026-06-16",
        "final_read_requested": true
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- GREEN BUTTON AUTHORISATIONS
-- =============================================================

CREATE TABLE green_button_authorisations (
    auth_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    usage_point_id      UUID NOT NULL REFERENCES usage_points(usage_point_id),
    third_party_name    VARCHAR(200) NOT NULL,
    third_party_id      VARCHAR(100) NOT NULL,
    scope               VARCHAR(50) NOT NULL DEFAULT 'usage_data',
    authorised_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ,
    revoked_at          TIMESTAMPTZ,
    status              VARCHAR(20) DEFAULT 'active',
    UNIQUE (customer_id, usage_point_id, third_party_id)
);

-- =============================================================
-- AI INSIGHTS — fully JSONB (insights vary wildly in structure)
-- =============================================================

CREATE TABLE ai_insights (
    insight_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id         UUID NOT NULL REFERENCES customers(customer_id),
    location_id         UUID REFERENCES service_locations(location_id),
    insight_type        VARCHAR(30) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_read             BOOLEAN DEFAULT FALSE,
    is_dismissed        BOOLEAN DEFAULT FALSE,

    -- JSONB: the insight content (varies by insight type)
    content             JSONB NOT NULL,
    /*
    Rate recommendation:
    {
        "title": "You could save $210/year by switching to TOU-D-B",
        "current_plan": "TOU-D-A",
        "recommended_plan": "TOU-D-B",
        "annual_savings": 210.40,
        "analysis_period": "2025-06 to 2026-05",
        "key_factors": [
            "62% of your usage is already in off-peak hours",
            "Your EV charges overnight, perfectly aligned with off-peak"
        ],
        "confidence": 0.94,
        "model_version": "rate-optimiser-v3.2"
    }

    Usage anomaly:
    {
        "title": "Unusual usage spike detected yesterday",
        "date": "2026-05-24",
        "actual_kwh": 42.5,
        "expected_kwh": 28.0,
        "deviation_pct": 51.8,
        "possible_causes": [
            {"cause": "HVAC running continuously", "probability": 0.65},
            {"cause": "Additional appliance load", "probability": 0.25}
        ],
        "model_version": "anomaly-detector-v2.0"
    }
    */
);

CREATE INDEX idx_ai_insights_customer ON ai_insights (customer_id, created_at DESC);
CREATE INDEX idx_ai_insights_unread ON ai_insights (customer_id) WHERE NOT is_read AND NOT is_dismissed;
```

### 10. Audit Log

```sql
-- =============================================================
-- AUDIT LOG — captures all state-changing actions for compliance
-- =============================================================

CREATE TABLE audit_log (
    log_id              BIGSERIAL PRIMARY KEY,
    entity_type         VARCHAR(50) NOT NULL,                   -- 'customer', 'bill', 'payment', etc.
    entity_id           UUID NOT NULL,
    action              VARCHAR(30) NOT NULL,                   -- 'create', 'update', 'delete', 'login', etc.
    actor_type          VARCHAR(20) NOT NULL,                   -- 'customer', 'system', 'admin', 'cis_sync'
    actor_id            VARCHAR(100),
    ip_address          INET,

    -- JSONB: what changed
    changes             JSONB,
    /*
    {
        "field": "status",
        "old_value": "issued",
        "new_value": "paid"
    }
    -- or for complex changes:
    {
        "fields_changed": ["phone_primary", "email"],
        "old_values": {"phone_primary": "555-0100", "email": "old@example.com"},
        "new_values": {"phone_primary": "555-0200", "email": "new@example.com"}
    }
    */

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_entity ON audit_log (entity_type, entity_id, created_at DESC);
CREATE INDEX idx_audit_actor ON audit_log (actor_id, created_at DESC);
```

---

## Pros and Cons

### Pros

1. **Best of both worlds.** Stable data gets relational integrity (foreign keys, check constraints, strong types). Variable data gets schema flexibility (JSONB). No need to choose one paradigm.

2. **Accommodates utility diversity.** The single biggest challenge in building a multi-utility portal is that rate plans, NEM rules, programme eligibility, and OMS metadata differ across every deployment. JSONB absorbs this variability without DDL changes. A new utility with a novel rate structure can be onboarded by defining a new JSONB document — no schema migration.

3. **Single database engine.** PostgreSQL handles both relational and document storage. No need for a separate MongoDB or document store, reducing operational complexity and infrastructure cost.

4. **Queryable documents.** PostgreSQL JSONB supports rich querying (containment, existence, path extraction), GIN indexing for full-document search, and expression indexes for specific paths. You can JOIN relational tables with JSONB-extracted values.

5. **Incremental adoption.** Start with JSONB for highly variable domains (rate plans, OMS metadata) and use traditional columns everywhere else. Move data from JSONB to columns as patterns stabilise — a column is always faster than a JSONB path extraction.

6. **AI insights store.** AI-generated content has inherently variable structure (different models produce different output formats, new insight types are added constantly). JSONB is the natural fit rather than creating a new table for each insight type.

7. **Reduced migration friction.** Adding a new field to a JSONB column requires no DDL migration, no downtime, and no application redeployment. This matters for a portal that integrates with multiple utility back-office systems, each with different data models.

### Cons

1. **No foreign keys on JSONB values.** You cannot create a foreign key from a JSONB field to another table. If your JSONB contains references to other entities (e.g., a device_id inside programme enrolment data), referential integrity is enforced only in application code.

2. **JSONB query performance nuances.** While GIN-indexed JSONB queries are fast for containment (`@>`) and existence (`?`) operators, complex path extractions (`payload->>'nested'->>'field'`) are slower than typed column access. Expression indexes mitigate this but must be created proactively.

3. **Schema validation is application-side.** Unlike typed columns with CHECK constraints, JSONB documents can contain any structure. While JSON Schema validation can be added as CHECK constraints or via triggers, it adds development overhead. Without discipline, JSONB columns can accumulate inconsistent data.

4. **Tooling gaps.** ORM support for JSONB varies. Some frameworks treat JSONB as opaque blobs, losing the queryability benefit. SQLAlchemy and Django ORM have good JSONB support; others may not.

5. **Reporting complexity.** Business intelligence tools and reporting queries often struggle with JSONB. Extracting rate plan details from JSONB for regulatory reporting requires custom SQL (jsonb_to_recordset, jsonb_array_elements, etc.) rather than simple SELECT statements.

6. **Storage overhead.** JSONB includes field names in every row, unlike typed columns where the column name is metadata. For high-volume tables like interval readings, this overhead matters — hence keeping interval_readings fully relational.

7. **Risk of JSONB overuse.** Without discipline, teams may push too much data into JSONB columns, losing the benefits of relational integrity. Clear guidelines on what belongs in columns vs. JSONB are essential.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with PostGIS 3.4+ |
| **JSONB validation** | JSON Schema validation via pg_jsonschema extension or application-layer validation |
| **ORM** | SQLAlchemy (Python) or Prisma (Node.js) — both have strong JSONB query support |
| **Migration tool** | Alembic (Python) or Prisma Migrate — handle both DDL and JSONB schema evolution |
| **JSONB indexing strategy** | GIN (jsonb_path_ops) for containment queries; expression indexes for specific high-traffic paths |
| **Connection pooling** | PgBouncer in transaction mode |
| **Caching** | Redis for session data, outage map cache, bill summaries |
| **Monitoring** | Track JSONB column sizes, query plans on JSONB paths, index usage |

---

## Migration and Scaling Considerations

### Migrating from a Fully Relational Schema

1. **Identify variable domains.** Audit which tables have the most nullable columns, the most frequent schema changes, or EAV patterns. These are candidates for JSONB consolidation.
2. **Column-to-JSONB migration.** Move variable columns into a JSONB column with a reversible migration. Keep the original columns temporarily for validation, then drop them after verification.
3. **Validation parity.** Replicate any CHECK constraints from the original columns as JSON Schema validation on the JSONB column. Test edge cases.

### JSONB Column Sizing

| Table | Estimated JSONB Size per Row |
|-------|------------------------------|
| customers.preferences | 500 bytes - 2 KB |
| rate_plans.rate_structure | 2 KB - 20 KB (complex TOU) |
| bills.line_items | 500 bytes - 5 KB |
| bills.ai_analysis | 1 KB - 10 KB |
| outage_events.oms_metadata | 2 KB - 50 KB |
| programmes.programme_details | 1 KB - 10 KB |
| nem_accounts.system_details | 1 KB - 5 KB |

### Scaling Strategy

1. **TOAST compression.** PostgreSQL automatically TOAST-compresses JSONB values larger than ~2 KB, keeping the main table compact. This happens transparently.
2. **Selective indexing.** Do not GIN-index every JSONB column. Index only the columns and paths used in WHERE clauses. Monitor index usage with pg_stat_user_indexes.
3. **Materialised views for reporting.** Create materialised views that extract and flatten JSONB data for BI tools. Refresh on a schedule (e.g., hourly) to avoid impacting transactional queries.
4. **JSONB column evolution protocol.** Establish a convention: document the expected JSONB structure in code comments and a schema registry. Use application-layer validation (e.g., Pydantic models, Zod schemas) to enforce structure. Treat JSONB schema changes like API contract changes — versioned and reviewed.

### When to Promote JSONB to Columns

Monitor JSONB usage patterns. If a specific JSONB path is:
- Used in more than 30% of queries against that table
- Used in JOIN conditions
- Required for foreign-key-like relationships
- Needed in ORDER BY or GROUP BY frequently

...then promote it to a dedicated column. This is a standard, reversible migration.

---

## Summary

The hybrid relational + JSONB model is the most pragmatic choice for a multi-utility customer portal. It delivers relational integrity where it matters most (customer identity, billing, payments, metering) while absorbing the enormous variability of the utility domain (rate plans, NEM rules, OMS metadata, programme structures) in queryable JSONB documents. The key risk is discipline — teams must resist the temptation to put everything in JSONB. With clear conventions on what belongs in columns vs. JSONB, this model provides the best balance of integrity, flexibility, and operational simplicity for a portal that must serve utilities with wildly different rate structures, programme offerings, and back-office systems.
