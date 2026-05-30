# Data Model Suggestion 4: Time-Series Specialised Model (PostgreSQL + TimescaleDB + PostGIS)

> Project: Electric Utility Customer Portal · Generated: 2026-05-26

---

## Overview

This model is purpose-built for the electric utility domain by pairing PostgreSQL with two specialised extensions: **TimescaleDB** for high-volume time-series meter data and **PostGIS** for geospatial outage mapping. The core insight is that an electric utility customer portal has two fundamentally different data workloads that strain a single general-purpose database:

1. **High-volume, high-velocity time-series data.** 15-minute interval readings from AMI smart meters produce tens of millions of rows per day for a mid-sized utility. These data need time-range queries, time-bucketed aggregations, compression, and continuous aggregate materialisation — all areas where TimescaleDB dramatically outperforms vanilla PostgreSQL partitioning.

2. **Geospatial outage data with real-time query load.** During major storms, thousands of concurrent users query the outage map, performing spatial containment queries (which customers are affected by which outage polygon) and spatial aggregations (how many customers affected per region). PostGIS provides the spatial indexing engine.

Rather than introducing separate database products (e.g., InfluxDB for time-series, Neo4j for network topology), this model uses PostgreSQL extensions so that the entire data layer speaks SQL, participates in transactions, uses the same connection pool, and is managed by a single operations team.

---

## Why This Architecture Fits the Utility Domain

### The Meter Data Challenge

A utility with 500,000 AMI meters producing 15-minute readings generates:

| Metric | Volume |
|--------|--------|
| Readings per meter per day | 96 |
| Daily readings (500K meters) | 48,000,000 |
| Monthly readings | ~1.44 billion |
| Annual readings | ~17.5 billion |
| Storage per reading (uncompressed) | ~120 bytes |
| Daily storage (uncompressed) | ~5.5 GB |
| Annual storage (uncompressed) | ~2 TB |
| Annual storage (TimescaleDB compressed, ~10x) | ~200 GB |

Vanilla PostgreSQL can partition this data by month but cannot:
- Automatically manage chunk creation, compression, and retention
- Provide `time_bucket()` for efficient time-range aggregations
- Create continuous aggregates that automatically update as data arrives
- Handle late-arriving or out-of-order data gracefully
- Compress old chunks in the background while keeping them queryable

TimescaleDB provides all of these as PostgreSQL extension functions with no application code changes.

### The Outage Map Challenge

During a major storm (e.g., a hurricane or derecho), an electric utility experiences:

| Metric | Typical Storm |
|--------|--------------|
| Active outage events | 50-500+ |
| Customers affected | 50,000 - 2,000,000 |
| Concurrent portal users | 10,000 - 100,000+ |
| Outage map queries per second | 500 - 5,000+ |
| Spatial operations per query | Point-in-polygon, polygon overlap, nearest crew |

PostGIS with GiST spatial indexing handles these operations efficiently, and when combined with Redis caching for tile-based map rendering, can serve the extreme read load of a major storm event.

---

## Schema Design

### 1. Core Relational Tables (Standard PostgreSQL)

The customer, billing, payment, and programme tables use the same relational design as Suggestion 1, with JSONB where appropriate (as in Suggestion 3). The following tables are abbreviated — see Suggestions 1 and 3 for full definitions.

```sql
-- =============================================================
-- CUSTOMERS, LOCATIONS, METERS — standard relational tables
-- =============================================================

CREATE TABLE customers (
    customer_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_cis_id     VARCHAR(50) UNIQUE,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    email               VARCHAR(255) UNIQUE NOT NULL,
    phone_primary       VARCHAR(20),
    customer_type       VARCHAR(20) NOT NULL CHECK (customer_type IN ('residential', 'commercial', 'industrial', 'agricultural')),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    preferences         JSONB NOT NULL DEFAULT '{}'::jsonb,
    security_metadata   JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

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
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    premise_attributes  JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_locations_geom ON service_locations USING GIST (geom);

CREATE TABLE meters (
    meter_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    meter_serial        VARCHAR(50) UNIQUE NOT NULL,
    meter_type          VARCHAR(20) NOT NULL CHECK (meter_type IN ('ami', 'amr', 'manual', 'net_meter')),
    is_bidirectional    BOOLEAN DEFAULT FALSE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    install_date        DATE,
    meter_config        JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE usage_points (
    usage_point_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meter_id            UUID NOT NULL REFERENCES meters(meter_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    green_button_id     VARCHAR(100) UNIQUE,
    status              VARCHAR(20) DEFAULT 'active',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 2. TimescaleDB Hypertables for Meter Data

```sql
-- =============================================================
-- INTERVAL READINGS — TimescaleDB hypertable
-- This is the highest-volume table in the entire system
-- =============================================================

CREATE TABLE interval_readings (
    time                TIMESTAMPTZ NOT NULL,                   -- TimescaleDB convention: primary time column named 'time'
    usage_point_id      UUID NOT NULL,
    consumption_wh      NUMERIC(12,2) NOT NULL,
    generation_wh       NUMERIC(12,2) DEFAULT 0,
    net_wh              NUMERIC(12,2) GENERATED ALWAYS AS (consumption_wh - generation_wh) STORED,
    demand_w            NUMERIC(10,2),
    voltage_v           NUMERIC(7,2),
    power_factor        NUMERIC(5,4),
    reading_quality     SMALLINT DEFAULT 0,                     -- 0=actual, 1=estimated, 2=validated, 3=edited (integer for compression)
    source              SMALLINT DEFAULT 0                      -- 0=mdms, 1=green_button, 2=manual, 3=estimated
);

-- Convert to hypertable with 1-day chunks
-- Partition by usage_point_id for efficient per-meter queries
SELECT create_hypertable(
    'interval_readings',
    by_range('time', INTERVAL '1 day'),
    create_default_indexes => FALSE
);

-- Add space partitioning for parallel query performance
SELECT add_dimension('interval_readings', by_hash('usage_point_id', 16));

-- Create optimised indexes
CREATE INDEX idx_readings_point_time ON interval_readings (usage_point_id, time DESC);
CREATE INDEX idx_readings_time ON interval_readings (time DESC);

-- =============================================================
-- COMPRESSION POLICY
-- Compress data older than 7 days for ~10-20x storage reduction
-- Compressed data remains fully queryable
-- =============================================================

ALTER TABLE interval_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'usage_point_id',
    timescaledb.compress_orderby = 'time DESC',
    timescaledb.compress_chunk_time_interval = '1 day'
);

SELECT add_compression_policy('interval_readings', INTERVAL '7 days');

-- =============================================================
-- RETENTION POLICY
-- Automatically drop data older than 5 years (regulatory minimum)
-- =============================================================

SELECT add_retention_policy('interval_readings', INTERVAL '5 years');

-- =============================================================
-- CONTINUOUS AGGREGATES — automatically maintained materialised views
-- These replace the manually-managed summary tables from Suggestion 1
-- =============================================================

-- HOURLY AGGREGATE (for intra-day usage charts)
CREATE MATERIALIZED VIEW hourly_usage
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    usage_point_id,
    SUM(consumption_wh) / 1000.0 AS consumption_kwh,
    SUM(generation_wh) / 1000.0 AS generation_kwh,
    SUM(consumption_wh - generation_wh) / 1000.0 AS net_kwh,
    MAX(demand_w) / 1000.0 AS peak_demand_kw,
    AVG(voltage_v) AS avg_voltage_v,
    AVG(power_factor) AS avg_power_factor,
    COUNT(*) AS reading_count,
    COUNT(*) FILTER (WHERE reading_quality != 0) AS estimated_count
FROM interval_readings
GROUP BY bucket, usage_point_id
WITH NO DATA;

-- Refresh policy: materialise hourly data with 30-minute lag
SELECT add_continuous_aggregate_policy('hourly_usage',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '30 minutes',
    schedule_interval => INTERVAL '1 hour'
);

-- DAILY AGGREGATE (for daily usage charts and bill calculation)
CREATE MATERIALIZED VIEW daily_usage
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    usage_point_id,
    SUM(consumption_wh) / 1000.0 AS consumption_kwh,
    SUM(generation_wh) / 1000.0 AS generation_kwh,
    SUM(consumption_wh - generation_wh) / 1000.0 AS net_kwh,
    MAX(demand_w) / 1000.0 AS peak_demand_kw,
    AVG(voltage_v) AS avg_voltage_v,
    COUNT(*) AS reading_count,
    COUNT(*) FILTER (WHERE reading_quality != 0) AS estimated_count
FROM interval_readings
GROUP BY bucket, usage_point_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('daily_usage',
    start_offset    => INTERVAL '7 days',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Compress the daily continuous aggregate after 30 days
ALTER MATERIALIZED VIEW daily_usage SET (
    timescaledb.compress = true
);
SELECT add_compression_policy('daily_usage', INTERVAL '30 days');

-- MONTHLY AGGREGATE (for bill summary, year-over-year comparison)
CREATE MATERIALIZED VIEW monthly_usage
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', time) AS bucket,
    usage_point_id,
    SUM(consumption_wh) / 1000.0 AS consumption_kwh,
    SUM(generation_wh) / 1000.0 AS generation_kwh,
    SUM(consumption_wh - generation_wh) / 1000.0 AS net_kwh,
    MAX(demand_w) / 1000.0 AS peak_demand_kw,
    AVG(CASE WHEN demand_w > 0 THEN demand_w END) / 1000.0 AS avg_demand_kw,
    COUNT(*) AS reading_count
FROM interval_readings
GROUP BY bucket, usage_point_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('monthly_usage',
    start_offset    => INTERVAL '3 months',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- =============================================================
-- TOU PERIOD USAGE — continuous aggregate with rate period classification
-- Requires a helper function to classify timestamps into TOU periods
-- =============================================================

CREATE OR REPLACE FUNCTION classify_tou_period(ts TIMESTAMPTZ, rate_plan_code VARCHAR DEFAULT 'TOU-D-PRIME')
RETURNS VARCHAR AS $$
DECLARE
    hour_of_day INTEGER := EXTRACT(HOUR FROM ts AT TIME ZONE 'America/Los_Angeles');
    month_num INTEGER := EXTRACT(MONTH FROM ts);
    day_of_week INTEGER := EXTRACT(DOW FROM ts);  -- 0=Sunday
    is_summer BOOLEAN := month_num BETWEEN 6 AND 9;
    is_weekday BOOLEAN := day_of_week BETWEEN 1 AND 5;
BEGIN
    -- Default TOU-D-PRIME schedule (configurable per rate plan)
    IF is_summer THEN
        IF is_weekday AND hour_of_day BETWEEN 16 AND 20 THEN
            RETURN 'on_peak';
        ELSE
            RETURN 'off_peak';
        END IF;
    ELSE
        IF is_weekday AND hour_of_day BETWEEN 16 AND 20 THEN
            RETURN 'on_peak';
        ELSIF hour_of_day BETWEEN 0 AND 5 THEN
            RETURN 'super_off_peak';
        ELSE
            RETURN 'off_peak';
        END IF;
    END IF;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Per-TOU-period daily aggregation (uses the helper function)
CREATE MATERIALIZED VIEW daily_tou_usage
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    usage_point_id,
    -- Cannot call a function in a continuous aggregate, so we use CASE logic inline
    -- Summer on-peak: weekday 16:00-20:59, June-September
    SUM(CASE
        WHEN EXTRACT(MONTH FROM time) BETWEEN 6 AND 9
             AND EXTRACT(DOW FROM time) BETWEEN 1 AND 5
             AND EXTRACT(HOUR FROM time) BETWEEN 16 AND 20
        THEN consumption_wh ELSE 0
    END) / 1000.0 AS on_peak_kwh,

    -- Winter super-off-peak: 00:00-05:59, October-May
    SUM(CASE
        WHEN EXTRACT(MONTH FROM time) NOT BETWEEN 6 AND 9
             AND EXTRACT(HOUR FROM time) BETWEEN 0 AND 5
        THEN consumption_wh ELSE 0
    END) / 1000.0 AS super_off_peak_kwh,

    -- Off-peak: everything else
    SUM(CASE
        WHEN NOT (
            (EXTRACT(MONTH FROM time) BETWEEN 6 AND 9
             AND EXTRACT(DOW FROM time) BETWEEN 1 AND 5
             AND EXTRACT(HOUR FROM time) BETWEEN 16 AND 20)
            OR
            (EXTRACT(MONTH FROM time) NOT BETWEEN 6 AND 9
             AND EXTRACT(HOUR FROM time) BETWEEN 0 AND 5)
        )
        AND NOT (
            EXTRACT(DOW FROM time) BETWEEN 1 AND 5
            AND EXTRACT(HOUR FROM time) BETWEEN 16 AND 20
        )
        THEN consumption_wh ELSE 0
    END) / 1000.0 AS off_peak_kwh,

    SUM(consumption_wh) / 1000.0 AS total_kwh
FROM interval_readings
GROUP BY bucket, usage_point_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('daily_tou_usage',
    start_offset    => INTERVAL '7 days',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

### 3. Time-Series for NEM Import/Export Tracking

```sql
-- =============================================================
-- NEM INTERVAL DATA — separate hypertable for solar generation tracking
-- Tracks import vs export at 15-minute resolution for NEM settlement
-- =============================================================

CREATE TABLE nem_interval_data (
    time                TIMESTAMPTZ NOT NULL,
    nem_account_id      UUID NOT NULL,
    usage_point_id      UUID NOT NULL,
    import_wh           NUMERIC(12,2) NOT NULL,                 -- consumed from grid
    export_wh           NUMERIC(12,2) NOT NULL,                 -- exported to grid
    generation_wh       NUMERIC(12,2),                          -- total solar generation (if separate meter)
    self_consumed_wh    NUMERIC(12,2),                          -- generation - export
    tou_period          VARCHAR(20),                            -- on_peak, off_peak, super_off_peak
    export_credit_rate  NUMERIC(8,5),                           -- $/kWh credit rate applicable at this interval
    export_credit       NUMERIC(8,4)                            -- monetary credit for this interval
);

SELECT create_hypertable('nem_interval_data', by_range('time', INTERVAL '1 day'));

ALTER TABLE nem_interval_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'nem_account_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('nem_interval_data', INTERVAL '30 days');

CREATE INDEX idx_nem_interval_account ON nem_interval_data (nem_account_id, time DESC);

-- Monthly NEM settlement continuous aggregate
CREATE MATERIALIZED VIEW monthly_nem_settlement
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', time) AS bucket,
    nem_account_id,
    SUM(import_wh) / 1000.0 AS total_import_kwh,
    SUM(export_wh) / 1000.0 AS total_export_kwh,
    SUM(import_wh - export_wh) / 1000.0 AS net_kwh,
    SUM(CASE WHEN tou_period = 'on_peak' THEN export_wh ELSE 0 END) / 1000.0 AS on_peak_export_kwh,
    SUM(CASE WHEN tou_period = 'off_peak' THEN export_wh ELSE 0 END) / 1000.0 AS off_peak_export_kwh,
    SUM(CASE WHEN tou_period = 'super_off_peak' THEN export_wh ELSE 0 END) / 1000.0 AS super_off_peak_export_kwh,
    SUM(export_credit) AS total_export_credit,
    SUM(self_consumed_wh) / 1000.0 AS total_self_consumed_kwh
FROM nem_interval_data
GROUP BY bucket, nem_account_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('monthly_nem_settlement',
    start_offset    => INTERVAL '3 months',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);
```

### 4. Time-Series for EV Charging

```sql
-- =============================================================
-- EV CHARGING TELEMETRY — real-time charging session data
-- =============================================================

CREATE TABLE ev_charging_telemetry (
    time                TIMESTAMPTZ NOT NULL,
    charger_id          UUID NOT NULL,
    session_id          UUID NOT NULL,
    power_w             NUMERIC(10,2),                          -- instantaneous power draw
    energy_wh_cumulative NUMERIC(12,2),                         -- cumulative energy this session
    voltage_v           NUMERIC(7,2),
    current_a           NUMERIC(7,2),
    soc_pct             NUMERIC(5,2),                           -- state of charge (if V2G/OCPP reports it)
    grid_rate_period    VARCHAR(20)                              -- TOU period at this moment
);

SELECT create_hypertable('ev_charging_telemetry', by_range('time', INTERVAL '1 day'));

ALTER TABLE ev_charging_telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'charger_id, session_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ev_charging_telemetry', INTERVAL '7 days');

CREATE INDEX idx_ev_telemetry_charger ON ev_charging_telemetry (charger_id, time DESC);
CREATE INDEX idx_ev_telemetry_session ON ev_charging_telemetry (session_id, time DESC);

-- Daily EV charging summary
CREATE MATERIALIZED VIEW daily_ev_charging
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    charger_id,
    session_id,
    MAX(energy_wh_cumulative) / 1000.0 AS energy_kwh,
    MAX(power_w) / 1000.0 AS peak_power_kw,
    AVG(power_w) / 1000.0 AS avg_power_kw,
    MAX(soc_pct) AS final_soc_pct,
    -- TOU period breakdown
    SUM(CASE WHEN grid_rate_period = 'on_peak' THEN 1 ELSE 0 END) AS on_peak_intervals,
    SUM(CASE WHEN grid_rate_period = 'off_peak' THEN 1 ELSE 0 END) AS off_peak_intervals,
    COUNT(*) AS telemetry_count
FROM ev_charging_telemetry
GROUP BY bucket, charger_id, session_id
WITH NO DATA;

SELECT add_continuous_aggregate_policy('daily_ev_charging',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);
```

### 5. Time-Series for Voltage and Power Quality

```sql
-- =============================================================
-- POWER QUALITY METRICS — separate hypertable for voltage monitoring
-- Used for voltage complaint investigation and DER impact analysis
-- =============================================================

CREATE TABLE power_quality_readings (
    time                TIMESTAMPTZ NOT NULL,
    usage_point_id      UUID NOT NULL,
    voltage_a           NUMERIC(7,2),                           -- phase A voltage
    voltage_b           NUMERIC(7,2),                           -- phase B voltage (if applicable)
    voltage_c           NUMERIC(7,2),                           -- phase C voltage (if applicable)
    frequency_hz        NUMERIC(6,3),
    thd_pct             NUMERIC(5,2),                           -- total harmonic distortion
    power_factor        NUMERIC(5,4),
    reactive_power_var  NUMERIC(10,2),
    sag_count           INTEGER DEFAULT 0,                      -- voltage sag events
    swell_count         INTEGER DEFAULT 0                       -- voltage swell events
);

SELECT create_hypertable('power_quality_readings', by_range('time', INTERVAL '1 day'));

ALTER TABLE power_quality_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'usage_point_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('power_quality_readings', INTERVAL '7 days');
SELECT add_retention_policy('power_quality_readings', INTERVAL '2 years');

CREATE INDEX idx_pq_readings ON power_quality_readings (usage_point_id, time DESC);
```

### 6. PostGIS-Optimised Outage Tables

```sql
-- =============================================================
-- OUTAGE EVENTS — PostGIS-enriched for spatial queries
-- =============================================================

CREATE TABLE outage_events (
    outage_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_oms_id     VARCHAR(100) UNIQUE,
    outage_type         VARCHAR(30) NOT NULL,
    cause               VARCHAR(50),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    severity            VARCHAR(20),
    customers_affected  INTEGER,
    start_time          TIMESTAMPTZ NOT NULL,
    estimated_restoration TIMESTAMPTZ,
    actual_restoration  TIMESTAMPTZ,
    crew_status         VARCHAR(100),

    -- PostGIS geometries for spatial operations
    affected_area       GEOMETRY(MultiPolygon, 4326),           -- polygon of affected area
    fault_location      GEOMETRY(Point, 4326),                  -- fault point (if known)
    center_point        GEOMETRY(Point, 4326),                  -- centroid for clustering

    -- OMS-specific metadata
    oms_metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    status_timeline     JSONB NOT NULL DEFAULT '[]'::jsonb,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Spatial indexes for outage map queries
CREATE INDEX idx_outage_affected_area ON outage_events USING GIST (affected_area);
CREATE INDEX idx_outage_fault_location ON outage_events USING GIST (fault_location);
CREATE INDEX idx_outage_center ON outage_events USING GIST (center_point);
CREATE INDEX idx_outage_status ON outage_events (status) WHERE status NOT IN ('restored', 'cancelled');
CREATE INDEX idx_outage_time ON outage_events (start_time DESC);

-- =============================================================
-- AFFECTED LOCATIONS — pre-computed join between outages and service locations
-- Enables fast "is this customer affected?" lookups
-- =============================================================

CREATE TABLE outage_affected_locations (
    outage_id           UUID NOT NULL REFERENCES outage_events(outage_id),
    location_id         UUID NOT NULL REFERENCES service_locations(location_id),
    customer_id         UUID NOT NULL,                          -- denormalised for fast lookup
    notified            BOOLEAN DEFAULT FALSE,
    notified_at         TIMESTAMPTZ,
    PRIMARY KEY (outage_id, location_id)
);

CREATE INDEX idx_affected_locations_customer ON outage_affected_locations (customer_id);
CREATE INDEX idx_affected_locations_outage ON outage_affected_locations (outage_id) WHERE NOT notified;

-- =============================================================
-- SPATIAL UTILITY FUNCTIONS
-- =============================================================

-- Find all customers affected by an outage polygon
CREATE OR REPLACE FUNCTION find_affected_customers(outage_polygon GEOMETRY)
RETURNS TABLE (
    location_id UUID,
    customer_id UUID,
    address TEXT,
    distance_m NUMERIC
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        sl.location_id,
        sl.customer_id,
        sl.address_line1 || ', ' || sl.city AS address,
        ST_Distance(sl.geom::geography, ST_Centroid(outage_polygon)::geography)::NUMERIC AS distance_m
    FROM service_locations sl
    WHERE sl.status = 'active'
      AND ST_Within(sl.geom, outage_polygon)
    ORDER BY distance_m;
END;
$$ LANGUAGE plpgsql;

-- Find nearest outage to a customer location
CREATE OR REPLACE FUNCTION find_nearest_outage(customer_point GEOMETRY, max_distance_m NUMERIC DEFAULT 50000)
RETURNS TABLE (
    outage_id UUID,
    external_oms_id VARCHAR,
    status VARCHAR,
    cause VARCHAR,
    distance_m NUMERIC,
    estimated_restoration TIMESTAMPTZ
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        oe.outage_id,
        oe.external_oms_id,
        oe.status,
        oe.cause,
        ST_Distance(oe.center_point::geography, customer_point::geography)::NUMERIC AS distance_m,
        oe.estimated_restoration
    FROM outage_events oe
    WHERE oe.status NOT IN ('restored', 'cancelled')
      AND ST_DWithin(oe.center_point::geography, customer_point::geography, max_distance_m)
    ORDER BY distance_m
    LIMIT 5;
END;
$$ LANGUAGE plpgsql;

-- Generate outage map tile data (for map clustering at different zoom levels)
CREATE OR REPLACE FUNCTION outage_map_clusters(
    bbox GEOMETRY,
    zoom_level INTEGER
)
RETURNS TABLE (
    cluster_center GEOMETRY,
    total_outages INTEGER,
    total_customers_affected INTEGER,
    most_severe VARCHAR,
    earliest_start TIMESTAMPTZ,
    latest_eta TIMESTAMPTZ
) AS $$
DECLARE
    grid_size NUMERIC;
BEGIN
    -- Grid size decreases with zoom level for progressive clustering
    grid_size := 360.0 / POWER(2, zoom_level);

    RETURN QUERY
    SELECT
        ST_Centroid(ST_Collect(oe.center_point)) AS cluster_center,
        COUNT(*)::INTEGER AS total_outages,
        SUM(oe.customers_affected)::INTEGER AS total_customers_affected,
        (ARRAY_AGG(oe.severity ORDER BY
            CASE oe.severity WHEN 'major' THEN 1 WHEN 'significant' THEN 2 WHEN 'minor' THEN 3 ELSE 4 END
        ))[1] AS most_severe,
        MIN(oe.start_time) AS earliest_start,
        MAX(oe.estimated_restoration) AS latest_eta
    FROM outage_events oe
    WHERE oe.status NOT IN ('restored', 'cancelled')
      AND oe.center_point && bbox
    GROUP BY
        ST_SnapToGrid(oe.center_point, grid_size);
END;
$$ LANGUAGE plpgsql;
```

### 7. Outage History Time-Series

```sql
-- =============================================================
-- OUTAGE HISTORY — time-series for reliability metrics and AI training
-- =============================================================

CREATE TABLE outage_history (
    time                TIMESTAMPTZ NOT NULL,                   -- outage start time
    outage_id           UUID NOT NULL,
    location_id         UUID,
    outage_type         VARCHAR(30),
    cause               VARCHAR(50),
    customers_affected  INTEGER,
    duration_minutes    NUMERIC(10,2),
    fault_location      GEOMETRY(Point, 4326),
    weather_conditions  JSONB,                                  -- wind, rain, temperature at time of outage
    equipment_type      VARCHAR(50)                             -- transformer, conductor, switch, fuse, etc.
);

SELECT create_hypertable('outage_history', by_range('time', INTERVAL '1 month'));

ALTER TABLE outage_history SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'cause',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('outage_history', INTERVAL '90 days');

CREATE INDEX idx_outage_history_location ON outage_history USING GIST (fault_location);

-- Monthly reliability metrics (SAIDI, SAIFI, CAIDI)
CREATE MATERIALIZED VIEW monthly_reliability_metrics
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', time) AS bucket,
    cause,
    COUNT(*) AS outage_count,
    SUM(customers_affected) AS total_customer_interruptions,
    AVG(duration_minutes) AS avg_duration_minutes,
    SUM(customers_affected * duration_minutes) AS total_customer_minutes
FROM outage_history
GROUP BY bucket, cause
WITH NO DATA;

SELECT add_continuous_aggregate_policy('monthly_reliability_metrics',
    start_offset    => INTERVAL '3 months',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);
```

### 8. Weather Correlation Data

```sql
-- =============================================================
-- WEATHER DATA — time-series for usage correlation and outage prediction
-- =============================================================

CREATE TABLE weather_observations (
    time                TIMESTAMPTZ NOT NULL,
    station_id          VARCHAR(20) NOT NULL,                   -- NOAA weather station ID
    latitude            NUMERIC(10,7),
    longitude           NUMERIC(10,7),
    temperature_f       NUMERIC(5,1),
    humidity_pct        NUMERIC(5,1),
    wind_speed_mph      NUMERIC(5,1),
    wind_gust_mph       NUMERIC(5,1),
    precipitation_in    NUMERIC(5,2),
    cloud_cover_pct     NUMERIC(5,1),
    uv_index            NUMERIC(4,1),
    solar_irradiance_wm2 NUMERIC(7,2),                          -- for solar generation estimation
    cooling_degree_hours NUMERIC(6,2),
    heating_degree_hours NUMERIC(6,2)
);

SELECT create_hypertable('weather_observations', by_range('time', INTERVAL '1 day'));

ALTER TABLE weather_observations SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'station_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('weather_observations', INTERVAL '7 days');
SELECT add_retention_policy('weather_observations', INTERVAL '5 years');

CREATE INDEX idx_weather_station ON weather_observations (station_id, time DESC);
```

### 9. AI Feature Store

```sql
-- =============================================================
-- AI FEATURE STORE — pre-computed features for ML models
-- Updated by continuous aggregates and batch jobs
-- =============================================================

CREATE TABLE ai_customer_features (
    time                TIMESTAMPTZ NOT NULL,                   -- feature computation date
    customer_id         UUID NOT NULL,
    location_id         UUID NOT NULL,

    -- Usage features (rolling windows)
    avg_daily_kwh_7d    NUMERIC(10,3),
    avg_daily_kwh_30d   NUMERIC(10,3),
    avg_daily_kwh_90d   NUMERIC(10,3),
    usage_trend_30d     NUMERIC(8,4),                           -- slope of daily usage over 30 days
    peak_to_avg_ratio   NUMERIC(6,3),
    on_peak_pct         NUMERIC(5,2),
    weekend_vs_weekday_ratio NUMERIC(5,3),

    -- Billing features
    avg_monthly_bill    NUMERIC(10,2),
    bill_volatility     NUMERIC(8,4),                           -- std dev of monthly bills
    payment_timeliness_score NUMERIC(5,2),                      -- 0-100, based on payment history

    -- NEM features (NULL if no NEM account)
    self_consumption_ratio NUMERIC(5,3),
    avg_daily_export_kwh NUMERIC(10,3),
    estimated_system_degradation_pct NUMERIC(5,2),

    -- EV features (NULL if no EV charger)
    avg_daily_ev_kwh    NUMERIC(8,3),
    off_peak_charging_pct NUMERIC(5,2),

    -- Engagement features
    portal_logins_30d   INTEGER,
    outage_reports_12m  INTEGER,
    programme_enrolments INTEGER,

    -- Derived scores
    churn_risk_score    NUMERIC(5,4),
    rate_switch_benefit NUMERIC(10,2),
    hardship_risk_score NUMERIC(5,4)
);

SELECT create_hypertable('ai_customer_features', by_range('time', INTERVAL '1 day'));

ALTER TABLE ai_customer_features SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'customer_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ai_customer_features', INTERVAL '7 days');

CREATE INDEX idx_ai_features ON ai_customer_features (customer_id, time DESC);
```

### 10. Billing, Payments, Programmes, etc.

```sql
-- The remaining tables (bills, payments, rate_plans, programmes, etc.)
-- use the same design as Suggestions 1 and 3.
-- They are standard relational tables, not time-series, and do not benefit
-- from TimescaleDB hypertables.

-- See Suggestion 1 for:
--   bills, bill_line_items, payments, autopay_enrolments, payment_plans
--   rate_plans, rate_schedule_periods, customer_rate_enrolments
--   nem_accounts, nem_monthly_settlements, nem_annual_trueups
--   programmes, programme_enrolments, dr_events, dr_event_participation
--   ev_chargers, ev_charging_schedules, ev_charging_sessions
--   service_orders, green_button_authorisations, ai_insights
--   customer_outage_reports, outage_notifications

-- These tables coexist in the same PostgreSQL database alongside the
-- TimescaleDB hypertables. Standard foreign keys and JOINs work across
-- regular tables and hypertables.
```

---

## Example Queries

### Usage Dashboard (Daily Chart with Weather Overlay)

```sql
-- Daily usage for the last 30 days with weather data
SELECT
    du.bucket::date AS usage_date,
    du.consumption_kwh,
    du.generation_kwh,
    du.net_kwh,
    du.peak_demand_kw,
    wo.temperature_f AS avg_temp
FROM daily_usage
    JOIN LATERAL (
        SELECT AVG(temperature_f) AS temperature_f
        FROM weather_observations
        WHERE station_id = 'KSFO'
          AND time >= du.bucket
          AND time < du.bucket + INTERVAL '1 day'
    ) wo ON TRUE
WHERE du.usage_point_id = 'abc-123'
  AND du.bucket >= NOW() - INTERVAL '30 days'
ORDER BY du.bucket;
```

### NEM Monthly Settlement with TOU Export Breakdown

```sql
-- Monthly NEM settlement for the current annual cycle
SELECT
    bucket::date AS settlement_month,
    total_import_kwh,
    total_export_kwh,
    net_kwh,
    on_peak_export_kwh,
    off_peak_export_kwh,
    super_off_peak_export_kwh,
    total_export_credit
FROM monthly_nem_settlement
WHERE nem_account_id = 'nem-456'
  AND bucket >= '2025-04-01'
ORDER BY bucket;
```

### Outage Map Query (Active Outages in Viewport)

```sql
-- Find all active outages within the user's map viewport
SELECT
    outage_id,
    external_oms_id,
    outage_type,
    cause,
    status,
    customers_affected,
    start_time,
    estimated_restoration,
    crew_status,
    ST_AsGeoJSON(affected_area) AS affected_area_geojson,
    ST_X(center_point) AS center_lng,
    ST_Y(center_point) AS center_lat
FROM outage_events
WHERE status NOT IN ('restored', 'cancelled')
  AND center_point && ST_MakeEnvelope(-122.5, 37.6, -122.0, 37.9, 4326)
ORDER BY customers_affected DESC;
```

### EV Charging Efficiency Analysis

```sql
-- Weekly EV charging summary with off-peak percentage
SELECT
    time_bucket('1 week', time) AS week,
    SUM(CASE WHEN grid_rate_period IN ('off_peak', 'super_off_peak') THEN 1 ELSE 0 END)::NUMERIC
        / NULLIF(COUNT(*), 0) * 100 AS off_peak_pct,
    SUM(power_w) / 1000.0 / COUNT(*) AS avg_power_kw,
    MAX(energy_wh_cumulative) / 1000.0 AS total_kwh
FROM ev_charging_telemetry
WHERE charger_id = 'charger-789'
  AND time >= NOW() - INTERVAL '12 weeks'
GROUP BY week
ORDER BY week;
```

---

## Pros and Cons

### Pros

1. **Purpose-built for the highest-volume workload.** TimescaleDB's automatic chunk management, compression (10-20x), and continuous aggregates are designed specifically for the interval-reading workload that dominates the portal's data volume. No manual partition management, no custom aggregation jobs, no compression scripts.

2. **Continuous aggregates eliminate batch ETL.** The hourly, daily, and monthly usage summaries are automatically maintained by TimescaleDB as data arrives. No cron jobs, no ETL pipelines, no stale summary tables. The dashboard always shows up-to-date data without application-layer aggregation code.

3. **Compression keeps costs manageable.** At 10-20x compression, 5 years of interval data for 500,000 meters fits in ~1 TB instead of ~10 TB. Compressed data remains fully queryable with standard SQL — no decompress-before-query step.

4. **Single SQL dialect.** Unlike architectures that pair PostgreSQL with InfluxDB or Prometheus, everything here speaks PostgreSQL SQL. Application developers write the same SQL for billing queries and time-series queries. JOINs across hypertables and regular tables work natively.

5. **PostGIS handles the storm-day scalability challenge.** The outage map is the portal's most demanding real-time workload. PostGIS spatial indexes, combined with the clustering function and Redis caching, can serve thousands of concurrent outage-map queries during major events.

6. **AI/ML-ready feature store.** The ai_customer_features hypertable provides pre-computed, time-indexed features for ML model training and inference. Daily feature snapshots enable temporal ML (e.g., "predict tomorrow's bill based on features computed today") without complex point-in-time joins.

7. **Retention policies automate data lifecycle.** TimescaleDB retention policies automatically drop data older than the configured window, eliminating manual partition management and storage growth concerns.

8. **Single operational footprint.** One PostgreSQL instance (or cluster) to operate, monitor, backup, and upgrade. No separate time-series database to maintain.

### Cons

1. **Extension dependency.** TimescaleDB is an extension, not core PostgreSQL. It requires specific PostgreSQL versions, adds upgrade complexity (must coordinate PostgreSQL and TimescaleDB versions), and is not available on all managed PostgreSQL services (though Timescale Cloud, Aiven, and self-hosted all support it).

2. **Continuous aggregate limitations.** TimescaleDB continuous aggregates have restrictions on supported SQL (no subqueries, limited function support, no JOINs in the aggregate definition). Complex TOU period classification requires inline CASE statements rather than function calls, as shown in the daily_tou_usage example.

3. **Licensing considerations.** TimescaleDB Community Edition is open-source (Apache 2.0), but some advanced features (multi-node, continuous aggregate compression, background jobs) require the Timescale License (free for self-hosted, commercial for managed). The portal must evaluate which features are needed.

4. **Not all tables benefit.** The customer, billing, payment, and programme tables are standard OLTP workloads that gain nothing from TimescaleDB. Only the time-series tables (interval_readings, nem_interval_data, ev_charging_telemetry, etc.) benefit. This means the database has two "personalities" — developers must understand when to use hypertable features vs. standard PostgreSQL.

5. **Space partitioning complexity.** The `add_dimension(by_hash('usage_point_id', 16))` call creates a 2D partition grid (time x usage_point_id hash). While this improves parallel query performance, it increases chunk count and can complicate chunk management.

6. **Foreign key limitations.** TimescaleDB hypertables have restrictions on foreign keys: a hypertable can reference a regular table's primary key, but a regular table cannot have a foreign key pointing into a hypertable (because the hypertable's primary key must include the time column). This means interval_readings cannot be referenced by other tables.

7. **Backup and restore time.** While compression reduces storage, the total data volume for a large utility is still substantial. Point-in-time recovery and logical backup/restore times must be planned for.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with TimescaleDB 2.15+ and PostGIS 3.4+ |
| **Hosting** | Timescale Cloud (managed) or self-hosted with Patroni for HA |
| **Connection pooling** | PgBouncer in transaction mode |
| **Caching** | Redis for outage map tile cache, session data, and dashboard summaries |
| **Ingestion** | Batch COPY from MDMS for interval readings (100K+ rows/sec); Kafka for real-time EV telemetry |
| **Map rendering** | Mapbox GL JS or MapLibre GL JS with vector tiles pre-rendered from PostGIS |
| **Monitoring** | TimescaleDB informational views (timescaledb_information.chunks, .compression_settings); Prometheus + Grafana |
| **Backup** | pgBackRest with parallel backup and S3 archival |

---

## Migration and Scaling Considerations

### Migrating from Vanilla PostgreSQL

1. **Install TimescaleDB extension.** `CREATE EXTENSION IF NOT EXISTS timescaledb;` on an existing PostgreSQL instance. No data loss; existing tables are unaffected.
2. **Convert interval_readings to hypertable.** If the existing table is partitioned, it can be migrated chunk-by-chunk. If unpartitioned, TimescaleDB provides a `create_hypertable` migration function that handles existing data.
3. **Create continuous aggregates.** Define the aggregate views and backfill historical data with `CALL refresh_continuous_aggregate('daily_usage', NULL, NOW());`.
4. **Enable compression.** Configure compression on each hypertable and run initial compression on historical chunks.
5. **Set retention policies.** Configure automatic data retention for each hypertable.

### Scaling Strategy

| Scale | Meters | Approach |
|-------|--------|----------|
| Small co-op | <50K | Single PostgreSQL + TimescaleDB instance, 4-8 cores, 32GB RAM |
| Mid-size municipal | 50K-500K | HA pair (Patroni), 16-32 cores, 128GB RAM, read replicas for dashboard |
| Large IOU | 500K-5M | TimescaleDB multi-node or Timescale Cloud with read scaling |
| Very large IOU | >5M | TimescaleDB multi-node with dedicated ingestion nodes, caching tier, CDN for map tiles |

### Compression Impact

| Table | Uncompressed Row Size | Compression Ratio | Compressed Row Size |
|-------|----------------------|-------------------|-------------------|
| interval_readings | ~120 bytes | ~12x | ~10 bytes |
| nem_interval_data | ~100 bytes | ~10x | ~10 bytes |
| ev_charging_telemetry | ~90 bytes | ~15x | ~6 bytes |
| weather_observations | ~100 bytes | ~10x | ~10 bytes |
| power_quality_readings | ~80 bytes | ~12x | ~7 bytes |

### Storm-Day Outage Map Architecture

During a major storm, the outage map must handle 10-100x normal traffic:

```
                          ┌──────────────┐
                          │     CDN      │ ← static map tiles, JS, CSS
                          └──────┬───────┘
                                 │
                          ┌──────▼───────┐
                          │ Load Balancer │
                          └──┬───────┬───┘
                             │       │
                      ┌──────▼──┐ ┌──▼──────┐
                      │ App     │ │ App     │ ← stateless Node.js/Python
                      │ Server 1│ │ Server 2│   (auto-scale group)
                      └──┬──────┘ └──────┬──┘
                         │               │
                      ┌──▼───────────────▼──┐
                      │      Redis Cache     │ ← outage map data cached for 30-60 sec
                      │  (ElastiCache/Dragonfly) │
                      └──────────┬───────────┘
                                 │ cache miss
                      ┌──────────▼───────────┐
                      │  PostgreSQL + PostGIS  │
                      │  (read replica for     │
                      │   map queries)         │
                      └────────────────────────┘
```

The Redis cache stores the GeoJSON output of outage queries with a 30-60 second TTL. During normal operations, cache hit rate is >95%. During major storms, the cache prevents read replicas from being overwhelmed while still providing near-real-time outage data.

---

## Summary

The PostgreSQL + TimescaleDB + PostGIS model is the domain-optimal architecture for an electric utility customer portal. It addresses the two workloads that generic databases handle poorly: high-volume time-series meter data and high-concurrency geospatial outage mapping. By using PostgreSQL extensions rather than separate database products, it maintains a single SQL dialect, single connection pool, single backup strategy, and single operations team. The trade-off is extension dependency and the need for developers to understand TimescaleDB-specific concepts (hypertables, continuous aggregates, compression). For utilities with more than 50,000 meters, the performance and storage benefits of TimescaleDB over vanilla PostgreSQL partitioning are substantial enough to justify this complexity.
