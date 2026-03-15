# Challenge 3: Live Operational Dashboard & Exception Alerting — Modular Breakdown

---

## Module 1: GPS Streaming Pipeline

### Purpose

Ingest GPS pings from 35 driver mobile applications at 5-second intervals, persist them to the time-series database, maintain a real-time current-state cache in Redis, and publish location events to the Kafka event bus for downstream stream processing. This is the foundation data pipeline on which all real-time features depend. It must be reliable, low-latency, and degrade gracefully under connectivity loss.

### Key Functions

1. **GPS Ingest Endpoint:** Accept POST requests from the driver app, validate payload, write to TimescaleDB asynchronously, update Redis current state, produce to Kafka `driver-locations` topic.
2. **Stationary Detection:** After each GPS write, evaluate if the driver has been stationary (speed < 3 km/h, position delta < 50m) for more than 10 minutes. If yes and the driver is not at a registered stop location (geofence radius 100m), produce a `driver_stationary` event to Kafka `alert-triggers`.
3. **Connection Health Monitoring:** Track last-received timestamp per driver. If no GPS ping received for > 30 seconds during an active route, flag the driver as `connection_degraded` in Redis and emit a `connection_lost` event.
4. **Battery Optimisation Protocol:** If the driver app reports battery < 20%, switch to 30-second GPS intervals. Communicate this mode change back to the app via the driver's Socket.io room.
5. **Kafka Fallback Write:** If Kafka is unavailable, buffer GPS events to a PostgreSQL `kafka_outbox` table. A background worker (`tasks/kafka_drain.py`) drains the outbox to Kafka when connectivity resumes.

### API Endpoints

```
POST /api/v1/gps
  Auth: Bearer JWT (driver-scoped)
  Body: { driver_id, route_id, lat, lng, speed_kmh, heading_degrees, accuracy_metres, timestamp_utc, battery_pct }
  Response: 202 Accepted
  Latency target: <50ms p99

GET /api/v1/drivers/{driver_id}/position
  Auth: Bearer JWT (dispatcher-scoped)
  Response: { driver_id, lat, lng, speed_kmh, heading, recorded_at, connection_status }
  Source: Redis driver:{driver_id}:position hash
  Latency target: <20ms

GET /api/v1/fleet/positions
  Auth: Bearer JWT (dispatcher-scoped, operator-scoped)
  Response: Array of current positions for all active drivers in operator's fleet
  Source: Redis SCAN driver:*:position (filtered by operator_id)
  Latency target: <100ms for 35 drivers
```

### Kafka Events Produced

- **Topic: `driver-locations`**
  - Schema: `{ event_id, driver_id, operator_id, route_id, lat, lng, speed_kmh, heading, accuracy, recorded_at }`
  - Key: `driver_id` (ensures ordering per driver)
  - Partition strategy: 35 drivers → 10 partitions (3–4 drivers per partition)

- **Topic: `alert-triggers`** (from stationary detection)
  - Schema: `{ trigger_type: "driver_stationary", driver_id, route_id, lat, lng, stationary_since, minutes_stationary }`

### State Management

Redis key schema for GPS pipeline:

```
driver:{driver_id}:position       → Hash (lat, lng, speed, heading, recorded_at, route_id)  TTL: 30s
driver:{driver_id}:connection     → Hash (status, last_ping_at, degraded_since)              TTL: 120s
driver:{driver_id}:stationary     → Hash (since_timestamp, lat, lng, minutes)               TTL: 600s
operator:{operator_id}:active_drivers → Set of driver_ids                                    No TTL
```

### Dependencies

- PostgreSQL + TimescaleDB (`driver_positions` hypertable)
- Redis ElastiCache
- Kafka on AWS MSK (`driver-locations`, `alert-triggers` topics)
- Driver mobile app (React Native, GPS location APIs)
- AWS Secrets Manager (JWT signing keys)

---

## Module 2: ETA Prediction Service

### Purpose

Provide continuously updated, statistically calibrated arrival time predictions for every pending stop on every active route. Predictions must include confidence intervals (P10/P90) to enable probabilistic SLA breach scoring downstream. Inference must complete within 200ms to stay within the real-time update cycle.

### Key Functions

1. **Feature Assembly:** On each GPS event consumed from `driver-locations`, assemble the 14-feature vector required for GBDT inference: current position, speed, remaining stop count, distance to next stop, distance to each remaining stop, historical average stop duration by suburb/zone, current traffic segment travel times, time of day (hour + minute as cyclical features), day of week, driver's personal stop duration percentile (derived from historical data), route type (residential/commercial/industrial).
2. **GBDT Inference:** Call LightGBM model loaded at service startup. Output predicted arrival time with confidence interval. Log prediction to `eta_predictions` table.
3. **Traffic Data Integration:** Fetch Google Maps Routes API for the driver's current leg (current position → next stop). Cache result in Redis `traffic:{route_segment_hash}` with 3-minute TTL. Use cached value if available; fresh API call if stale.
4. **Prediction Cache Write:** Write prediction result to Redis `eta:{stop_id}:prediction` (TTL 30s) for SLA Breach Evaluator to consume without hitting the database.
5. **Model Version Tracking:** Every prediction record includes `model_version` field. When a new model is deployed, predictions from the old and new model are logged separately for A/B comparison during the first 7 days post-deployment.
6. **Staleness Flagging:** If traffic data is older than 3 minutes (cached fallback), or if the driver's last GPS ping is older than 30 seconds, set `traffic_data_source = historical_fallback` and reduce `confidence_score` accordingly.

### Feature Vector (14 Features)

| # | Feature | Source |
|---|---|---|
| 1 | Distance to next stop (km) | Computed from current GPS + stop lat/lng |
| 2 | Distance to all remaining stops sum (km) | Computed |
| 3 | Current speed (km/h) | GPS ping |
| 4 | Remaining stop count | Route execution record |
| 5 | Current traffic multiplier (current leg) | Google Maps API |
| 6 | Time of day — hour (0–23) | System clock |
| 7 | Time of day — minute as sin/cos (cyclical) | System clock |
| 8 | Day of week (0–6) | System clock |
| 9 | Historical avg stop duration — zone (seconds) | PostgreSQL `zone_stop_durations` |
| 10 | Driver personal stop duration percentile | PostgreSQL `driver_stats` |
| 11 | Zone type (residential/commercial/industrial) | Geocoding cache |
| 12 | Historical traffic multiplier — zone × hour | PostgreSQL `zone_traffic_history` |
| 13 | Route completion percentage | Route execution record |
| 14 | Days since driver joined (experience proxy) | Driver record |

### API Endpoints

```
POST /internal/predict-eta
  Auth: Internal service JWT (not exposed externally)
  Body: { stop_execution_id, driver_id, feature_vector: [f1..f14], committed_window_end }
  Response: {
    stop_execution_id,
    predicted_arrival: "ISO8601",
    p10_arrival: "ISO8601",
    p90_arrival: "ISO8601",
    confidence_score: 0.0–1.0,
    model_version: "v2.3.1",
    traffic_data_source: "google_maps|cached|historical_fallback",
    inference_ms: 45
  }
  Latency target: <200ms p99

GET /internal/model/health
  Response: { model_version, last_loaded_at, inference_count_today, avg_latency_ms_p99 }

POST /internal/model/reload
  Auth: Internal admin JWT
  Triggers: reload LightGBM model artifact from S3
```

### State Management

```
eta:{stop_id}:prediction        → JSON (full prediction object)  TTL: 30s
traffic:{segment_hash}:travel   → JSON (Google Maps response)    TTL: 180s
model:current_version           → String                         No TTL
model:inference_count           → Counter (daily)                TTL: 86400s
```

### Dependencies

- LightGBM 4.x (Python package, model artifact loaded from S3 `s3://odocs-models/eta-gbdt/latest.lgb`)
- Google Maps Routes API (env var: `GOOGLE_MAPS_API_KEY`)
- Redis (traffic cache, prediction cache)
- PostgreSQL (`zone_stop_durations`, `driver_stats`, `zone_traffic_history` tables)
- Kafka consumer (`driver-locations` topic, consumer group: `eta-recalc`)

---

## Module 3: SLA Breach Evaluator

### Purpose

For every pending stop on every active route, continuously evaluate the probability that the predicted arrival will breach the committed delivery window. Score each at-risk stop by its economic cost of breach. Trigger downstream alerts when breach probability crosses configured thresholds.

### Key Functions

1. **Per-Stop SLA Window Tracking:** At route start, load all stop SLA windows from `stop_executions` into Redis for O(1) lookup during stream processing. On each ETA prediction update, evaluate the stop's breach risk.
2. **Breach Probability Scoring:** Given predicted arrival distribution (P10/P90/median from GBDT), compute `P(arrival > window_end)`. Method: assume log-normal distribution parameterised by P10/P90; integrate above `window_end`.
3. **Economic Risk Score Calculation:** `economic_risk_score = breach_probability × penalty_amount + (1 - breach_probability) × 0`. Where `penalty_amount` is sourced from the client's SLA config for this delivery category.
4. **Threshold Evaluation:** Compare `breach_probability` and `economic_risk_score` against per-client alert thresholds. Default thresholds: `sla_at_risk` trigger at 60% breach probability; `critical` trigger at 80%. Produce to `alert-triggers` if threshold crossed and no open alert already exists for this stop.
5. **Alert Deduplication State:** Maintain Redis set `alert:open_stop_ids` with TTL per entry matching the stop's committed window. Before producing to `alert-triggers`, check membership. If stop is already in set, update the existing alert record rather than creating a new trigger.
6. **Client SLA Configuration:** SLA configs are loaded into Redis at operator login and refreshed every 5 minutes. Schema: `sla_config:{client_id}` → Hash with fields: `breach_probability_threshold`, `economic_risk_threshold`, `penalty_soft_window_minutes`, `delivery_category_penalties` (JSON).
7. **Soft Window Buffer:** Some clients define a soft buffer (e.g., "window is 14:00–16:00 but we don't penalise until 16:15"). The evaluator accounts for this: breach probability calculation targets `window_end + soft_buffer_minutes`.

### API Endpoints

```
GET /api/v1/routes/{route_id}/sla-status
  Auth: Dispatcher JWT
  Response: Array of stops with current breach_probability, economic_risk_score, predicted_arrival
  Source: Redis (hot path) with PostgreSQL fallback

GET /api/v1/operators/{operator_id}/sla-health
  Auth: Dispatcher JWT
  Response: {
    total_active_stops: integer,
    stops_at_risk_count: integer,
    stops_critical_count: integer,
    total_economic_risk_dollars: numeric,
    expected_penalty_no_intervention: numeric,
    routes_behind_schedule: integer
  }
  Latency target: <500ms (aggregated from Redis)

PATCH /api/v1/clients/{client_id}/sla-config
  Auth: Admin JWT
  Body: { breach_probability_threshold, economic_risk_threshold, penalty_amounts: {...} }
  Effect: Update PostgreSQL + refresh Redis cache
```

### WebSocket Events Emitted

- `sla:stop_risk_updated` → `{ stop_id, breach_probability, economic_risk_score, predicted_arrival, minutes_to_breach }`
- `sla:portfolio_health_updated` → `{ total_at_risk, total_critical, total_economic_risk_dollars }` (emitted every 30 seconds, aggregated)

### State Management

```
sla_config:{client_id}              → Hash (thresholds, penalties)            TTL: 300s
sla:breach_prob:{stop_id}           → Float                                    TTL: 60s
sla:economic_risk:{stop_id}         → Numeric string                           TTL: 60s
alert:open_stop_ids:{operator_id}   → Set of stop_ids with current open alerts TTL: per-window
sla:portfolio:{operator_id}         → Hash (aggregated health metrics)         TTL: 30s
```

### Dependencies

- ETA Prediction Service (Redis `eta:{stop_id}:prediction` reads)
- PostgreSQL (`stop_executions`, `clients`, `sla_configs` tables)
- Redis
- Kafka producer (`alert-triggers` topic)

---

## Module 4: Alert Decision Engine

### Purpose

Consume alert trigger events from Kafka, apply alert taxonomy and prioritisation logic, deduplicate against open alerts, generate ranked intervention options, push structured alerts to connected dispatcher clients via WebSocket, schedule escalation SMS if unacknowledged, and manage the full alert lifecycle from open through resolution or auto-execution.

### Key Functions

1. **Alert Taxonomy Classification:** Classify each incoming trigger into one of five alert types based on trigger payload: `route_behind_schedule` (>15 min behind ETA), `sla_at_risk` (breach probability >60%), `driver_stopped` (>10 min stationary, not at stop), `cascade_risk` (if this driver fails, N stops are affected), `driver_drop` (confirmed drop event).
2. **Economic Priority Ranking:** When multiple alerts are open simultaneously, rank them by `economic_risk_score DESC`. This ranking is what the dispatcher sees in the AlertPanel — not time order, not route order, but dollar-cost order.
3. **Intervention Option Generation:** For each alert, generate up to 3 intervention options using a rules engine (`services/intervention_generator.py`). Option types: `reassign_stop` (move specific stop to another driver), `reroute_driver` (resequence remaining stops for speed), `contact_customer_proactive` (trigger proactive comms to convert failure to reschedule), `manual_review` (no automated option; escalate to dispatcher judgment). Each option includes: `estimated_time_saving_minutes`, `estimated_cost_delta`, `sla_stops_saved`, `implementation_steps`.
4. **Alert Deduplication:** Before writing a new AlertEvent, check PostgreSQL for open alerts on the same `stop_execution_id`. If found: update `breach_probability`, `economic_risk_score`, `intervention_options` on the existing record. Do not create a duplicate. Emit `alert:updated` Socket.io event instead of `alert:new`.
5. **Escalation Timer Scheduling:** On alert creation, enqueue a Celery countdown task `tasks.escalate_alert(alert_id)` with `countdown=escalation_timeout_seconds` (default 300). If the dispatcher acknowledges the alert before the timer fires, the task is revoked (`celery.control.revoke(task_id)`). If the timer fires, send an SMS to the operator's dispatcher phone via Twilio.
6. **Alert Lifecycle Management:**
   - `open` → `acknowledged` (dispatcher taps the alert)
   - `acknowledged` → `resolved` (dispatcher marks resolved or stop is completed)
   - `open` → `auto_executing` (escalation timer fires and autonomous action begins)
   - `auto_executing` → `resolved` (autonomous action completed)
   - Any state → `dismissed` (dispatcher explicitly dismisses)
7. **Causal Explanation Generation:** When generating an alert, construct a structured causal explanation string: `"Route behind by 43 min. Causes: Traffic +25min (Gateway Motorway 11:32-11:54), 2 access failures (+10min), driver lunch break at 12:15 (+8min)."` This is assembled from structured data in the `route_executions` audit trail and displayed in the alert detail view.

### API Endpoints

```
GET /api/v1/alerts
  Auth: Dispatcher JWT
  Query: ?status=open&severity=high,critical&limit=50
  Response: Array of AlertEvent objects, ordered by economic_risk_score DESC

GET /api/v1/alerts/{alert_id}
  Response: Full AlertEvent with intervention_options array

PATCH /api/v1/alerts/{alert_id}/acknowledge
  Body: { dispatcher_id }
  Effect: Set status=acknowledged, record acknowledged_at, revoke escalation Celery task

PATCH /api/v1/alerts/{alert_id}/approve-intervention
  Body: { option_rank, dispatcher_id }
  Effect: Execute selected intervention via Celery task; set status=acknowledged; log dispatcher_action

PATCH /api/v1/alerts/{alert_id}/dismiss
  Body: { reason, dispatcher_id }
  Effect: Set status=dismissed; log dismissal reason

GET /api/v1/alerts/history
  Query: ?date=2025-01-15&client_id=...
  Response: Paginated alert history with resolution outcomes
```

### WebSocket Events

**Emitted to dispatcher clients (room: `operator:{operator_id}`):**
- `alert:new` → Full AlertEvent object
- `alert:updated` → `{ alert_id, breach_probability, economic_risk_score, minutes_to_breach }`
- `alert:acknowledged` → `{ alert_id, acknowledged_by, acknowledged_at }`
- `alert:auto_executed` → `{ alert_id, action_taken, stops_affected, reasoning }`
- `alert:resolved` → `{ alert_id, resolution_type, resolved_at }`

**Consumed from Kafka:**
- Topic: `alert-triggers` (consumer group: `alert-processor`)

### State Management

```
alert:open:{operator_id}          → Sorted set (alert_id → economic_risk_score)  No TTL
alert:celery_task:{alert_id}      → String (Celery task ID for revocation)        TTL: alert window
alert:acknowledged:{alert_id}     → Boolean                                       TTL: 3600s
```

### Dependencies

- Kafka consumer (`alert-triggers` topic)
- PostgreSQL (`alert_events`, `intervention_actions`, `route_executions`, `stop_executions`)
- Redis (alert state, SLA risk cache)
- Socket.io server (push to dispatcher)
- Celery (escalation timer tasks)
- Twilio (escalation SMS)
- `services/intervention_generator.py` (rules-based option generation)

---

## Module 5: Autonomous Cascade Orchestrator

### Purpose

When a driver drops mid-run (confirmed breakdown, accident, illness, or vehicle fault), immediately execute a full cascade assessment: calculate remaining stops, find available receiving drivers, generate redistribution options ranked by SLA economics, present to dispatcher for 1-click approval with a countdown fallback, and execute the redistribution (autonomously if dispatcher doesn't respond) including driver app updates and customer notifications.

### Key Functions

1. **Driver-Drop Detection:** Listen for three trigger types on Kafka `cascade-events` topic: (a) dispatcher-initiated `driver_drop` event via API, (b) stationary detection escalation (>15 min stationary at non-stop, no response on follow-up), (c) vehicle fault event from Teletrac Navman integration (optional).
2. **Remaining Stop Calculation:** Query `stop_executions` for the dropped driver's route where `status IN ('pending', 'en_route')`. For each stop, compute breach urgency score: `breach_probability × (1 + (minutes_to_window_close / 60))`. Sort by urgency score DESC to determine redistribution priority.
3. **Available Driver Discovery:** Query all active drivers in the operator's fleet with `status = 'active'` and `route_execution.status = 'active'`. For each candidate driver, compute: (a) proximity to each unassigned stop cluster (haversine distance), (b) remaining capacity (configured vehicle capacity minus current assigned stop count), (c) estimated time to first available redistribution point (ETA to next stop completion). Filter to drivers within 20 km radius and with >5 remaining capacity slots.
4. **Redistribution Options Generation:** Generate up to 3 redistribution options using a greedy insertion algorithm (`services/cascade_planner.py`): Variant A (minimise SLA breach count), Variant B (minimise total overtime cost), Variant C (minimise recipient ETA disruption). For each variant: assign unassigned stops to receiving drivers using nearest-insertion heuristic; re-optimise each receiving driver's remaining route; compute: `sla_stops_saved`, `estimated_overtime_hours`, `estimated_overtime_cost`, `recipient_average_eta_delta_minutes`.
5. **Dispatcher Approval Interface:** Emit `cascade:options_ready` Socket.io event with full `CascadeIncident` object. The dashboard renders a `CascadeApprovalModal` component with 3 option cards and a countdown timer (default 5 minutes, displayed as a progress bar).
6. **Auto-Execute on Non-Response:** Celery task `tasks.cascade_auto_execute(incident_id)` with `countdown=300`. If dispatcher doesn't call `POST /api/v1/cascade/{incident_id}/approve` within 5 minutes, the task fires and executes the top-ranked option (Variant A).
7. **Execution Steps (on approval or auto-execute):**
   a. Update `stop_executions` records: set `route_execution_id` to receiving driver's route, `stop_sequence` recalculated, status remains `pending`
   b. Call driver app push API for each receiving driver: `POST /internal/driver-app/{driver_id}/route-update` with new stop list
   c. For each reassigned stop with `recipient_phone` set: trigger `CustomerCommunicationAgent.send_update(stop_id, reason='driver_change', new_eta=...)` — queued as Celery task in `medium` priority
   d. Write `InterventionAction` record with `autonomous=True|False` and full `payload`
   e. Update `CascadeIncident` record: `resolved_at`, `auto_executed`, `dispatcher_response_seconds`
   f. Emit `cascade:resolved` Socket.io event to dispatcher
   g. Send confirmation SMS to dispatcher: "Cascade resolved: 24 stops redistributed across 3 drivers. 4 SLA windows preserved. Action: [manual/auto]."
8. **Post-Incident Audit:** Full audit trail in `intervention_actions` table. Cascade incident summary available in `GET /api/v1/cascade/{incident_id}/report`.

### API Endpoints

```
POST /api/v1/cascade/trigger
  Auth: Dispatcher JWT
  Body: { driver_id, route_id, trigger_reason: "manual|vehicle_fault|stationary_escalation" }
  Effect: Initiates cascade assessment; returns incident_id immediately
  Response: { incident_id, status: "assessing", estimated_options_ready_seconds: 30 }

GET /api/v1/cascade/{incident_id}
  Response: Full CascadeIncident with redistribution_options array

POST /api/v1/cascade/{incident_id}/approve
  Auth: Dispatcher JWT
  Body: { option_rank, dispatcher_id }
  Effect: Execute selected option; revoke auto-execute Celery task

POST /api/v1/cascade/{incident_id}/reject
  Body: { reason, dispatcher_id }
  Effect: Cancel auto-execute timer; alert remains open for manual handling

GET /api/v1/cascade/history
  Query: ?date_from=...&date_to=...
  Response: Paginated cascade incidents with outcomes
```

### WebSocket Events

**Emitted:**
- `cascade:incident_opened` → `{ incident_id, dropped_driver_id, affected_stop_count, at_risk_window_stop_count }`
- `cascade:options_ready` → Full `CascadeIncident` object with `redistribution_options`
- `cascade:executing` → `{ incident_id, option_rank, autonomous }`
- `cascade:resolved` → `{ incident_id, stops_saved, stops_breached, execution_ms }`

**Consumed from Kafka:**
- Topic: `cascade-events` (consumer group: `cascade-processor`)

### State Management

```
cascade:incident:{incident_id}:status    → String (assessing|options_ready|executing|resolved)  TTL: 3600s
cascade:celery_task:{incident_id}        → String (auto-execute task ID)                        TTL: 600s
cascade:available_drivers:{operator_id} → Sorted set (driver_id → proximity_score)             TTL: 60s
```

### Dependencies

- PostgreSQL (`route_executions`, `stop_executions`, `cascade_incidents`, `drivers`, `vehicles`)
- Redis
- Celery
- Socket.io server
- Twilio (dispatcher confirmation SMS)
- `services/cascade_planner.py` (greedy insertion algorithm)
- Driver app push API (internal)
- Customer Communication Agent (Module 7, triggered post-execution)
- Kafka consumer (`cascade-events` topic)

---

## Module 6: Live Dispatcher Dashboard

### Purpose

Provide the dispatcher with a single-screen operational command centre that displays real-time fleet positions, ranked active alerts with actionable intervention options, route progress across all active routes, portfolio-level SLA health, and the cascade approval flow. The dashboard must update in real time without manual refresh and must surface the most urgent issues without requiring the dispatcher to scan the entire map.

### Key Functions

1. **FleetMap Component:** Render all active driver positions on a Mapbox GL JS map. Marker colour encodes status: green (on schedule), amber (behind schedule), red (at risk / SLA breach probability >60%), grey (connection degraded). Click a marker to see: driver name, route progress %, current ETA vs scheduled, active alerts for this driver. Route line overlay shows planned route with completed stops (greyed out) and remaining stops (coloured by risk). Update marker positions on every `driver:position` Socket.io event.
2. **AlertPanel Component:** Display open alerts ordered by `economic_risk_score DESC`. Each alert card shows: severity badge, alert type, affected stop/delivery details, minutes to breach, economic risk amount, and up to 3 intervention option buttons. Unacknowledged alerts older than 2 minutes flash amber. Unacknowledged alerts older than 4 minutes (1 minute before auto-execute) display a red countdown timer. Alert cards are animated into the list on `alert:new` event.
3. **RouteProgressList Component:** Tabular view of all active routes. Columns: driver name, route progress bar (% complete), completed/total stops, current ETA completion vs planned, behind-by minutes, active alert count for route. Rows sortable by any column. Behind-schedule rows highlighted amber/red. Click row to filter FleetMap to that route.
4. **SLARiskGauge Component:** Portfolio-level SLA health indicator. Displays: total economic risk ($) currently live, percentage of active stops at risk, percentage of active stops critical, projected breach count if no intervention. Updated every 30 seconds via `sla:portfolio_health_updated` Socket.io event. A prominent dollar-amount "economic exposure" number at the top.
5. **DriverStatusGrid Component:** Compact grid of all 35 drivers with status icons. Statuses: active (moving), at stop, stationary (alert), offline, off-duty. Click driver to jump to their route in FleetMap.
6. **CascadeApprovalModal Component:** Rendered as a high-priority modal overlay when `cascade:options_ready` is received. Three option cards side-by-side. Each card: option type label, stops reassigned, SLA windows preserved, estimated overtime hours and cost, affected customer count, "Approve" button. Countdown timer progress bar across the top. "Auto-execute in X:XX" label. Dismiss requires confirmation dialog (auto-execute proceeds).
7. **Real-Time KPI Strip:** Fixed header strip across top of dashboard. Metrics: routes active, stops completed today (with completion rate vs plan), on-time delivery % (rolling 4-hour window), open alerts count (red badge), economic risk live.
8. **WebSocket Connection Management:** Socket.io client subscribes to `operator:{operator_id}` room on connection. On reconnection, calls `GET /api/v1/fleet/positions` and `GET /api/v1/alerts?status=open` to resync state. Reconnection UI: toast notification "Reconnecting..." with spinner. On reconnect success: toast "Connected — data refreshed".

### React Component Tree

```
App
└── DashboardLayout
    ├── KPIStrip                         (real-time counters)
    ├── AlertPanel                       (ranked alerts, action buttons)
    │   └── AlertCard (×N)
    │       └── InterventionOptionRow (×3)
    ├── FleetMap (Mapbox GL JS)
    │   ├── DriverMarker (×35)
    │   ├── RouteOverlay (×N)
    │   └── StopMarker (×N)
    ├── RouteProgressList
    │   └── RouteProgressRow (×N)
    ├── SLARiskGauge
    ├── DriverStatusGrid
    │   └── DriverStatusCell (×35)
    └── CascadeApprovalModal (conditional)
        └── CascadeOptionCard (×3)
```

### WebSocket Events Consumed

- `driver:position` → update FleetMap marker
- `route:progress` → update RouteProgressList row
- `alert:new` → insert AlertCard into AlertPanel
- `alert:updated` → update existing AlertCard in place
- `alert:acknowledged` → mark AlertCard as acknowledged
- `alert:auto_executed` → show auto-execute toast, update AlertCard state
- `alert:resolved` → remove AlertCard (animate out)
- `sla:portfolio_health_updated` → update SLARiskGauge
- `cascade:incident_opened` → show "cascade incoming" banner on AlertPanel
- `cascade:options_ready` → render CascadeApprovalModal
- `cascade:resolved` → close CascadeApprovalModal, show success toast

### State Management (Zustand stores)

```typescript
// store/fleet.ts
interface FleetStore {
  driverPositions: Map<string, DriverPosition>;
  routeExecutions: Map<string, RouteExecution>;
  updateDriverPosition: (pos: DriverPosition) => void;
}

// store/alerts.ts
interface AlertStore {
  openAlerts: AlertEvent[];           // sorted by economic_risk_score desc
  acknowledgedAlerts: AlertEvent[];
  addAlert: (alert: AlertEvent) => void;
  updateAlert: (id: string, patch: Partial<AlertEvent>) => void;
  removeAlert: (id: string) => void;
}

// store/cascade.ts
interface CascadeStore {
  activeIncident: CascadeIncident | null;
  incidentStatus: 'idle' | 'assessing' | 'options_ready' | 'executing' | 'resolved';
  setIncident: (incident: CascadeIncident) => void;
  clearIncident: () => void;
}

// store/sla.ts
interface SLAStore {
  portfolioHealth: PortfolioSLAHealth;
  updateHealth: (health: PortfolioSLAHealth) => void;
}
```

### Dependencies

- React 18 + TypeScript + Vite
- Mapbox GL JS (env var: `VITE_MAPBOX_ACCESS_TOKEN`)
- Socket.io Client 4.x
- TanStack Query 5 (initial data load, REST queries)
- Zustand 4 (real-time state)
- shadcn/ui + Tailwind CSS
- Recharts (SLA gauges, completion trend)

---

## Module 7: Proactive Customer Communication Agent

### Purpose

Detect at-risk deliveries before the driver arrives and fails, contact the recipient via SMS with a personalised LLM-generated message offering actionable alternatives (confirm home, safe drop, reschedule), parse inbound replies, update stop execution records accordingly, and eliminate the would-be complaint before it enters the pipeline.

### Key Functions

1. **At-Risk Delivery Detection:** Subscribe to `sla:stop_risk_updated` Socket.io events from SLA Breach Evaluator. When `breach_probability > 0.40` AND `customer_contacted = false` AND `recipient_phone IS NOT NULL` AND stop is not yet `arrived`: enqueue `tasks.send_proactive_contact(stop_execution_id)` in Celery `medium` priority queue.
2. **Contact Eligibility Check:** Before sending, verify: (a) customer has not opted out (check `customer_opt_outs` table), (b) stop has not already been contacted in the last 2 hours (idempotency check via Redis `contact_sent:{stop_id}`), (c) stop is still in `pending` or `en_route` status, (d) recipient phone number is valid E.164 format.
3. **LLM Message Generation:** Call OpenAI GPT-4o-mini (or Claude Haiku) with a structured prompt including: recipient first name, delivery description (sanitised), predicted arrival window (P10–P90), three option descriptions. Prompt template in `prompts/proactive_contact.jinja2`. LLM output is validated against a Pydantic schema before sending. Maximum message length: 160 characters (single SMS). If LLM output exceeds 160 characters, truncate to template fallback.
4. **SMS Dispatch:** Send via Twilio Messaging API. Log `CustomerContact` record with `twilio_message_sid`, `delivery_status`. Set Redis `contact_sent:{stop_id}` with TTL 7200s (idempotency window). Update `stop_executions.customer_contacted = true`.
5. **Inbound Reply Processing:** Twilio inbound webhook fires on reply. FastAPI endpoint `POST /api/v1/webhooks/twilio-inbound` validates signature, extracts message body, produces event to Kafka `customer-contact-queue`.
6. **Intent Classification:** Celery consumer processes `customer-contact-queue`. Send reply text to LLM for intent classification: one of `confirm_home`, `safe_drop` (+ safe drop location), `reschedule` (+ preference if stated), `opt_out`, `unknown`. Structured output via Pydantic.
7. **Intent Execution:**
   - `confirm_home` → update `stop_executions.customer_response = 'confirmed_home'`; no route change; log customer contact; emit `stop:customer_confirmed` Socket.io event to dispatcher
   - `safe_drop` → update `stop_executions.customer_response = 'safe_drop'`, store safe drop note; push note to driver app via `route:stop_updated` event
   - `reschedule` → update stop status to `rescheduled`; remove from driver's active route; create a rescheduled delivery record for tomorrow's route planning; send confirmation SMS to customer; emit `stop:rescheduled` to dispatcher
   - `opt_out` → add to `customer_opt_outs` table; send STOP confirmation SMS; do not contact again
   - `unknown` → log and do not act; if 2 unknown responses in a row, flag for dispatcher review
8. **Follow-up for Reschedule Preference:** If intent is `reschedule` but no window preference stated, send a follow-up SMS: "Would you like AM (8am–12pm) or PM (12pm–5pm) tomorrow?" Parse reply as `am` or `pm`.
9. **Opt-Out Compliance:** All outbound messages include "Reply STOP to opt out." Opt-out database is checked at every contact attempt. Opted-out recipients are never contacted again for any delivery.

### API Endpoints

```
POST /api/v1/webhooks/twilio-inbound
  Auth: Twilio request signature validation (no user auth)
  Body: Twilio webhook POST fields (From, Body, MessageSid, etc.)
  Effect: Produce to Kafka customer-contact-queue; return TwiML response
  Response: <Response></Response> (empty TwiML, no auto-reply from Twilio)
  Latency target: <200ms (Twilio requires fast response to prevent retry)

GET /api/v1/stops/{stop_id}/customer-contacts
  Auth: Dispatcher JWT
  Response: Array of CustomerContact records for this stop (outbound + inbound)

POST /api/v1/stops/{stop_id}/contact-manual
  Auth: Dispatcher JWT
  Body: { message_override?: string }
  Effect: Trigger proactive contact immediately (bypass probability threshold)

GET /api/v1/operators/{operator_id}/contact-stats
  Response: {
    outbound_today: integer,
    confirmed_home: integer,
    safe_drop: integer,
    rescheduled: integer,
    no_response: integer,
    response_rate_pct: float,
    ftd_reduction_estimated_pct: float
  }
```

### WebSocket Events

**Emitted to dispatcher room:**
- `stop:customer_confirmed` → `{ stop_id, delivery_id, response_type: 'confirmed_home' }`
- `stop:safe_drop_set` → `{ stop_id, safe_drop_notes }`
- `stop:rescheduled` → `{ stop_id, delivery_id, rescheduled_to_date }`

### State Management

```
contact_sent:{stop_id}              → Boolean               TTL: 7200s (idempotency)
contact_reply:{stop_id}             → String (last reply)   TTL: 3600s
contact_pending_followup:{stop_id}  → Boolean               TTL: 1800s (await window preference)
```

### Kafka Integration

- **Produces to:** `customer-contact-queue` (from inbound webhook handler)
- **Consumes from:** `customer-contact-queue` (consumer group: `contact-processor`)

### Dependencies

- Twilio Python SDK (`twilio` 8.x; env vars: `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_NUMBER`)
- OpenAI Python SDK (`openai` 1.x; env var: `OPENAI_API_KEY`) or Anthropic SDK (`anthropic`; env var: `ANTHROPIC_API_KEY`)
- Jinja2 (prompt templates: `prompts/proactive_contact.jinja2`, `prompts/classify_intent.jinja2`, `prompts/reschedule_followup.jinja2`)
- Pydantic (LLM output validation schemas)
- Redis (idempotency, reply state)
- PostgreSQL (`customer_contacts`, `stop_executions`, `customer_opt_outs`)
- Kafka consumer (`customer-contact-queue`)
- Celery (`medium` priority queue)
- Socket.io server (dispatcher notifications)
