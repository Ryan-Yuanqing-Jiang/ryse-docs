# Challenge 3: Live Operational Dashboard & Exception Alerting — System Architecture

---

## Tech Stack

### Frontend

| Layer | Technology | Rationale |
|---|---|---|
| Framework | React 18 + TypeScript 5 | Component model suits the real-time dashboard pattern; TypeScript enforces data contract rigour across WebSocket payloads |
| Map rendering | Mapbox GL JS 3.x | GPU-accelerated WebGL rendering handles 35 simultaneous moving markers without jank; custom layer support for route overlays and risk heat maps |
| Real-time client | Socket.io Client 4.x | Matches backend Socket.io server; automatic reconnection, room-based subscriptions, and event namespacing out of the box |
| Server-state cache | React Query (TanStack Query 5) | Stale-while-revalidate for non-streaming data (route summaries, driver profiles, client SLA configs); separates REST polling from WebSocket stream state |
| UI components | shadcn/ui + Tailwind CSS 3 | Headless component primitives; Tailwind utility classes scale well for a dense operational dashboard with compact data tables |
| State management | Zustand 4 | Lightweight global store for WebSocket event dispatch; avoids Redux boilerplate for this scale |
| Charts | Recharts 2 | SLA risk gauges, route completion trend lines; renders well in dashboards without heavy D3 coupling |
| Build tooling | Vite 5 + pnpm | Fast HMR for frontend development; pnpm workspace for monorepo |

### Backend Services

| Service | Technology | Rationale |
|---|---|---|
| API Gateway | Python FastAPI 0.111 | Async-first, native Pydantic validation, auto-generated OpenAPI docs; fast enough for the <200ms inference budget |
| WebSocket server | Socket.io (Python-socketio 5.x, ASGI mode) | Bidirectional push from Kafka consumer to connected dispatcher clients; namespace-per-tenant for multi-operator isolation |
| ML Inference | FastAPI micro-service (separate deployment) | Isolated scaling; GBDT inference is CPU-bound, scaled independently from API gateway |
| Async task queue | Celery 5.x + Redis broker | Autonomous action execution with countdown timers; retry policies; task result persistence |
| Background consumers | confluent-kafka-python 2.x | Kafka consumer groups for stream processing; one consumer group per downstream service |

### Streaming

| Component | Technology | Notes |
|---|---|---|
| Event bus | Apache Kafka on AWS MSK | Managed Kafka; 3-broker cluster, multi-AZ. Topics: `driver-locations`, `stop-events`, `alert-triggers`, `cascade-events`, `customer-contact-queue` |
| Stream processing | AWS Lambda + Kinesis (lightweight) OR Kafka Streams (JVM sidecar) | For operators at this scale, Lambda + Kinesis is operationally simpler. Kafka Streams is preferred if latency budget requires sub-100ms ETA update cycles |
| Schema registry | AWS Glue Schema Registry | Avro schemas for all Kafka topics; enforces producer/consumer contract across services |

### Storage

| Store | Technology | Purpose |
|---|---|---|
| Primary relational DB | PostgreSQL 16 on AWS RDS | Routes, stops, drivers, clients, SLA configs, alerts, interventions, audit logs |
| Time-series GPS store | TimescaleDB (PostgreSQL extension) on RDS or self-managed EC2 | `driver_positions` hypertable; continuous aggregates for 1-min and 5-min position rollups; retention policy: raw 30 days, aggregated 1 year |
| Cache | Redis 7 on AWS ElastiCache | Current driver positions (latest GPS state), active alert state, session data, Celery broker, ETA prediction cache (5s TTL per driver) |
| Object storage | AWS S3 | Route plan files, audit log archives, ML model artifacts, driver app APKs |
| Search | (Optional) OpenSearch | Alert history search, driver activity audit search for compliance queries |

### ML Inference Service

- **Model type:** Gradient Boosted Decision Tree (LightGBM preferred over XGBoost for inference speed; ~40% faster prediction at equivalent accuracy)
- **Framework:** LightGBM 4.x, served via FastAPI, loaded once at startup
- **Deployment:** Separate AWS ECS Fargate task; 2 vCPU / 4 GB RAM per task; scales to 3 tasks under load
- **Inference latency budget:** <200ms p99 (end-to-end from Kafka event receipt to prediction stored in Redis)

### Queue and Async

- **Celery 5.x** with Redis as both broker and result backend
- Workers: 3 Celery worker containers on ECS Fargate; 8 concurrent tasks per worker
- Beat scheduler: Celery Beat for periodic tasks (hourly SLA summary, daily model retraining trigger)
- Priority queues: `high` (cascade orchestration, auto-execute timers), `medium` (proactive customer comms), `low` (reporting, analytics writes)

### Customer Communications

- **Outbound SMS:** Twilio Programmable Messaging API
- **Inbound reply webhook:** Twilio webhook → FastAPI endpoint → Celery task for reply parsing
- **Message generation:** OpenAI GPT-4o-mini (or Anthropic Claude Haiku) via API; <1s generation time; cost ~$0.008 per message
- **Opt-out compliance:** Twilio opt-out handling (STOP keyword) + internal opt-out registry in PostgreSQL

### Infrastructure

| Component | Technology |
|---|---|
| Container orchestration | AWS ECS Fargate (serverless containers; no EC2 fleet to manage) |
| Load balancing | AWS ALB with sticky sessions for WebSocket connections |
| CDN | AWS CloudFront for frontend static assets |
| DNS + SSL | AWS Route 53 + ACM certificates |
| Secrets | AWS Secrets Manager (DB passwords, API keys, Twilio auth) |
| CI/CD | GitHub Actions → ECR image push → ECS rolling deployment |
| Monitoring | Datadog APM + infrastructure metrics + log management |
| Alerting | PagerDuty for ops-critical alerts (Kafka consumer lag, inference latency spikes, DB connection pool exhaustion) |
| IaC | Terraform 1.8 (AWS provider) + Terragrunt for environment promotion |

---

## AI Architecture

### Full Pipeline: GPS Ping to Dispatcher Dashboard

```
Driver App (5s GPS ping)
    │
    ▼ REST POST /api/v1/gps (FastAPI API Gateway)
    │
    ├── Write to TimescaleDB driver_positions (async, non-blocking)
    ├── Write to Redis: driver:{driver_id}:current_position (TTL 30s)
    │
    ▼ Produce to Kafka topic: driver-locations
    │
    ▼ GPS Stream Consumer (confluent-kafka-python consumer group: eta-recalc)
    │   Consumes: driver-locations
    │   For each GPS event:
    │     1. Fetch driver's active route from Redis/PostgreSQL
    │     2. Fetch current traffic segment data (Google Maps Routes API, cached 3min)
    │     3. Assemble feature vector for GBDT inference
    │
    ▼ ETA Prediction Service (FastAPI, separate service)
    │   POST /internal/predict-eta
    │   Input: feature vector (position, speed, remaining stops, traffic, time-of-day, historical stop durations)
    │   Output: predicted_arrival_time, confidence_interval_p10, confidence_interval_p90
    │   Write result to: Redis eta:{stop_id}:prediction (TTL 30s)
    │                     PostgreSQL eta_predictions table (for model audit)
    │
    ▼ SLA Breach Evaluator (runs after each ETA update)
    │   For each stop with updated ETA:
    │     1. Fetch stop's committed delivery window from PostgreSQL
    │     2. Calculate breach probability: P(predicted_arrival > window_end)
    │       (Uses confidence interval distribution from GBDT output)
    │     3. Calculate Economic Risk Score: breach_probability × penalty_amount
    │     4. Write to Redis sla:{stop_id}:risk (TTL 30s)
    │     5. If Economic Risk Score crosses threshold → produce to Kafka: alert-triggers
    │
    ▼ Alert Decision Engine (consumer group: alert-processor)
    │   Consumes: alert-triggers
    │   For each trigger:
    │     1. Deduplicate (has an open alert for this stop already? update, don't duplicate)
    │     2. Apply alert taxonomy (RouteDelay, SLAAtRisk, DriverStopped, CascadeRisk)
    │     3. Generate intervention options (ranked by Economic Risk Score reduction)
    │     4. Write AlertEvent to PostgreSQL alerts table
    │     5. Push to dispatcher via Socket.io: event alert:new or alert:updated
    │     6. If unacknowledged after escalation_timeout → send SMS via Twilio
    │
    ▼ Dispatcher Dashboard (React, Socket.io client)
        Receives: alert:new, alert:updated, driver:position, route:progress
        Renders: AlertPanel (ranked alerts), FleetMap (live positions), RouteProgressList
```

### Autonomous Action Pipeline

```
Alert Decision Engine: alert created, escalation timer started
    │
    ▼ Celery Task: schedule_escalation(alert_id, timeout_seconds=300)
    │   Countdown task queued in Celery Beat with eta = now + timeout
    │
    ▼ Dispatcher action (within timeout)?
    │   YES → cancel Celery task, mark alert acknowledged, record dispatcher_action
    │   NO  → Celery task fires: auto_execute_intervention(alert_id)
    │           1. Fetch top-ranked intervention from alert record
    │           2. Execute: update stop assignments, push to driver apps, trigger customer comms
    │           3. Write InterventionAction to PostgreSQL (autonomous=True, reasoning=...)
    │           4. Push Socket.io event: alert:auto_executed to dispatcher
    │           5. Send SMS to dispatcher: "Auto-executed: [summary]"
    │           6. Write to Audit Log (immutable, S3 + PostgreSQL)
```

### Component Specifications

#### GPS Ingest Endpoint

- **Inputs:** `driver_id`, `lat`, `lng`, `speed_kmh`, `heading_degrees`, `accuracy_metres`, `timestamp_utc`
- **Outputs:** HTTP 202 Accepted; Kafka produce to `driver-locations`; Redis write
- **Latency budget:** <50ms p99 (API response; Kafka produce is async)
- **Failure mode:** If Kafka is unavailable, write to a PostgreSQL fallback queue; background worker drains to Kafka on recovery. Never block the driver app.

#### ETA Prediction Service

- **Inputs:** Feature vector (14 features — see Modular Breakdown for full list)
- **Outputs:** `predicted_arrival_epoch`, `p10_arrival_epoch`, `p90_arrival_epoch`, `confidence_score`
- **Latency budget:** <200ms p99 end-to-end (includes Redis lookup for cached traffic data)
- **Failure mode:** If prediction service is unavailable, fall back to last known ETA + 5-minute staleness flag. Do not block SLA evaluation — use stale prediction with degraded confidence score.

#### SLA Breach Evaluator

- **Inputs:** Updated ETA prediction, stop's committed window, client SLA profile
- **Outputs:** `breach_probability` (0–1 float), `economic_risk_score` (dollar amount), `alert_threshold_crossed` (boolean)
- **Latency budget:** <20ms (pure computation, no external I/O after Redis cache hits)
- **Failure mode:** If SLA profile unavailable for a stop, apply default threshold (60% probability triggers alert, $0 penalty). Log the missing profile for backfill.

#### Alert Decision Engine

- **Inputs:** Alert trigger event from Kafka `alert-triggers` topic
- **Outputs:** AlertEvent written to PostgreSQL; Socket.io push to dispatcher; optional Celery escalation task scheduled
- **Latency budget:** <500ms from Kafka consume to Socket.io push
- **Failure mode:** If Socket.io push fails (client disconnected), alert is persisted in PostgreSQL and surfaced on next dashboard load. Escalation SMS fires regardless of dashboard push status.

---

## Integration Layer

### Driver App SDK

The driver mobile application (iOS + Android, React Native) communicates with the backend via two channels:

**GPS Push (REST):**
- Endpoint: `POST /api/v1/gps`
- Frequency: every 5 seconds while route is active
- Payload: `{ driver_id, route_id, lat, lng, speed_kmh, heading, accuracy, timestamp_utc }`
- Auth: Bearer JWT issued at route start (scoped to driver_id, expires at route_end + 4 hours)
- Battery optimisation: if speed < 2 km/h for > 60 seconds, reduce to 30-second intervals; resume 5-second on movement detection

**Stop Event Push (REST):**
- Endpoint: `POST /api/v1/stop-events`
- Events: `stop_arrived`, `stop_completed`, `stop_failed`, `stop_skipped`
- Payload includes: `stop_id`, `event_type`, `timestamp_utc`, `failure_reason` (enum), `signature_collected` (boolean), `photo_url` (S3 pre-signed URL)

**Driver App Inbound (WebSocket via Socket.io):**
- The driver app subscribes to a Socket.io room keyed to `driver:{driver_id}`
- Events received: `route:updated` (new stop assignments from cascade redistribution), `route:stop_removed` (stop removed due to customer reschedule), `route:resequenced` (optimised stop order updated)

### Client Systems Integration

- **Outbound webhooks:** On SLA events (breach, at-risk, resolved), system POSTs to client-configured webhook URL
- Webhook payload: `{ event_type, delivery_id, stop_id, client_ref, predicted_arrival, committed_window, breach_probability, economic_risk_score, timestamp }`
- Webhook signing: HMAC-SHA256 with per-client shared secret (stored in Secrets Manager)
- Retry policy: 3 attempts with exponential backoff; failed webhooks logged and visible in client portal

**Client SLA Configuration API:**
- `POST /api/v1/clients/{client_id}/sla-config` — set window type, penalty amounts, alert thresholds
- `GET /api/v1/clients/{client_id}/sla-dashboard` — real-time SLA health feed for client portal embedding

### Twilio Integration

**Outbound SMS:**
- Twilio Messaging API via `twilio` Python SDK
- Messages sent from a Twilio-managed long code (or toll-free number for higher throughput)
- Rate limiting: 1 message/second per number; message queue in Celery `medium` priority

**Inbound Reply Webhook:**
- Twilio posts to: `POST /api/v1/webhooks/twilio-inbound`
- Validated via Twilio request signature verification (`twilio.request_validator`)
- Reply body parsed by LLM: classifies intent as `confirm_home`, `safe_drop`, `reschedule`, `opt_out`, `unknown`
- For `reschedule`: extracts preferred window if stated, or prompts for preference via follow-up SMS
- All reply events produce to Kafka topic: `customer-contact-queue` for async processing

### Google Maps API Integration

- **Routes API (Traffic-aware ETA):** Called per-route per 3-minute cache TTL; returns segment-level travel times
- **Geocoding API:** Used at route planning time to validate stop addresses
- **Cost management:** Routes API calls are cached in Redis with 3-minute TTL per route segment; estimated API cost ~$120–180/month for full fleet
- **Failure mode:** If Google Maps API is unavailable, fall back to historical average travel times by zone and time-of-day (stored in PostgreSQL `zone_travel_times` table). Degrade gracefully with a `traffic_data_stale` flag on ETA predictions.

### Teletrac Navman API (Optional Integration)

- **Purpose:** Vehicle health telemetry as a cascade risk signal
- **Data consumed:** Engine fault codes, battery voltage, tyre pressure alerts
- **Integration method:** Teletrac Navman REST API, polled every 5 minutes per vehicle
- **Use in cascade detection:** Vehicle fault event → produces to Kafka `cascade-events` topic → Cascade Orchestrator evaluates redistribution pre-emptively before driver reports breakdown
- **Dependency classification:** Optional enrichment; system functions fully without it

### Delivery Management System Integration

- **Route import:** `POST /api/v1/routes/import` — accepts route plan in JSON or CSV format; assigns to driver, creates StopExecution records
- **Stop completion feed:** Two-way; driver app pushes events, client portal can also mark stops complete via API (for office pickups, etc.)
- **Client portal SLA feed:** `GET /api/v1/clients/{client_id}/live-sla` — polling endpoint for clients who prefer pull over webhooks; returns current breach risk across all active stops for the client

---

## Data Models

### Core Entities and Field Definitions

#### `driver_positions` (TimescaleDB hypertable, partitioned by `recorded_at`)

```sql
CREATE TABLE driver_positions (
    id              BIGSERIAL,
    driver_id       UUID NOT NULL,
    route_id        UUID,
    lat             DOUBLE PRECISION NOT NULL,
    lng             DOUBLE PRECISION NOT NULL,
    speed_kmh       REAL,
    heading_degrees SMALLINT,
    accuracy_metres REAL,
    recorded_at     TIMESTAMPTZ NOT NULL,  -- hypertable time column
    received_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, recorded_at)
);

SELECT create_hypertable('driver_positions', 'recorded_at', chunk_time_interval => INTERVAL '1 day');

-- Continuous aggregate: 1-minute position rollups
CREATE MATERIALIZED VIEW driver_positions_1min
WITH (timescaledb.continuous) AS
SELECT
    driver_id,
    time_bucket('1 minute', recorded_at) AS bucket,
    avg(lat) AS avg_lat,
    avg(lng) AS avg_lng,
    avg(speed_kmh) AS avg_speed_kmh,
    count(*) AS ping_count
FROM driver_positions
GROUP BY driver_id, bucket;
```

#### `live_driver_positions` (Redis key pattern)

```
Key: driver:{driver_id}:position
Type: Hash
Fields: lat, lng, speed_kmh, heading, accuracy, recorded_at, route_id
TTL: 30 seconds (stale if driver app disconnected or GPS lost)
```

#### `route_executions`

```sql
CREATE TABLE route_executions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_plan_id       UUID NOT NULL REFERENCES route_plans(id),
    driver_id           UUID NOT NULL REFERENCES drivers(id),
    vehicle_id          UUID REFERENCES vehicles(id),
    operator_id         UUID NOT NULL REFERENCES operators(id),
    date                DATE NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- status: pending | active | completed | cascaded | abandoned
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    total_stops         INTEGER NOT NULL,
    completed_stops     INTEGER NOT NULL DEFAULT 0,
    failed_stops        INTEGER NOT NULL DEFAULT 0,
    completion_pct      REAL GENERATED ALWAYS AS (
                            CASE WHEN total_stops > 0
                            THEN (completed_stops::REAL / total_stops) * 100
                            ELSE 0 END
                        ) STORED,
    current_sla_risk    REAL,  -- portfolio-level breach probability for this route
    economic_risk_score NUMERIC(10,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### `stop_executions`

```sql
CREATE TABLE stop_executions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    route_execution_id      UUID NOT NULL REFERENCES route_executions(id),
    stop_sequence           SMALLINT NOT NULL,
    delivery_id             VARCHAR(100) NOT NULL,  -- client's reference
    client_id               UUID NOT NULL REFERENCES clients(id),
    recipient_name          VARCHAR(200),
    recipient_phone         VARCHAR(20),
    address_line1           VARCHAR(300) NOT NULL,
    address_lat             DOUBLE PRECISION,
    address_lng             DOUBLE PRECISION,
    committed_window_start  TIMESTAMPTZ,
    committed_window_end    TIMESTAMPTZ,
    window_type             VARCHAR(10),  -- hard | soft
    status                  VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- status: pending | en_route | arrived | completed | failed | skipped | reassigned | rescheduled
    predicted_arrival       TIMESTAMPTZ,
    predicted_arrival_p10   TIMESTAMPTZ,
    predicted_arrival_p90   TIMESTAMPTZ,
    prediction_confidence   REAL,
    breach_probability      REAL,
    economic_risk_score     NUMERIC(10,2),
    sla_penalty_amount      NUMERIC(10,2),
    arrived_at              TIMESTAMPTZ,
    completed_at            TIMESTAMPTZ,
    failure_reason          VARCHAR(50),
    failure_notes           TEXT,
    signature_collected     BOOLEAN,
    photo_s3_key            VARCHAR(500),
    customer_contacted      BOOLEAN NOT NULL DEFAULT FALSE,
    customer_response       VARCHAR(20),
    -- customer_response: confirmed_home | safe_drop | rescheduled | no_response | opted_out
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_stop_exec_route ON stop_executions(route_execution_id);
CREATE INDEX idx_stop_exec_status ON stop_executions(status) WHERE status NOT IN ('completed', 'failed');
CREATE INDEX idx_stop_exec_breach ON stop_executions(breach_probability DESC) WHERE status = 'pending';
```

#### `alert_events`

```sql
CREATE TABLE alert_events (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id         UUID NOT NULL REFERENCES operators(id),
    alert_type          VARCHAR(30) NOT NULL,
    -- alert_type: route_behind_schedule | sla_at_risk | driver_stopped | cascade_risk | driver_drop
    severity            VARCHAR(10) NOT NULL,  -- low | medium | high | critical
    status              VARCHAR(20) NOT NULL DEFAULT 'open',
    -- status: open | acknowledged | auto_executing | resolved | dismissed
    route_execution_id  UUID REFERENCES route_executions(id),
    stop_execution_id   UUID REFERENCES stop_executions(id),
    driver_id           UUID REFERENCES drivers(id),
    title               TEXT NOT NULL,
    description         TEXT,
    economic_risk_score NUMERIC(10,2),
    breach_probability  REAL,
    minutes_to_breach   INTEGER,
    intervention_options JSONB,
    -- [{rank, type, description, estimated_cost, estimated_time_saving, sla_stops_saved}]
    recommended_option  INTEGER,  -- rank of recommended option
    escalation_timeout_seconds INTEGER NOT NULL DEFAULT 300,
    escalation_sms_sent BOOLEAN NOT NULL DEFAULT FALSE,
    acknowledged_by     UUID REFERENCES dispatcher_users(id),
    acknowledged_at     TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    auto_executed       BOOLEAN NOT NULL DEFAULT FALSE,
    auto_executed_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### `intervention_actions`

```sql
CREATE TABLE intervention_actions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_event_id      UUID NOT NULL REFERENCES alert_events(id),
    operator_id         UUID NOT NULL REFERENCES operators(id),
    action_type         VARCHAR(40) NOT NULL,
    -- action_type: stop_reassigned | route_resequenced | customer_contacted |
    --              driver_notified | sla_waiver_logged | cascade_redistribution
    autonomous          BOOLEAN NOT NULL DEFAULT FALSE,
    executed_by         UUID REFERENCES dispatcher_users(id),  -- NULL if autonomous
    celery_task_id      VARCHAR(200),
    payload             JSONB NOT NULL,  -- full action payload for audit
    outcome             VARCHAR(20),     -- success | partial | failed
    outcome_notes       TEXT,
    reasoning           TEXT,            -- LLM or rules-engine reasoning string
    executed_at         TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### `cascade_incidents`

```sql
CREATE TABLE cascade_incidents (
    id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    operator_id                 UUID NOT NULL REFERENCES operators(id),
    dropped_driver_id           UUID NOT NULL REFERENCES drivers(id),
    trigger_type                VARCHAR(30) NOT NULL,
    -- trigger_type: manual_report | stationary_detected | vehicle_fault | sos
    affected_stop_count         INTEGER NOT NULL,
    at_risk_window_stop_count   INTEGER NOT NULL,
    redistribution_options      JSONB,
    -- [{rank, option_type, receiving_drivers: [{driver_id, stop_count, estimated_overtime}], sla_stops_saved, total_cost_estimate}]
    selected_option_rank        INTEGER,
    auto_executed               BOOLEAN NOT NULL DEFAULT FALSE,
    dispatcher_response_seconds INTEGER,  -- NULL if auto-executed
    stops_successfully_saved    INTEGER,
    stops_breached              INTEGER,
    incident_cost_estimate      NUMERIC(10,2),
    triggered_at                TIMESTAMPTZ NOT NULL,
    resolved_at                 TIMESTAMPTZ
);
```

#### `sla_breaches`

```sql
CREATE TABLE sla_breaches (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stop_execution_id   UUID NOT NULL REFERENCES stop_executions(id),
    client_id           UUID NOT NULL REFERENCES clients(id),
    operator_id         UUID NOT NULL REFERENCES operators(id),
    committed_window_end TIMESTAMPTZ NOT NULL,
    actual_arrival_time  TIMESTAMPTZ,
    breach_confirmed_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    penalty_amount       NUMERIC(10,2),
    penalty_status       VARCHAR(20) DEFAULT 'pending',
    -- penalty_status: pending | waived | invoiced | disputed | paid
    causal_factors       JSONB,
    -- {traffic_delay_minutes, access_failures, driver_lunch_minutes, cascade_from_driver_id, ...}
    intervention_attempted BOOLEAN NOT NULL DEFAULT FALSE,
    intervention_action_id UUID REFERENCES intervention_actions(id),
    notes               TEXT
);
```

#### `customer_contacts`

```sql
CREATE TABLE customer_contacts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stop_execution_id   UUID NOT NULL REFERENCES stop_executions(id),
    recipient_phone     VARCHAR(20) NOT NULL,
    channel             VARCHAR(10) NOT NULL DEFAULT 'sms',
    direction           VARCHAR(10) NOT NULL,  -- outbound | inbound
    message_body        TEXT NOT NULL,
    twilio_message_sid  VARCHAR(50),
    delivery_status     VARCHAR(20),
    -- delivery_status: queued | sent | delivered | failed | undelivered
    intent_classified   VARCHAR(30),
    -- For inbound: confirm_home | safe_drop | reschedule | opt_out | unknown
    parsed_window_preference VARCHAR(100),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

#### `eta_predictions`

```sql
CREATE TABLE eta_predictions (
    id                  BIGSERIAL PRIMARY KEY,
    stop_execution_id   UUID NOT NULL REFERENCES stop_executions(id),
    driver_id           UUID NOT NULL,
    prediction_input    JSONB NOT NULL,  -- full feature vector (for model audit)
    predicted_arrival   TIMESTAMPTZ NOT NULL,
    p10_arrival         TIMESTAMPTZ NOT NULL,
    p90_arrival         TIMESTAMPTZ NOT NULL,
    confidence_score    REAL NOT NULL,
    model_version       VARCHAR(20) NOT NULL,
    traffic_data_source VARCHAR(20) NOT NULL,  -- google_maps | cached | historical_fallback
    predicted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Partition by day; retain 90 days for model retraining
CREATE INDEX idx_eta_pred_stop ON eta_predictions(stop_execution_id);
CREATE INDEX idx_eta_pred_driver_time ON eta_predictions(driver_id, predicted_at DESC);
```

#### `dispatcher_actions`

```sql
CREATE TABLE dispatcher_actions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dispatcher_id       UUID NOT NULL REFERENCES dispatcher_users(id),
    operator_id         UUID NOT NULL REFERENCES operators(id),
    alert_event_id      UUID REFERENCES alert_events(id),
    action_type         VARCHAR(40) NOT NULL,
    payload             JSONB,
    session_id          VARCHAR(100),
    ip_address          INET,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Entity Relationships (Summary)

```
operators
  ├── drivers (many)
  ├── vehicles (many)
  ├── clients (many)
  │     └── sla_configs (one per client)
  ├── route_plans (many)
  │     └── route_executions (one per plan per day)
  │           └── stop_executions (many per route)
  │                 ├── eta_predictions (many, time-series)
  │                 ├── sla_breaches (zero or one)
  │                 └── customer_contacts (many)
  ├── alert_events (many)
  │     └── intervention_actions (many per alert)
  └── cascade_incidents (many)

driver_positions (TimescaleDB hypertable)
  └── FK: driver_id → drivers, route_id → route_executions
```
