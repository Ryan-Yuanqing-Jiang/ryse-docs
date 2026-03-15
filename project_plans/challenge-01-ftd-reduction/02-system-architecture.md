# Challenge 1: Real-Time Route Intelligence & FTD Reduction — System Architecture

---

## Tech Stack

### Frontend

| Layer | Technology | Rationale |
|---|---|---|
| Framework | React 18 + TypeScript 5 | Component model suits complex real-time dashboard state; TypeScript enforces strict data contracts across a large shared codebase |
| Map rendering | Mapbox GL JS 3.x | WebGL-based vector tile rendering handles 35 concurrent vehicle tracks at 1–5 second update intervals without frame drops; custom layer API supports risk score heatmaps and stop status overlays |
| State management | Zustand + React Query | Zustand for UI state (selected driver, active panel, filter state); React Query for server state with stale-while-revalidate patterns on delivery lists and route data |
| Real-time transport | WebSocket (native browser API) via a thin reconnect wrapper | Server-sent GPS updates pushed to frontend; avoids polling overhead on the 35-vehicle GPS feed |
| Charting | Recharts | Lightweight, composable; adequate for FTD trend charts, driver performance histograms, risk score distributions |
| Component library | shadcn/ui + Tailwind CSS 3 | Unstyled primitives + utility-first CSS; fast iteration on dense operational UIs |
| Build tooling | Vite 5 | Fast HMR for development; optimal chunk splitting for production |
| Testing | Vitest + React Testing Library + Playwright | Unit/component tests in Vitest; E2E dispatcher workflow tests in Playwright |

### Backend

All backend services are Python 3.12. The system is decomposed into six microservices, each independently deployable via Docker containers on AWS ECS Fargate.

| Service | Framework | Responsibility |
|---|---|---|
| `api-gateway` | FastAPI 0.115 + Uvicorn | Central ingress for all client-facing API traffic. JWT auth, rate limiting (slowapi), request routing to downstream services. |
| `risk-scoring-service` | FastAPI 0.115 | Hosts the GBDT inference pipeline. Accepts batch scoring requests (up to 500 parcels), returns risk scores with feature contribution explanations. |
| `sms-service` | FastAPI 0.115 + Celery 5 | Outbound SMS scheduling, inbound webhook processing, reply intent classification, preference persistence. |
| `route-optimizer-service` | FastAPI 0.115 | Wraps Google OR-Tools VRP solver. Accepts route optimisation requests with constraint sets. Returns optimised stop sequences. Event-triggered via Kafka consumer. |
| `cascade-agent-service` | FastAPI 0.115 + Celery 5 | Monitors driver delay events. Triggers at-risk stop detection, automated outreach, dispatcher notifications. |
| `learning-loop-service` | FastAPI 0.115 + Celery Beat | Collects delivery outcomes, triggers model retraining jobs, monitors model accuracy, publishes model artifacts to S3. |

### ML Stack

| Component | Technology |
|---|---|
| Primary model | XGBoost 2.x GBDT ensemble (Python) |
| Feature engineering | Pandas + custom `FeatureBuilder` class per model version |
| Feature store | Redis (online, <10ms lookup) + PostgreSQL (offline, batch training) |
| Model serialisation | joblib for XGBoost artifacts; versioned by `{model_name}/{version}/model.joblib` in S3 |
| Experiment tracking | MLflow (self-hosted on ECS, metadata in PostgreSQL, artifacts in S3) |
| Hyperparameter tuning | Optuna (async trial execution, SQLite backend for small-scale tuning runs) |
| LLM inference (SMS parsing) | OpenAI API (gpt-4o-mini); called only for ambiguous free-text SMS replies not matched by structured parser |

### Streaming

| Component | Technology |
|---|---|
| Message broker | Apache Kafka via AWS MSK (Managed Streaming for Kafka) |
| Stream processing | Kafka Streams (Java, deployed as a single consumer group service) for stateful aggregations (driver delay calculation, route completion percentage) |
| Producer clients | `confluent-kafka-python` in FastAPI services |
| Consumer clients | `confluent-kafka-python` in Celery workers and route-optimizer-service |
| GPS event topic | `gps.driver.positions` — keyed by `driver_id`, ~1–5 second publish interval per active driver |
| Stop event topic | `delivery.stop.events` — keyed by `route_id`, published on every stop state transition |
| Route event topic | `route.optimizer.events` — triggers for re-optimisation |
| SMS event topic | `sms.events` — inbound and outbound SMS state transitions |

### Cache and State

| Use Case | Technology | Configuration |
|---|---|---|
| Feature store (online) | Redis 7 (AWS ElastiCache) | Hash sets keyed by `address_id`, `recipient_id`, `driver_id`. TTL: 24h for address features, 7d for recipient preferences. |
| API response cache | Redis | Route data cached 5 minutes; risk score cache keyed by `{manifest_id}:{parcel_id}` invalidated on manifest update |
| Session state | Redis | JWT denylist for logout, rate limiter counters |
| Celery broker + result backend | Redis | Separate Redis instance from feature store to avoid eviction pressure |

### Databases

| Database | Technology | Data |
|---|---|---|
| Primary OLTP | PostgreSQL 16 (AWS RDS, Multi-AZ) | All business entities: deliveries, routes, drivers, recipients, risk scores, SMS conversations, FTD events |
| Time-series GPS | TimescaleDB 2.x (PostgreSQL extension, separate RDS instance) | `gps_positions` hypertable partitioned by `time` (1-day chunks). Retains 90 days of raw positions; downsampled 1-minute aggregates retained 365 days. |
| Migrations | Alembic | All schema migrations version-controlled in `alembic/versions/`. Applied automatically on service startup in staging; manual approval required in production. |

### Queue

| Component | Technology |
|---|---|
| Task queue | Celery 5 + Redis broker |
| Scheduled tasks | Celery Beat (nightly retraining trigger, morning pre-dispatch SMS batch, hourly model accuracy check) |
| Workers | Separate ECS task definitions per service: `sms-worker`, `learning-loop-worker`, `cascade-worker`. Autoscale on queue depth via AWS Application Auto Scaling. |

### SMS

| Component | Technology |
|---|---|
| Gateway | Twilio Messaging (A2P 10DLC registered for Australian market) |
| Inbound webhooks | Twilio POST webhook → `sms-service` `/webhooks/twilio/inbound` |
| Outbound | Twilio REST API via `twilio-python` SDK |
| Phone number pool | Twilio short code (if throughput >100 SMS/min) or 10DLC long code pool |
| Delivery receipts | Twilio status callback webhooks → `sms-service` `/webhooks/twilio/status` |

### Maps and Routing

| Component | Technology |
|---|---|
| Geocoding | Google Maps Geocoding API (address validation at manifest ingest) |
| Traffic-aware routing | Google Maps Directions API (used as input to OR-Tools for road segment travel times) |
| Route display | Mapbox GL JS (frontend map rendering, vector tiles) |
| VRP solver | Google OR-Tools 9.x (Python bindings, `ortools.constraint_solver`) |
| Driver navigation | Mapbox Navigation SDK (embedded in driver mobile app) |

### Infrastructure

| Component | Technology |
|---|---|
| Cloud | AWS (ap-southeast-2, Sydney region) |
| Container runtime | AWS ECS Fargate (serverless containers, no EC2 management) |
| Container registry | AWS ECR |
| Load balancer | AWS ALB (path-based routing to microservices) |
| Object storage | AWS S3 (model artifacts, training datasets, manifest file uploads) |
| Secrets | AWS Secrets Manager (DB credentials, API keys, Twilio auth) |
| DNS + CDN | AWS Route 53 + CloudFront (frontend static assets) |
| CI/CD | GitHub Actions (build → test → push ECR → deploy ECS) |
| IaC | Terraform (all AWS resources defined in `infra/terraform/`) |
| Monitoring | Datadog (APM, infrastructure metrics, log aggregation, custom dashboards) |
| Error tracking | Sentry (Python SDK in all services; React SDK in frontend) |
| Alerting | Datadog monitors → PagerDuty integration for P1/P2 incidents |

---

## AI Architecture

The AI pipeline is a sequence of five interconnected components that operate in two temporal modes: **pre-dispatch** (synchronous batch, triggered once per manifest upload, ~6:00am daily) and **in-flight** (asynchronous event-driven, continuous throughout the operating day).

### Component 1: Feature Engineering Service

**Temporal mode**: Both (batch for pre-dispatch, streaming for in-flight updates)

**Inputs**:
- Raw manifest data (parcel IDs, recipient addresses, phone numbers, parcel types, client IDs)
- Historical delivery outcomes table (`delivery_attempts`, PostgreSQL)
- Recipient preference store (Redis + `recipient_preferences` PostgreSQL table)
- Address metadata table (`addresses` PostgreSQL — geocoded coordinates, address type classification, postcode FTD rate)
- Weather forecast API (Bureau of Meteorology or OpenWeatherMap — queried once per morning batch)
- Driver roster (current day's driver assignments, experience ratings from `drivers` table)

**Processing**:
- `FeatureBuilder.build_address_features(address_id)`: Queries address history — last 90-day FTD rate, number of attempts, failure mode distribution (not home / access / wrong address), apartment/commercial flag, geocoded coordinates
- `FeatureBuilder.build_recipient_features(recipient_id)`: Prior SMS engagement rate, number of confirmed windows in history, number of prior failures, last known delivery preference
- `FeatureBuilder.build_temporal_features(delivery_date, planned_window)`: Day of week encoding, hour of day for start/end of window, public holiday flag, school term flag (affects residential afternoon deliveries)
- `FeatureBuilder.build_driver_features(driver_id, address_type)`: Driver FTD rate by address type (apartment vs. residential vs. commercial), driver total deliveries (experience proxy), driver current-day load
- `FeatureBuilder.build_external_features(postcode, delivery_date)`: Postcode-level FTD rate (rolling 30-day), weather forecast (rain probability — increases not-home rate for residential), postcode density category

**Outputs**: Feature vectors written to Redis feature store (key: `features:{parcel_id}`) and logged to `feature_log` PostgreSQL table for training data collection.

**Latency budget**: Batch processing 3,500 parcels in <45 seconds total (parallelised with `asyncio` + connection pooling).

### Component 2: Risk Scoring Model

**Temporal mode**: Pre-dispatch batch (primary); on-demand re-score for late-added parcels

**Model type**: XGBoost GBDT ensemble. Primary model: 300 trees, max depth 6, learning rate 0.05. Trained on rolling 90-day window of delivery outcomes (~270,000 labelled examples at steady state). Feature set: 47 features across address, recipient, temporal, driver, and external categories.

**Inputs**: Feature vectors from Feature Engineering Service (via Redis lookup or direct batch array)

**Outputs**:
- `risk_score` (float, 0–100): Calibrated probability of failure × 100, via Platt scaling post-processing
- `risk_tier` (enum: LOW / MEDIUM / HIGH / CRITICAL): LOW <35, MEDIUM 35–65, HIGH 65–85, CRITICAL >85
- `top_risk_factors` (list of 3): SHAP values identifying the top contributing features, used for dispatcher UI explainability (e.g., "High risk: apartment building (no intercom history), recipient never engaged with prior SMS")
- `recommended_intervention` (enum): NONE / SMS_CONFIRM / DISPATCHER_REVIEW / MANUAL_RESCHEDULE

**Latency budget**: <200ms for batch of 500 parcels (single container); <20ms for single parcel on-demand re-score.

**Model versioning**: Each model artifact stored in S3 at `s3://odocs-models/risk-scoring/{version}/model.joblib`. Active version tracked in `model_registry` PostgreSQL table. Rollback by updating `is_active` flag — no redeployment required.

### Component 3: SMS Decision Engine

**Temporal mode**: Pre-dispatch (initial send batch) and event-driven (re-sends, reply processing)

**Inputs**:
- Risk scores and `recommended_intervention` from Risk Scoring Model
- Recipient opt-out status (Redis set `sms:optouts`)
- Recipient contact preferences (`recipient_preferences` table — preferred contact time, channel, language)
- Planned delivery windows from route plan

**Processing**:
- `SMSDecisionEngine.should_send(parcel_id)`: Checks risk tier >= HIGH, opt-out status, prior engagement in last 24h (avoid duplicate contact), TCPA quiet hours (8am–9pm local time enforcement)
- `SMSScheduler.schedule_day_before(parcel_id)`: Creates `SMSJob` Celery task for 4:00–6:00pm day prior (within quiet hours, batched to avoid carrier throttling)
- `SMSScheduler.schedule_pre_arrival(parcel_id, eta)`: Creates `SMSJob` task for 2 hours before predicted driver arrival at this stop
- `ReplyProcessor.process(twilio_payload)`: Parses inbound SMS. Structured parser first (regex on "1"/"2"/"3"/"4" responses). LLM fallback (GPT-4o-mini) for free text with prompt: "Extract delivery intent from this SMS reply. Return JSON: {intent: confirm|reschedule|access_instructions|redirect, details: string|null}"
- `PreferenceStore.update(recipient_id, intent, details)`: Persists extracted preference to `recipient_preferences` and pushes access instructions to driver app via `route-optimizer-service` stop update endpoint

**Outputs**: `SMSConversation` records written to PostgreSQL; driver stop notes updated; route time windows adjusted for confirmed/rescheduled slots.

**Latency budget**: Inbound webhook processing <500ms (Twilio requires 200 OK within 15 seconds or it retries).

### Component 4: Route Optimization Engine

**Temporal mode**: Pre-dispatch (initial route build) and event-driven in-flight re-optimisation

**Inputs**:
- Parcel list with risk scores, time windows, and recipient preferences
- Driver and vehicle roster (vehicle capacity, driver start/end depot, shift end time)
- Real-time driver positions (from Kafka `gps.driver.positions` topic consumer)
- Traffic travel time matrix (Google Maps Distance Matrix API, queried for current conditions)
- Re-optimisation trigger event (from Kafka `route.optimizer.events` topic)

**Processing (pre-dispatch)**:
- Build VRP problem instance using OR-Tools `RoutingModel`: nodes are depot + all delivery stops; arc costs are traffic-aware travel times; capacity constraints on vehicle loads; time window constraints on delivery stops; driver shift end as hard deadline.
- Solver configuration: `AUTOMATIC` first solution strategy + `GUIDED_LOCAL_SEARCH` metaheuristic, 30-second time limit for initial solve.
- Risk-weighted cost function: stops with HIGH/CRITICAL risk scores that have SMS confirmation (confirmed window) get tightened time windows; unconfirmed HIGH risk stops get dispatched earlier in sequence to allow cascade recovery time.

**Processing (in-flight re-optimisation)**:
- Triggered by events: driver delay >15 minutes against plan, stop marked failed, manual dispatcher override.
- Re-optimise remaining stops only (completed stops excluded). Update travel time matrix for current conditions. Solve with 10-second time limit.
- Publish updated stop sequence to `route.optimizer.events` Kafka topic → consumed by `api-gateway` → pushed to driver app via WebSocket.

**Outputs**: Route objects written to `routes` and `route_stops` PostgreSQL tables; dispatched to driver app; visualised on dispatcher map.

**Latency budget**: Pre-dispatch route build <60 seconds for full 35-vehicle fleet. In-flight re-optimisation for single driver <8 seconds.

### Component 5: Continuous Learning Loop

**Temporal mode**: Batch (nightly retraining) + continuous outcome capture

**Inputs**:
- Delivery outcomes written to `delivery_attempts` as drivers complete/fail stops throughout the day
- Feature vectors logged to `feature_log` at scoring time (ensures training data exactly matches inference-time features)
- Model accuracy metrics from `model_accuracy_log` table

**Processing**:
- `OutcomeCapture.record(parcel_id, outcome, failure_reason)`: Joins `feature_log` entry with delivery outcome to create labelled training record. Written to `training_data` table.
- `RetrainingScheduler` (Celery Beat, runs nightly at 2:00am AEST): Pulls last 90 days of labelled records from `training_data`. Trains new XGBoost model with Optuna hyperparameter search (50 trials). Evaluates on held-out 20% split. If new model AUC-ROC >= current model AUC-ROC - 0.01 (allows up to 1pp degradation), promotes to candidate. If new model AUC-ROC > current + 0.005, auto-promotes to active. Otherwise, alerts on-call ML engineer via PagerDuty.
- `DriftDetector.check()` (hourly): Computes KL divergence between last 7-day risk score distribution and training-time distribution. If divergence >0.15, triggers alert and flags model for review.
- `AccuracyMonitor.compute()` (end of each operating day): Computes precision/recall at HIGH risk threshold. Logs to `model_accuracy_log`. Feeds the weekly accuracy dashboard.

**Outputs**: Updated model artifacts in S3; model registry updated; accuracy metrics in Datadog.

---

## Integration Layer

### Client Manifest Ingest

**Pattern**: REST API + SFTP polling fallback

Clients submit delivery manifests via `POST /api/v1/manifests` (JSON, authenticated with per-client API key). Manifest payload: array of parcels with recipient name, address, phone, parcel type, client reference ID, and optional time window preference. The API validates, geocodes unmatched addresses via Google Maps Geocoding API, creates `Delivery` records, and enqueues the risk scoring batch job.

Legacy clients unable to integrate via REST submit CSV files to a client-specific SFTP directory on AWS Transfer Family. A polling Lambda (every 5 minutes, 5:00am–7:00am window) detects new files, parses them, and submits to the same manifest ingest API.

### Driver App Integration

**Pattern**: REST API for CRUD + WebSocket for real-time push + Kafka for GPS streaming

The driver mobile app (React Native) authenticates with a driver JWT, fetches their daily route on app open (`GET /api/v1/routes/{route_id}/driver`), and receives stop updates via WebSocket connection to `api-gateway`. GPS positions are published to the backend at 1–5 second intervals via `POST /api/v1/gps/position` (batched in 5-second windows client-side to reduce network overhead) — the API gateway forwards these to Kafka `gps.driver.positions` topic.

Stop completion events (successful delivery, failed attempt with reason code, driver note) are submitted via `POST /api/v1/stops/{stop_id}/complete`. Access instructions and recipient notes are surfaced on the stop detail screen, pulled from `route_stops.driver_notes` field populated by SMS reply processing.

### Twilio Webhooks

**Pattern**: POST webhooks to `sms-service`, validated via Twilio signature verification middleware

- Inbound SMS: `POST /webhooks/twilio/inbound` — receives `From`, `Body`, `MessageSid`. Route matched to open `SMSConversation` by `From` phone number. Intent extracted and preference updated.
- Delivery status: `POST /webhooks/twilio/status` — receives `MessageSid`, `MessageStatus` (sent/delivered/failed). Updates `sms_messages.status`. Failed delivery triggers fallback (push notification if driver app installed, or dispatcher alert).

### Google Maps Platform

**Pattern**: Server-side REST API calls from `route-optimizer-service` and manifest ingest

- Geocoding API: Called during manifest ingest for each unresolved address. Results cached in `addresses.geocode_cache` (JSONB column) to avoid re-geocoding known addresses. Cache hit rate >90% at steady state.
- Distance Matrix API: Called pre-dispatch and during re-optimisation to build travel time matrices. Batched to respect 100 elements/request limit. Results cached in Redis with 30-minute TTL (traffic conditions stale quickly).

### Client SLA Reporting Portal

**Pattern**: Read-only API endpoints + scheduled report generation

Clients access a white-labelled portal at `{client_subdomain}.platform.com`. The portal surfaces: daily delivery summary (success rate, FTD rate, SLA compliance), individual parcel tracking (recipient-facing tracking link also generated from this), weekly trend charts. Data served from read replica PostgreSQL to avoid OLTP load. Scheduled daily PDF reports generated by a Celery Beat job at 8:00pm, stored in S3, and emailed via AWS SES.

### External Data Sources

| Source | Integration | Data |
|---|---|---|
| Bureau of Meteorology (BOM) | HTTP polling, daily at 5:30am | Rainfall probability, temperature (feature for risk model) |
| Google Maps Traffic | REST API, on-demand | Real-time and predicted traffic for travel time matrix |
| Australia Post Postcode Database | Static import, monthly refresh | Postcode → suburb → region mapping, used in postcode FTD rate feature |

---

## Data Models

The following describes the core PostgreSQL schema. All tables include `created_at TIMESTAMPTZ DEFAULT NOW()` and `updated_at TIMESTAMPTZ DEFAULT NOW()` unless noted.

### `deliveries`

The canonical record for a single parcel delivery assignment within a day's manifest.

```
id                  UUID PRIMARY KEY
manifest_id         UUID NOT NULL REFERENCES manifests(id)
client_id           UUID NOT NULL REFERENCES clients(id)
recipient_id        UUID NOT NULL REFERENCES recipients(id)
address_id          UUID NOT NULL REFERENCES addresses(id)
parcel_type         VARCHAR(50)           -- 'standard' | 'fragile' | 'signature_required' | 'healthcare'
client_reference    VARCHAR(100)          -- client's own order/parcel ID
status              VARCHAR(30)           -- 'pending' | 'scheduled' | 'in_transit' | 'delivered' | 'failed' | 'rescheduled'
planned_window_start TIMESTAMPTZ
planned_window_end   TIMESTAMPTZ
actual_delivery_at  TIMESTAMPTZ
delivery_date       DATE NOT NULL
```

### `delivery_attempts`

Each physical delivery attempt by a driver. A `delivery` may have 1–3 attempts.

```
id                  UUID PRIMARY KEY
delivery_id         UUID NOT NULL REFERENCES deliveries(id)
driver_id           UUID NOT NULL REFERENCES drivers(id)
route_stop_id       UUID NOT NULL REFERENCES route_stops(id)
attempt_number      INT NOT NULL
outcome             VARCHAR(30)           -- 'success' | 'not_home' | 'access_failure' | 'wrong_address' | 'refused' | 'out_of_time'
attempted_at        TIMESTAMPTZ
completed_at        TIMESTAMPTZ
failure_reason      TEXT
driver_note         TEXT
photo_url           VARCHAR(500)          -- S3 URL for proof of delivery photo
gps_lat             DECIMAL(9,6)
gps_lng             DECIMAL(9,6)
```

### `risk_scores`

One record per delivery per scoring run. Multiple records may exist if a parcel is re-scored.

```
id                  UUID PRIMARY KEY
delivery_id         UUID NOT NULL REFERENCES deliveries(id)
model_version       VARCHAR(20) NOT NULL
scored_at           TIMESTAMPTZ NOT NULL
risk_score          DECIMAL(5,2)          -- 0.00 to 100.00
risk_tier           VARCHAR(10)           -- 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL'
top_risk_factors    JSONB                 -- [{feature: string, contribution: float, direction: 'increase'|'decrease'}]
recommended_intervention VARCHAR(30)
is_active           BOOLEAN DEFAULT TRUE  -- false when superseded by re-score
feature_snapshot    JSONB                 -- full feature vector at scoring time (for training data)
```

### `recipients`

De-duplicated recipient entities, matched across deliveries by phone number normalisation.

```
id                  UUID PRIMARY KEY
phone_e164          VARCHAR(20) UNIQUE    -- normalised E.164 format
name                VARCHAR(200)
email               VARCHAR(200)
sms_opt_out         BOOLEAN DEFAULT FALSE
sms_opt_out_at      TIMESTAMPTZ
engagement_rate     DECIMAL(4,3)          -- rolling 90-day SMS response rate (computed column)
total_deliveries    INT DEFAULT 0         -- updated by trigger on delivery_attempts
total_failures      INT DEFAULT 0
```

### `recipient_preferences`

Structured preferences extracted from SMS replies and manual input.

```
id                  UUID PRIMARY KEY
recipient_id        UUID NOT NULL REFERENCES recipients(id)
preference_type     VARCHAR(50)           -- 'time_window' | 'access_instructions' | 'safe_drop' | 'neighbour_drop' | 'collection_point'
value               TEXT NOT NULL
source              VARCHAR(30)           -- 'sms_reply' | 'dispatcher_manual' | 'customer_portal'
confidence          DECIMAL(3,2)          -- 0.00 to 1.00, LLM extraction confidence
valid_from          TIMESTAMPTZ
valid_until         TIMESTAMPTZ           -- null = indefinite
address_id          UUID REFERENCES addresses(id)  -- null = applies to any address
```

### `addresses`

Geocoded and enriched address records. Shared across all deliveries to the same physical location.

```
id                  UUID PRIMARY KEY
raw_address         TEXT NOT NULL
normalised_address  TEXT
street_number       VARCHAR(20)
street_name         VARCHAR(100)
suburb              VARCHAR(100)
postcode            VARCHAR(10)
state               VARCHAR(3)
lat                 DECIMAL(9,6)
lng                 DECIMAL(9,6)
address_type        VARCHAR(30)           -- 'residential_house' | 'residential_apartment' | 'commercial' | 'medical' | 'industrial'
apartment_complex_id UUID REFERENCES apartment_complexes(id)
geocode_cache       JSONB                 -- raw Google Maps geocode API response
ftd_rate_30d        DECIMAL(4,3)          -- computed nightly by learning loop
total_deliveries    INT DEFAULT 0
total_failures      INT DEFAULT 0
```

### `routes`

One route per driver per day.

```
id                  UUID PRIMARY KEY
driver_id           UUID NOT NULL REFERENCES drivers(id)
vehicle_id          UUID NOT NULL REFERENCES vehicles(id)
delivery_date       DATE NOT NULL
status              VARCHAR(20)           -- 'planned' | 'active' | 'completed' | 'incomplete'
planned_start_at    TIMESTAMPTZ
actual_start_at     TIMESTAMPTZ
planned_end_at      TIMESTAMPTZ
actual_end_at       TIMESTAMPTZ
depot_id            UUID NOT NULL REFERENCES depots(id)
total_stops         INT
completed_stops     INT DEFAULT 0
optimiser_version   VARCHAR(20)           -- OR-Tools version + solve parameters hash
reoptimisation_count INT DEFAULT 0       -- incremented each time mid-day re-opt runs
```

### `route_stops`

Ordered stops within a route. Reordered on re-optimisation; history preserved via `sequence_version`.

```
id                  UUID PRIMARY KEY
route_id            UUID NOT NULL REFERENCES routes(id)
delivery_id         UUID NOT NULL REFERENCES deliveries(id)
sequence_order      INT NOT NULL
sequence_version    INT NOT NULL DEFAULT 1   -- incremented on each re-optimisation
planned_arrival     TIMESTAMPTZ
predicted_arrival   TIMESTAMPTZ              -- updated in real time
actual_arrival      TIMESTAMPTZ
status              VARCHAR(20)              -- 'pending' | 'in_progress' | 'completed' | 'failed' | 'skipped'
driver_notes        TEXT                     -- populated from SMS reply processing
is_risk_flagged     BOOLEAN DEFAULT FALSE
cascade_at_risk     BOOLEAN DEFAULT FALSE    -- set by cascade agent
```

### `sms_conversations`

One conversation per delivery, tracking the full two-way exchange.

```
id                  UUID PRIMARY KEY
delivery_id         UUID NOT NULL REFERENCES deliveries(id)
recipient_id        UUID NOT NULL REFERENCES recipients(id)
status              VARCHAR(20)           -- 'pending' | 'sent' | 'delivered' | 'replied' | 'expired' | 'opted_out'
outcome             VARCHAR(30)           -- 'confirmed' | 'rescheduled' | 'access_provided' | 'redirected' | 'no_reply'
day_before_sent_at  TIMESTAMPTZ
day_before_replied_at TIMESTAMPTZ
pre_arrival_sent_at TIMESTAMPTZ
pre_arrival_replied_at TIMESTAMPTZ
extracted_intent    VARCHAR(30)
extracted_details   TEXT
intent_confidence   DECIMAL(3,2)
llm_used            BOOLEAN DEFAULT FALSE  -- true if GPT fallback was invoked
```

### `sms_messages`

Individual messages within a conversation.

```
id                  UUID PRIMARY KEY
conversation_id     UUID NOT NULL REFERENCES sms_conversations(id)
direction           VARCHAR(10)           -- 'outbound' | 'inbound'
body                TEXT NOT NULL
twilio_message_sid  VARCHAR(50)
status              VARCHAR(20)           -- 'queued' | 'sent' | 'delivered' | 'failed' | 'received'
sent_at             TIMESTAMPTZ
delivered_at        TIMESTAMPTZ
```

### `drivers`

```
id                  UUID PRIMARY KEY
name                VARCHAR(200) NOT NULL
phone_e164          VARCHAR(20)
license_number      VARCHAR(50)
employment_type     VARCHAR(20)           -- 'employee' | 'contractor'
experience_months   INT
ftd_rate_overall    DECIMAL(4,3)          -- rolling 90-day, computed by learning loop
ftd_rate_apartment  DECIMAL(4,3)
ftd_rate_residential DECIMAL(4,3)
ftd_rate_commercial DECIMAL(4,3)
languages           VARCHAR[]             -- for driver-recipient language matching feature
is_active           BOOLEAN DEFAULT TRUE
```

### `vehicles`

```
id                  UUID PRIMARY KEY
registration        VARCHAR(20) UNIQUE
vehicle_type        VARCHAR(30)           -- 'van_small' | 'van_large' | 'truck' | 'bike'
capacity_parcels    INT
capacity_kg         DECIMAL(6,2)
depot_id            UUID REFERENCES depots(id)
is_active           BOOLEAN DEFAULT TRUE
```

### `ftd_events`

Immutable audit log of every first-time delivery failure. Source of truth for SLA reporting and model training labels.

```
id                  UUID PRIMARY KEY
delivery_id         UUID NOT NULL REFERENCES deliveries(id) UNIQUE
delivery_attempt_id UUID NOT NULL REFERENCES delivery_attempts(id)
failure_reason      VARCHAR(50)           -- standardised reason code
risk_score_at_dispatch DECIMAL(5,2)       -- denormalised from risk_scores for fast reporting
risk_tier_at_dispatch VARCHAR(10)
sms_sent            BOOLEAN
sms_outcome         VARCHAR(30)
cascade_at_risk_flagged BOOLEAN
sla_breach          BOOLEAN
sla_penalty_amount  DECIMAL(8,2)
recorded_at         TIMESTAMPTZ NOT NULL
```

### Entity Relationship Summary

```
manifests ──< deliveries >── recipients
                   │               │
                   ├── risk_scores  └── recipient_preferences
                   ├── route_stops >── routes >── drivers
                   │                              vehicles
                   ├── delivery_attempts
                   ├── sms_conversations >── sms_messages
                   ├── ftd_events
                   └── addresses >── apartment_complexes
```

### Key Indexes

```sql
-- Critical query path: pre-dispatch risk scoring batch
CREATE INDEX idx_deliveries_delivery_date ON deliveries(delivery_date);
CREATE INDEX idx_deliveries_manifest_status ON deliveries(manifest_id, status);

-- Real-time GPS lookups
CREATE INDEX idx_gps_positions_driver_time ON gps_positions(driver_id, time DESC);  -- TimescaleDB

-- SMS conversation lookup by phone (inbound webhook hot path)
CREATE INDEX idx_recipients_phone ON recipients(phone_e164);
CREATE INDEX idx_sms_conversations_recipient_delivery ON sms_conversations(recipient_id, delivery_id);

-- FTD reporting
CREATE INDEX idx_ftd_events_recorded_at ON ftd_events(recorded_at);
CREATE INDEX idx_delivery_attempts_outcome ON delivery_attempts(outcome, attempted_at);

-- Route stop lookup (real-time dispatcher map)
CREATE INDEX idx_route_stops_route_status ON route_stops(route_id, status);
```
