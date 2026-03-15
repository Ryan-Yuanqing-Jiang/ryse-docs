# Challenge 3: Live Operational Dashboard & Exception Alerting — Execution Blueprint

---

## Phase 1: Foundation (Weeks 1–3)

### Goals

Stand up the complete GPS ingestion pipeline from driver app to TimescaleDB and Redis. Deliver a live map dashboard that shows real driver positions updating in real time. Validate the end-to-end data flow before any intelligence layer is built on top of it. Infrastructure is provisioned as code; no manual console clicking.

### Atomic Tasks

**Infrastructure Setup (Week 1, Days 1–3)**

1. Initialise Terraform workspace at `infra/terraform/`. Create modules: `vpc`, `ecs`, `rds`, `elasticache`, `msk`, `alb`.
2. Provision VPC with 3 public subnets (ALB) and 3 private subnets (ECS, RDS, ElastiCache, MSK) across `ap-southeast-2a/b/c`.
3. Provision AWS MSK cluster: 3 brokers, `kafka.t3.small` instance type, MSK Serverless not recommended here (latency variance), `kafka.2.8.0`. Set `auto.create.topics.enable=false`. Create topics manually:
   - `driver-locations` (10 partitions, replication factor 3, retention 24h)
   - `stop-events` (5 partitions, RF 3, retention 48h)
   - `alert-triggers` (5 partitions, RF 3, retention 24h)
   - `cascade-events` (3 partitions, RF 3, retention 48h)
   - `customer-contact-queue` (3 partitions, RF 3, retention 24h)
4. Provision AWS RDS PostgreSQL 16 instance: `db.t3.medium`, Multi-AZ enabled, storage 100GB gp3. Enable Performance Insights.
5. Enable TimescaleDB extension on RDS instance: connect via `psql`, run `CREATE EXTENSION IF NOT EXISTS timescaledb;`. Run migration `migrations/001_driver_positions_hypertable.sql` (create table + `create_hypertable` call + continuous aggregates).
6. Provision Redis ElastiCache: `cache.t3.medium`, cluster mode disabled (single node for Phase 1), enable in-transit and at-rest encryption.
7. Set up AWS Secrets Manager entries: `odocs/dev/db-password`, `odocs/dev/redis-url`, `odocs/dev/jwt-secret`, `odocs/dev/kafka-bootstrap-servers`.
8. Create ECR repositories: `odocs/api-gateway`, `odocs/eta-service`, `odocs/celery-worker`, `odocs/frontend`.
9. Set up GitHub Actions workflow `.github/workflows/deploy-dev.yml`: on push to `main`, build Docker images, push to ECR, trigger ECS rolling deployment.

**Database Migrations (Week 1, Days 3–5)**

10. Install `alembic` in `services/api-gateway/`. Set `SQLALCHEMY_DATABASE_URL` from Secrets Manager at runtime.
11. Write `migrations/001_core_tables.py`: create tables `operators`, `drivers`, `vehicles`, `clients`, `route_plans`, `route_executions`, `stop_executions` with all fields from the data model in `02-system-architecture.md`. Include indexes.
12. Write `migrations/002_driver_positions_hypertable.py`: create `driver_positions` table, call `create_hypertable`, create continuous aggregate `driver_positions_1min`, add compression policy (compress chunks older than 7 days).
13. Write `migrations/003_retention_policy.py`: add TimescaleDB retention policy — drop raw chunks older than 30 days; retain aggregated view for 1 year.

**GPS Ingest Service (Week 1, Days 4–5 through Week 2, Day 2)**

14. Create FastAPI app at `services/api-gateway/main.py`. Use `uvicorn` with `--workers 4`. Configure `asyncpg` connection pool (min 5, max 20). Configure `aiokafka` producer.
15. Create Pydantic schema `schemas/gps.py`: `GPSPingRequest` with fields `driver_id` (UUID), `route_id` (UUID, optional), `lat` (float, ge=-90, le=90), `lng` (float, ge=-180, le=180), `speed_kmh` (float, ge=0, le=200), `heading_degrees` (int, ge=0, le=359), `accuracy_metres` (float), `timestamp_utc` (datetime), `battery_pct` (int, ge=0, le=100).
16. Create router `routers/gps.py`. Implement `POST /api/v1/gps`:
    - Validate JWT (middleware `middleware/auth.py`, decode with `python-jose`, verify `driver_id` claim matches body)
    - Write to TimescaleDB `driver_positions` table via asyncpg (non-blocking, fire-and-forget with error capture)
    - Write to Redis hash `driver:{driver_id}:position` via `aioredis` (pipeline write, TTL 30s)
    - Produce to Kafka `driver-locations` topic via `aiokafka` (non-blocking, confirm on background task)
    - Return `{"status": "accepted"}` HTTP 202 immediately
17. Create `services/kafka_outbox.py`: Celery periodic task that polls `kafka_outbox` PostgreSQL table (created in migration `004_kafka_outbox.py`) and retries any failed Kafka produces. Run every 30 seconds via Celery Beat.
18. Write `routers/gps.py` endpoint `GET /api/v1/fleet/positions`: fetch all keys matching `driver:*:position` pattern in Redis (use `SCAN`, not `KEYS`). Filter by `operator_id` from JWT claim. Return array.
19. Write unit tests `tests/test_gps_ingest.py`: mock Kafka producer, mock Redis, mock DB. Test: valid payload accepted, invalid coordinates rejected, expired JWT rejected, Kafka fallback to outbox on produce failure.

**Stationary Detection (Week 2, Days 1–3)**

20. Create Kafka consumer `consumers/gps_consumer.py` using `confluent-kafka-python`. Consumer group: `stationary-detector`. Topic: `driver-locations`.
21. Implement stationary detection logic in `services/stationary_detector.py`:
    - On each GPS event: read `driver:{driver_id}:stationary` from Redis
    - If `speed_kmh < 3.0` AND position delta from last ping < 50 metres: increment stationary duration counter in Redis hash, TTL reset to 600s
    - If stationary duration > 600 seconds (10 minutes): check if driver is within 100m geofence of any stop in their route (fetch stop coordinates from Redis `route:{route_id}:stops` set)
    - If not at a stop: produce `driver_stationary` event to Kafka `alert-triggers` topic
    - If speed > 3.0: delete `driver:{driver_id}:stationary` key (reset)
22. Write `services/geofence.py`: `is_within_geofence(lat, lng, stops: List[StopCoordinate], radius_metres=100) -> bool` using Haversine formula. Pure function, no I/O.

**WebSocket Server (Week 2, Days 2–4)**

23. Install `python-socketio[asyncio_client]` and `uvicorn`. Create `services/websocket_server.py` as an ASGI Socket.io application mounted on the FastAPI app via `socketio.ASGIApp`.
24. Implement Socket.io namespaces: `/dispatch` for dispatcher clients. On connect: validate JWT from `auth.token` handshake parameter. Join room `operator:{operator_id}` on successful validation.
25. Create Kafka consumer `consumers/position_broadcaster.py`: consumer group `position-broadcaster`, topic `driver-locations`. On each message: emit Socket.io event `driver:position` to room `operator:{operator_id}` with `{ driver_id, lat, lng, speed_kmh, heading, recorded_at }`.
26. Configure ALB sticky sessions for WebSocket connections (enable stickiness on target group, `lb_cookie` duration 3600s). Document this in `infra/README.md`.

**Live Map Dashboard — Phase 1 (Week 2, Day 4 through Week 3)**

27. Create React app at `frontend/` using Vite + TypeScript: `pnpm create vite frontend --template react-ts`. Install dependencies: `mapbox-gl`, `socket.io-client`, `@tanstack/react-query`, `zustand`, `tailwindcss`, `shadcn-ui`.
28. Set environment variables: `VITE_API_BASE_URL`, `VITE_SOCKET_URL`, `VITE_MAPBOX_ACCESS_TOKEN` (loaded from `.env.local` in dev, injected at build time in CI).
29. Create `store/fleet.ts` Zustand store with `driverPositions: Map<string, DriverPosition>` and `updateDriverPosition` action.
30. Create `hooks/useSocket.ts`: initialise Socket.io client, connect to `/dispatch` namespace with JWT from local auth context, handle reconnection, expose `socket` instance.
31. Create `hooks/useFleetPositions.ts`: subscribe to `driver:position` Socket.io events; call `store.updateDriverPosition` on each event.
32. Create component `components/FleetMap.tsx`: Mapbox GL JS map centred on Brisbane (-27.47, 153.02), zoom 11. For each entry in `fleet.driverPositions`, render a `Marker` component. Marker colour: green (speed > 2), amber (stationary > 5 min), grey (stale > 30s). On marker click: open popup with `{ driver_id, speed_kmh, recorded_at }`.
33. Create `components/DriverStatusGrid.tsx`: 35 cells in a CSS grid. Each cell: driver name, status dot, last update time delta ("12s ago"). Data sourced from `fleet.driverPositions` Zustand store.
34. Create `pages/Dashboard.tsx`: layout with `FleetMap` (full left panel), `DriverStatusGrid` (right panel top). Hardcode layout for now; responsive sizing in Phase 4.
35. Create `components/KPIStrip.tsx`: static placeholders for Phase 1 — "Routes Active: —", "Stops Completed: —". Wire to API in Phase 2.
36. Write `frontend/cypress/e2e/live_map.cy.ts`: integration test that connects to dev environment, verifies driver markers appear and update position within 10 seconds.

### Key Technical Decisions

- **TimescaleDB over plain PostgreSQL for GPS data:** At 5-second intervals, 35 drivers, and 10-hour operating days, the `driver_positions` table accumulates approximately 252,000 rows/day. TimescaleDB's automatic chunking and compression makes this manageable without DBA overhead. The continuous aggregate enables fast 1-minute rollup queries for the map trail feature in Phase 4.
- **Socket.io over raw WebSocket:** Automatic reconnection, namespace-based routing, and room management are non-trivial to build on raw WebSocket. At the scale of 35 drivers and 5 dispatcher clients, Socket.io's overhead is negligible and the operational simplicity is worth it.
- **aiokafka over confluent-kafka-python for the ingest service:** The GPS ingest endpoint must be async (non-blocking) to maintain <50ms response. `aiokafka` is the correct async Kafka client for asyncio FastAPI. `confluent-kafka-python` (librdkafka wrapper) is used for consumers, which run as separate processes and benefit from its superior throughput and configuration options.

### Definition of Done

- [ ] GPS ping sent from a test driver app script reaches TimescaleDB `driver_positions` table within 200ms
- [ ] Driver position visible on live map dashboard within 5 seconds of GPS ping
- [ ] 35 simulated drivers (load test script `scripts/load_test_gps.py`) posting at 5-second intervals show no latency degradation in API response times (p99 < 50ms)
- [ ] Stationary driver triggers `alert-triggers` Kafka message after 10 minutes
- [ ] Dashboard reconnects automatically after simulated WebSocket drop; positions resync within 5 seconds
- [ ] All Terraform resources provisioned via `terraform apply` from a clean state with no errors
- [ ] GitHub Actions pipeline deploys to dev on push to `main`

### Risks and Mitigations

| Risk | Probability | Mitigation |
|---|---|---|
| TimescaleDB extension not available on RDS version | Low | Verify RDS PostgreSQL 16 supports timescaledb 2.x before provisioning; fallback is self-managed EC2 PostgreSQL with timescaledb installed |
| ALB sticky sessions insufficient for Socket.io at scale | Low (5 dispatchers) | Monitor connection drops in Phase 1; if issues arise, switch to Redis adapter for Socket.io (`python-socketio` supports this natively) |
| GPS simulator doesn't match real driver app behaviour | Medium | Build GPS simulator (`scripts/load_test_gps.py`) to mirror the exact payload schema and interval of the real driver app; align with driver app developers during Week 1 |
| Kafka MSK bootstrap latency in ap-southeast-2 | Low | Run latency test from ECS task to MSK brokers in Week 1; if >10ms, switch to MSK Serverless or co-locate ECS tasks in same AZ as MSK brokers |

---

## Phase 2: Intelligence (Weeks 4–6)

### Goals

Build the ETA Prediction Service (GBDT model, feature assembly, inference API). Build the SLA Breach Evaluator. Build the Alert Decision Engine with basic alert taxonomy and dispatcher push. Wire alerts into the dashboard AlertPanel. The dispatcher can now see ranked alerts with breach probabilities.

### Atomic Tasks

**ETA Prediction Service (Week 4)**

37. Create new ECS service `services/eta-service/`. FastAPI app `main.py`. Separate ECR repo `odocs/eta-service`. Dockerfile: Python 3.12-slim, install `lightgbm`, `fastapi`, `uvicorn`, `redis`, `boto3`, `pydantic`.
38. Create `models/eta_model.py`: `ETAModel` class. On `__init__`: download model artifact from S3 `s3://odocs-models/eta-gbdt/latest.lgb` via `boto3`. Load with `lgb.Booster(model_file=...)`. Store as class attribute. Expose `predict(feature_vector: list[float]) -> dict`.
39. Create Pydantic schema `schemas/prediction.py`: `ETAPredictionRequest` (stop_execution_id, driver_id, feature_vector as list of 14 floats, committed_window_end), `ETAPredictionResponse` (all fields from data model).
40. Implement inference endpoint `POST /internal/predict-eta`:
    - Validate input
    - Call `ETAModel.predict(feature_vector)` — returns raw LightGBM prediction (median) and quantile predictions (P10: q=0.1, P90: q=0.9 — requires model trained with quantile regression or use `predict_proba` with two quantile models)
    - Convert raw duration seconds to ISO8601 timestamps based on current time
    - Write result to Redis `eta:{stop_id}:prediction` (JSON, TTL 30s)
    - Write record to `eta_predictions` PostgreSQL table (async, non-blocking)
    - Return response
41. Create `services/feature_assembler.py`: `assemble_feature_vector(driver_id, stop_id, current_lat, current_lng, current_speed) -> list[float]`. Fetches: route state from Redis `route:{route_id}:state`, traffic data from Redis `traffic:{segment_hash}:travel` or Google Maps API, stop duration averages from Redis `zone_stats:{suburb_code}`, driver stats from Redis `driver_stats:{driver_id}`. Returns 14-element float list matching feature order in `02-system-architecture.md`.
42. Create `services/traffic_fetcher.py`: `get_traffic_multiplier(origin_lat, origin_lng, dest_lat, dest_lng) -> TrafficData`. Checks Redis cache first. On cache miss: call Google Maps Routes API (`POST https://routes.googleapis.com/directions/v2:computeRoutes`), parse `durationInTraffic`, compute multiplier vs baseline duration, write to Redis with 180s TTL.
43. Create `consumers/eta_recalc_consumer.py`: Kafka consumer group `eta-recalc`, topic `driver-locations`. For each GPS event: (a) fetch driver's active route and remaining pending stops from Redis, (b) for each pending stop call `feature_assembler.assemble_feature_vector()`, (c) call `POST /internal/predict-eta` via internal HTTP (or direct function call if same process), (d) after each prediction: evaluate SLA breach (see task 44).
44. Create `scripts/train_eta_model.py`: LightGBM training script. Features: columns defined in `02-system-architecture.md` feature table. Target: actual stop arrival time − predicted time (train on `eta_predictions` + `stop_executions` join). Use 90-day rolling training window. Trained model uploaded to S3 `s3://odocs-models/eta-gbdt/{version}/model.lgb`. Initial model: train on synthetic data generated by `scripts/generate_synthetic_training_data.py` — generate 10,000 stop records with realistic distributions for Brisbane last-mile delivery.

**SLA Breach Evaluator (Week 4–5)**

45. Create `services/sla_evaluator.py`: `evaluate_breach_risk(stop_execution_id, prediction: ETAPredictionResponse, sla_config: SLAConfig) -> SLABreachRisk`.
    - Load committed window from `stop_executions` (cached in Redis `stop:{stop_id}:window`)
    - Compute breach probability using P10/P90 lognormal approximation: `mu = median_arrival`, `sigma = (P90 - P10) / (2 * 1.645)`. `breach_prob = 1 - norm.cdf(window_end, loc=mu, scale=sigma)`.
    - Apply soft window buffer: `effective_window_end = window_end + timedelta(minutes=sla_config.soft_buffer_minutes)`
    - Compute `economic_risk_score = breach_prob × sla_config.penalty_amount`
    - Write to Redis `sla:breach_prob:{stop_id}` (TTL 60s) and `sla:economic_risk:{stop_id}` (TTL 60s)
    - If `breach_prob > sla_config.breach_probability_threshold` AND stop not in Redis `alert:open_stop_ids:{operator_id}` set: produce to Kafka `alert-triggers`
46. Create migration `migrations/005_sla_configs.py`: create `sla_configs` table with `client_id`, `window_type`, `soft_buffer_minutes`, `breach_probability_threshold`, `economic_risk_threshold`, `delivery_category_penalties` (JSONB). Seed with one test client SLA config.
47. Create `services/sla_config_loader.py`: loads SLA configs from PostgreSQL into Redis `sla_config:{client_id}` on operator login. Celery Beat task `tasks/refresh_sla_configs.py` refreshes every 5 minutes.
48. Add endpoint `GET /api/v1/operators/{operator_id}/sla-health` (returns portfolio aggregation from Redis `sla:portfolio:{operator_id}` hash). Write Celery periodic task `tasks/aggregate_sla_health.py` (runs every 30 seconds): scans all `sla:breach_prob:{stop_id}` keys for the operator, computes aggregates, writes to Redis `sla:portfolio:{operator_id}`.

**Alert Decision Engine (Week 5)**

49. Create `consumers/alert_consumer.py`: Kafka consumer group `alert-processor`, topic `alert-triggers`.
50. Create `services/alert_classifier.py`: `classify_alert(trigger: KafkaAlertTrigger) -> AlertType`. Maps trigger payload to enum `AlertType`: `route_behind_schedule`, `sla_at_risk`, `driver_stopped`, `cascade_risk`, `driver_drop`.
51. Create `services/intervention_generator.py`: `generate_options(alert: AlertEvent) -> List[InterventionOption]`. Rules-based generation for Phase 2 (no ML yet):
    - `sla_at_risk` + `minutes_to_breach > 30`: suggest `reroute_driver` (resequence stops for speed) and `contact_customer_proactive`
    - `sla_at_risk` + `minutes_to_breach < 30`: suggest `reassign_stop` (find nearest driver) and `contact_customer_proactive`
    - `driver_stopped`: suggest `contact_driver` (dispatcher call), `cascade_risk_assessment` if stationary > 15 min
    - `route_behind_schedule`: suggest `reroute_driver`, notify operations
52. Create `services/alert_manager.py`: `process_alert_trigger(trigger)`:
    - Check deduplication: query PostgreSQL for open alert on same `stop_execution_id`; if found, call `update_alert()` instead of `create_alert()`
    - Write `AlertEvent` to PostgreSQL
    - Add stop to Redis `alert:open_stop_ids:{operator_id}` set
    - Emit `alert:new` or `alert:updated` Socket.io event to room `operator:{operator_id}`
    - Schedule Celery countdown task `tasks.escalate_alert(alert_id, countdown=300)`
    - Store Celery task ID in Redis `alert:celery_task:{alert_id}` (TTL 360s)
53. Create `tasks/escalate_alert.py`: Celery task. On fire: check if alert still open and unacknowledged. If yes: send Twilio SMS to operator dispatcher phone. Set `alert_events.escalation_sms_sent = true`.
54. Add API endpoints: `PATCH /api/v1/alerts/{alert_id}/acknowledge`, `PATCH /api/v1/alerts/{alert_id}/approve-intervention`, `PATCH /api/v1/alerts/{alert_id}/dismiss`, `GET /api/v1/alerts`. See Module 4 spec for request/response schemas.

**Dashboard AlertPanel (Week 5–6)**

55. Create `store/alerts.ts` Zustand store (schema in Module 6 spec).
56. Create `hooks/useAlerts.ts`: initial load via `GET /api/v1/alerts?status=open` (React Query); subscribe to Socket.io events `alert:new`, `alert:updated`, `alert:acknowledged`, `alert:resolved`. Update Zustand store on each event.
57. Create `components/AlertCard.tsx`: renders one `AlertEvent`. Fields displayed: severity badge (colour-coded), alert type label, stop address, delivery window, `minutes_to_breach` countdown, `economic_risk_score` (formatted as $X.XX), causal explanation text, intervention option buttons (up to 3). Unacknowledged + age > 120s: amber border pulse animation. Age > 240s: red border pulse + countdown timer to auto-execute.
58. Create `components/InterventionOptionRow.tsx`: renders one `InterventionOption`. Fields: option type label, estimated time saving, estimated cost, SLA stops saved, "Approve" button. On Approve click: call `PATCH /api/v1/alerts/{alert_id}/approve-intervention` with `option_rank`; optimistic UI update.
59. Create `components/AlertPanel.tsx`: renders sorted list of `AlertCard` components from `alerts.openAlerts`. Sort: `economic_risk_score DESC`. Empty state: "No active alerts" with green check. Panel is scrollable with sticky header showing total open alert count and total economic risk.
60. Update `pages/Dashboard.tsx`: add `AlertPanel` to right panel below `DriverStatusGrid`. Add `SLARiskGauge` component (stub for now — just shows `portfolioHealth.total_economic_risk_dollars`).
61. Write integration tests `tests/integration/test_alert_pipeline.py`: simulate GPS ping → breach evaluator → alert trigger → alert created in DB → Socket.io event received. Use pytest-asyncio + fakeredis + TestcontainersSQL.

### Key Technical Decisions

- **LightGBM over XGBoost:** LightGBM is approximately 40% faster at inference for decision tree models of comparable accuracy. At 25,200 inferences/hour, this matters. LightGBM also has better quantile regression support via `alpha` parameter, which is needed for P10/P90 confidence intervals.
- **Synthetic training data for Phase 2:** Real operator historical data is not available at build time. Synthetic data with realistic Brisbane last-mile distributions (stop durations, traffic patterns, zone types) is sufficient to produce a working model that can be retrained on real data after 2–4 weeks of operation.
- **Lognormal distribution for breach probability:** Delivery time distributions are right-skewed (there are occasional very long delays but a hard floor at the predicted minimum). Lognormal provides a better fit than Gaussian, especially for short time windows.

### Definition of Done

- [ ] ETA prediction service responds to `POST /internal/predict-eta` with P10/P90/median within 200ms p99
- [ ] SLA breach evaluator correctly flags a stop at >60% breach probability and produces Kafka alert trigger
- [ ] Alert appears in dispatcher dashboard AlertPanel within 2 seconds of Kafka trigger
- [ ] Alert panel is sorted by `economic_risk_score DESC`
- [ ] Escalation SMS is sent to dispatcher phone if alert is unacknowledged for 5 minutes (tested in staging with real Twilio sandbox)
- [ ] Dispatcher can acknowledge an alert and the escalation task is cancelled (no SMS sent)
- [ ] All 14 features are correctly assembled from real-time sources (unit test for `feature_assembler.py` with fixture data)

### Risks and Mitigations

| Risk | Probability | Mitigation |
|---|---|---|
| Synthetic training data produces poor model accuracy on real data | High | Define model acceptance criteria before deployment: MAE < 8 minutes on validation set. Instrument `eta_predictions` table from day 1 to enable rapid retraining on real data after 2 weeks |
| Google Maps API rate limiting at full fleet scale | Low | Cache aggressively (3-minute TTL per segment). Implement exponential backoff. Monitor API quota in Datadog. Set up a backup traffic source (HERE Maps API) as failover |
| Kafka consumer lag accumulating under high load | Medium | Monitor consumer group lag in Datadog (MSK metric: `EstimatedTimeLag`). Alert PagerDuty if lag > 10s. Scale consumer containers horizontally — `confluent-kafka-python` consumer groups handle this automatically |
| Lognormal approximation produces unreliable breach probability near window boundaries | Medium | Add guardrails: clamp breach probability to [0.02, 0.98] range; document assumption; validate against actual breach outcomes weekly using `scripts/validate_breach_predictions.py` |

---

## Phase 3: Autonomous Action (Weeks 7–9)

### Goals

Build the Cascade Orchestrator. Implement the dispatcher 1-click approval flow with countdown timer. Implement auto-execute fallback. Push route updates to driver apps. Validate the full autonomous action pipeline end-to-end in staging.

### Atomic Tasks

**Cascade Orchestrator Core (Week 7)**

62. Create migration `migrations/006_cascade_incidents.py`: create `cascade_incidents` table (schema from data model).
63. Create Kafka consumer `consumers/cascade_consumer.py`: consumer group `cascade-processor`, topic `cascade-events`.
64. Create `services/cascade_detector.py`: subscribes to `alert:open` events (internal). If `driver_stationary` alert reaches 15 minutes without acknowledgement: produce `cascade_risk` event to `cascade-events` Kafka topic. API endpoint `POST /api/v1/cascade/trigger` also produces to this topic for manual dispatcher-initiated cascades.
65. Create `services/cascade_planner.py`: `plan_redistribution(incident: CascadeIncident) -> List[RedistributionOption]`.
    - Input: `dropped_driver_id`, `operator_id`
    - Step 1: Query `stop_executions` for remaining stops — fetch all where `route_execution_id = dropped_route_id AND status IN ('pending', 'en_route')`. Sort by `breach_urgency_score DESC` (= `breach_probability × (1 + 1/(max(minutes_to_window_close, 1)/60))`).
    - Step 2: Query available drivers — `SELECT d.id, re.id, re.total_stops, re.completed_stops FROM route_executions re JOIN drivers d ON ... WHERE re.status = 'active' AND re.date = today AND d.id != dropped_driver_id`. Compute remaining capacity = `vehicle_capacity - (total_stops - completed_stops)`. Compute proximity = haversine distance from driver's current GPS position to centroid of unassigned stops cluster.
    - Step 3: Filter to drivers within 20km AND remaining capacity > 5.
    - Step 4: Generate 3 redistribution variants using `_greedy_insert()` private method:
      - Variant A: minimise total `breach_probability × penalty_amount` across all reassigned stops
      - Variant B: minimise total overtime cost (estimated extra distance × vehicle cost per km × driver hourly rate)
      - Variant C: minimise average customer ETA delta (absolute deviation from original ETA)
    - Return top 3 options with metadata.
66. Create `services/greedy_insert.py`: `greedy_insert(unassigned_stops, available_drivers, objective_fn) -> StopAssignment`. Nearest-insertion heuristic: for each unassigned stop (in urgency order), find the receiving driver for whom inserting this stop minimises `objective_fn`. Insert stop into driver's route using cheapest insertion position. O(stops × drivers × stops_per_driver) — acceptable for max 24 stops across 5 candidate drivers.
67. Create Celery task `tasks/cascade_auto_execute.py`: countdown task scheduled for `incident.escalation_timeout_seconds` (default 300). On fire: if `cascade_incidents.selected_option_rank IS NULL` (dispatcher hasn't approved): execute Variant A automatically. Set `auto_executed = True`, `dispatcher_response_seconds = NULL`.
68. Create `services/cascade_executor.py`: `execute_redistribution(incident_id, option_rank, autonomous=False)`:
    - Update `stop_executions` records: set `route_execution_id` to receiving driver's route, recalculate `stop_sequence`
    - Update dropped driver's `route_execution.status = 'cascaded'`
    - For each receiving driver: call `_push_route_update_to_driver_app(driver_id, new_stops)` (see task 69)
    - For each reassigned stop with recipient phone: enqueue `tasks.send_eta_update_contact(stop_id, new_eta)` in Celery `medium` queue
    - Write `InterventionAction` record
    - Update `CascadeIncident` record: `resolved_at`, `auto_executed`, `selected_option_rank`
    - Emit Socket.io `cascade:resolved` to operator room
    - Send Twilio SMS to dispatcher: confirm action taken, stops saved count
    - Write to `intervention_actions` table (full audit payload)
69. Create `services/driver_app_push.py`: `push_route_update(driver_id, updated_stops)`. Socket.io emit to room `driver:{driver_id}`, event `route:updated`, payload: `{ stops: [{ stop_id, sequence, address, lat, lng, window_start, window_end, notes }] }`. Also write updated stop list to Redis `driver:{driver_id}:route_stops` (TTL 24h).

**Cascade Approval UI (Week 8)**

70. Create `store/cascade.ts` Zustand store (schema from Module 6 spec).
71. Create `hooks/useCascade.ts`: subscribe to Socket.io events `cascade:incident_opened`, `cascade:options_ready`, `cascade:executing`, `cascade:resolved`. Update Zustand store.
72. Create `components/CascadeOptionCard.tsx`: displays one redistribution option. Fields: option label (e.g., "Minimise SLA Breaches"), stops reassigned count, SLA windows preserved count, affected drivers (names + stop counts), estimated overtime hours and cost, average customer ETA delta minutes. "Approve This Option" button. Button disabled while another option is being approved (loading state).
73. Create `components/CascadeCountdownTimer.tsx`: circular progress timer. Props: `total_seconds`, `elapsed_seconds`. Shows "Auto-executing in X:XX" when < 60 seconds remain. Red flash animation at < 30 seconds.
74. Create `components/CascadeApprovalModal.tsx`: full-screen modal overlay. Header: "Driver Cascade Required — [Driver Name] is down with [X] stops remaining." Three `CascadeOptionCard` components side-by-side. `CascadeCountdownTimer` across the top. "Dismiss (auto-execute will proceed)" link at bottom — requires confirmation dialog.
75. Update `pages/Dashboard.tsx`: render `CascadeApprovalModal` when `cascade.incidentStatus === 'options_ready'`. Modal is always on top of everything else (z-index management in Tailwind).
76. Add API call in `CascadeOptionCard` approve button: `POST /api/v1/cascade/{incident_id}/approve` with `{ option_rank, dispatcher_id }`. On success: revoke Celery auto-execute task via `DELETE /internal/celery/cascade-task/{task_id}` (internal endpoint). Show confirmation toast.

**Driver App Integration (Week 8–9)**

77. Document driver app Socket.io room subscription protocol in `docs/driver-app-integration.md`: driver subscribes to `driver:{driver_id}` room on route start. Events received: `route:updated`, `route:stop_removed`, `route:resequenced`, `route:stop_updated` (safe drop note added).
78. Create `routers/driver_app.py`: endpoints `POST /api/v1/driver/route-ack` (driver confirms receipt of route update), `GET /api/v1/driver/{driver_id}/current-route` (fetches current stop list from Redis).
79. Create `tasks/verify_driver_app_receipt.py`: after pushing a route update, schedule a Celery task to check `driver:{driver_id}:route_ack_pending` Redis key. If driver hasn't acknowledged within 60 seconds, retry push. If no ack after 3 retries: alert dispatcher. (Handles case where driver app is backgrounded or has connectivity issues.)
80. Write end-to-end test `tests/e2e/test_cascade_pipeline.py`: simulate driver drop → cascade trigger → options generated → approval → route updates pushed to receiving drivers → customer contact tasks enqueued. Assert all state transitions in PostgreSQL and Redis.

**Intervention Action Audit (Week 9)**

81. Create `services/audit_logger.py`: `write_audit_entry(action_type, payload, autonomous, actor_id)`. Writes to `intervention_actions` PostgreSQL table AND archives to S3 `s3://odocs-audit-logs/{operator_id}/{date}/{action_id}.json` via background Celery task. S3 objects are immutable (use S3 Object Lock in Compliance mode for regulated operators).
82. Add endpoint `GET /api/v1/audit/interventions` (query: date range, action type, autonomous only). Returns paginated audit log for compliance review.

### Key Technical Decisions

- **Greedy insertion over full re-optimisation:** A full route optimisation run (e.g., calling the routing engine to re-optimise all receiving drivers' routes) would take 30–90 seconds and disrupt in-progress deliveries. Greedy nearest-insertion produces a good (not optimal) redistribution in under 2 seconds, which is the correct trade-off when SLA windows are closing. A full re-optimisation can be offered as a secondary option in the approval UI ("or let us fully re-optimise overnight for tomorrow").
- **5-minute auto-execute default:** This is the balance between giving the dispatcher enough time to review (3 minutes of review + 2 minutes of buffer) and not letting the cascade get out of control. The default is configurable per operator (`operator.cascade_auto_execute_timeout_seconds`). High-trust operators can increase to 10 minutes; high-urgency operators can decrease to 2 minutes.
- **Socket.io for driver app push (not APNS/FCM):** Driver apps are running foreground during a delivery day. Socket.io provides reliable, low-latency push without the complexity of managing APNs/FCM certificates and delivery receipts. If the driver app goes to background, the retry mechanism (task 79) handles it.

### Definition of Done

- [ ] Simulated driver drop triggers cascade incident within 30 seconds
- [ ] Three redistribution options are generated and displayed in CascadeApprovalModal within 30 seconds of trigger
- [ ] Countdown timer is visible and accurate
- [ ] Dispatcher approval executes redistribution within 5 seconds of approval click
- [ ] Auto-execute fires exactly at T+5 minutes if dispatcher does not respond (validated in staging with real Celery countdown)
- [ ] Receiving driver apps receive `route:updated` event within 10 seconds of execution
- [ ] Full audit trail written to PostgreSQL `intervention_actions` table and S3 for every executed cascade
- [ ] All cascade actions (manual and autonomous) are distinguishable in audit log via `autonomous` flag

### Risks and Mitigations

| Risk | Probability | Mitigation |
|---|---|---|
| Celery countdown timer fires early or late | Low | Use `eta` parameter (absolute datetime) rather than `countdown` (relative seconds) for Celery tasks — avoids drift. Test countdown accuracy with `tests/test_celery_countdown.py`. |
| Greedy insertion produces suboptimal redistribution that causes more breaches | Medium | Validate algorithm against historical cascade events (if any exist) and synthetic scenarios. Add `estimated_breach_count` to each option card so dispatcher can see the consequence of choosing each option. |
| Driver app push delivery not confirmed | Medium | Implement the receipt acknowledgement mechanism (task 79). Track unconfirmed pushes in Datadog. Alert dispatcher if driver app unreachable. |
| Race condition: dispatcher approves while auto-execute task fires simultaneously | Low | Use PostgreSQL row-level locking (`SELECT FOR UPDATE`) on `cascade_incidents.selected_option_rank` when checking if cascade is already resolved. Celery task checks this lock before executing. |

---

## Phase 4: Proactive Comms & Polish (Weeks 10–12)

### Goals

Build the Proactive Customer Communication Agent including LLM message generation and Twilio inbound reply handling. Implement the SLA Economics Layer with full economic risk scoring and portfolio-level health dashboard. Polish the full dashboard — responsive layout, dark mode, route trail visualisation, causal delay explanations. Set up Datadog monitoring, PagerDuty alerting, and load testing.

### Atomic Tasks

**Proactive Customer Communication Agent (Week 10)**

83. Create migration `migrations/007_customer_contacts.py`: create `customer_contacts` table and `customer_opt_outs` table.
84. Create Jinja2 prompt templates directory `prompts/`. Create `prompts/proactive_contact.jinja2`:
    ```
    You are writing a friendly, concise SMS for a parcel delivery.
    Recipient: {{ recipient_first_name }}
    Delivery: {{ delivery_description }}
    Estimated arrival: {{ eta_window_start }} to {{ eta_window_end }}
    Write an SMS (max 155 characters) that:
    1. Tells them their parcel is arriving today in the time window
    2. Asks them to reply: 1 to confirm they're home, 2 for a safe drop location, 3 to reschedule
    Be warm and brief. No emojis. End with "Reply STOP to opt out."
    ```
85. Create `prompts/classify_intent.jinja2`: structured classification prompt. Output must be valid JSON matching `IntentClassification` Pydantic schema: `{ intent: "confirm_home|safe_drop|reschedule|opt_out|unknown", safe_drop_notes: string|null, reschedule_preference: string|null }`.
86. Create `services/llm_client.py`: `LLMClient` class. `generate_sms(template_name, context) -> str`. Uses `openai.AsyncOpenAI` with model `gpt-4o-mini`. Temperature 0.3 (low randomness for SMS generation). Retry on rate limit (3 attempts, exponential backoff). Validates output length ≤ 160 characters; falls back to Jinja2 template string if LLM output is too long.
87. Create `services/intent_classifier.py`: `classify_reply(message_body: str, stop_context: StopContext) -> IntentClassification`. Calls LLM with `classify_intent.jinja2` prompt. Parses JSON output. Falls back to keyword matching (`"yes"/"confirm"` → `confirm_home`, `"stop"` → `opt_out`) if LLM API is unavailable.
88. Create `tasks/send_proactive_contact.py`: Celery task (`medium` queue). Steps: check opt-out, check idempotency (Redis `contact_sent:{stop_id}`), generate message via `llm_client.generate_sms()`, send via Twilio, write `CustomerContact` record, update `stop_executions.customer_contacted = True`, set Redis idempotency key.
89. Create `routers/webhooks.py`: `POST /api/v1/webhooks/twilio-inbound`. Validate Twilio request signature using `RequestValidator(TWILIO_AUTH_TOKEN).validate(url, post, signature)`. Extract `From`, `Body`, `MessageSid`. Match `From` phone number to `stop_execution_id` via PostgreSQL lookup on `stop_executions.recipient_phone` where `status = 'pending' AND customer_contacted = true`. Produce event to Kafka `customer-contact-queue`. Return empty TwiML `<Response></Response>`.
90. Create `consumers/contact_reply_consumer.py`: consumer group `contact-processor`, topic `customer-contact-queue`. On each event: call `intent_classifier.classify_reply()`. Execute intent actions (update stop status, push driver app note for safe drop, create reschedule record, log opt-out). Write inbound `CustomerContact` record. Emit Socket.io event to dispatcher.
91. Create `tasks/send_eta_update_contact.py`: Celery task for post-cascade customer notification. Called by `cascade_executor.py`. Generates a different message variant: "Your delivery has been rerouted due to an unexpected change. New estimated arrival: [window]. Reply 2 for safe drop or 3 to reschedule."
92. Add endpoint `GET /api/v1/operators/{operator_id}/contact-stats` (returns metrics from Module 7 spec).
93. Add `CommunicationLog` panel to dispatcher dashboard: small panel showing recent customer contacts and their response status. Badge on `StopMarker` on FleetMap if that stop has an active customer conversation.

**SLA Economics Layer — Full Implementation (Week 10–11)**

94. Update `SLARiskGauge` component to display full portfolio health: total economic risk ($), expected breach count (no intervention), stops at risk count, stops critical count. Add a colour-coded health band: green (<$500 economic risk), amber ($500–$2,000), red (>$2,000).
95. Create endpoint `GET /api/v1/clients/{client_id}/sla-dashboard`: returns per-client SLA health for embedding in client portal. Fields: `active_stops`, `at_risk_stops`, `critical_stops`, `economic_risk`, `projected_breach_count`. Suitable for polling at 60-second intervals from a client-facing portal.
96. Create `scripts/validate_breach_predictions.py`: weekly validation script. Joins `eta_predictions` and `sla_breaches` tables to compute precision/recall of breach probability predictions at threshold 0.6. Outputs CSV and logs to Datadog custom metric `odocs.sla.prediction_accuracy`. Run via Celery Beat every Monday 09:00 AEST.
97. Create endpoint `GET /api/v1/operators/{operator_id}/sla-report/daily`: generates end-of-day SLA report. Fields per client: total deliveries, on-time count, at-risk interventions attempted, breaches confirmed, penalties triggered, proactive contacts sent, FTD count. Used by ops manager for daily EOD review.

**Dashboard Polish (Week 11)**

98. Add route trail visualisation to `FleetMap`: when a driver marker is clicked, render a polyline showing the last 30 minutes of GPS positions (query TimescaleDB `driver_positions_1min` continuous aggregate). Render completed stops as greyed-out markers with checkmarks. Render pending stops with colour encoding: green (on time), amber (at risk), red (critical).
99. Add causal delay explanation to `AlertCard`: field `alert.description` from Alert Decision Engine should contain the structured causal string (e.g., "Route behind by 43 min. Causes: Traffic +25min, 2 access failures (+10min), driver lunch +8min"). Display this in an expandable section "Why is this late?" with an info icon.
100. Implement responsive layout: dashboard should work on 1920×1080 (full ops screen), 1440×900 (laptop), and 1024×768 (tablet). Use Tailwind responsive prefixes. FleetMap shrinks but remains primary panel; AlertPanel becomes a collapsible drawer on tablet.
101. Add dark mode: Tailwind `dark:` class variants on all components. Toggle stored in localStorage. Default: dark mode for operational environments (easier on eyes during long shifts).
102. Add keyboard shortcuts: `A` to focus AlertPanel, `M` to focus map, `Esc` to close modals, number keys `1`–`3` to approve intervention options from AlertPanel (accessibility for high-stress situations). Implement via `useHotkeys` hook.
103. Implement `RouteProgressList` sorting: clickable column headers for `completion_pct`, `behind_by_minutes`, `active_alert_count`. Default sort: `behind_by_minutes DESC` (most behind routes at top).

**Monitoring and Alerting (Week 11–12)**

104. Install Datadog agent on all ECS tasks via `DD_API_KEY` environment variable and Datadog ECS task definition sidecar. Enable APM traces on all FastAPI services (`ddtrace-run` in Dockerfile CMD).
105. Create Datadog dashboard `odocs-operational` with panels:
    - GPS ingest rate (events/sec)
    - ETA prediction latency (p50, p95, p99)
    - SLA breach evaluator throughput
    - Kafka consumer lag per consumer group
    - Active alert count
    - Alert-to-action time (percentiles)
    - Twilio message delivery rate
    - LLM API latency and cost
106. Create PagerDuty integration. Configure Datadog monitors to alert PagerDuty on:
    - Kafka consumer lag > 30 seconds (any consumer group) — P1
    - ETA prediction service latency p99 > 500ms for 5 consecutive minutes — P2
    - GPS ingest error rate > 5% for 2 consecutive minutes — P1
    - Redis memory usage > 80% — P2
    - RDS CPU > 85% for 10 consecutive minutes — P2
    - WebSocket server connection failures > 10% — P1
107. Create `scripts/load_test_full_pipeline.py`: Locust load test that simulates 35 drivers posting GPS at 5-second intervals, 5 dispatcher WebSocket connections receiving updates, and 10 simultaneous SLA breach alert scenarios. Target: GPS ingest p99 < 50ms, ETA update to dashboard WebSocket push p95 < 3 seconds, no consumer lag accumulation. Run in staging before production deployment.
108. Create runbook `docs/runbooks/kafka-consumer-lag.md`, `docs/runbooks/cascade-auto-execute-failure.md`, `docs/runbooks/twilio-outage.md`. Each runbook: symptoms, diagnosis steps, remediation steps, escalation path.

**Production Cutover (Week 12)**

109. Provision production Terraform workspace: copy dev configuration, increase RDS to `db.m6g.large`, ECS task counts to minimum 2 per service, enable RDS Multi-AZ, enable MSK Multi-AZ.
110. Run `alembic upgrade head` on production RDS. Verify TimescaleDB hypertable and continuous aggregates created.
111. Load production SLA configs for first operator customer via `scripts/seed_sla_configs.py`.
112. Onboard first operator: create operator account, create driver accounts, import first week of route plans. Run shadow mode for 5 business days: system generates alerts and predictions but does NOT auto-execute (auto-execute disabled via feature flag `FEATURE_AUTO_EXECUTE=false`). Review prediction accuracy and alert quality.
113. Enable auto-execute after shadow mode validation: set `FEATURE_AUTO_EXECUTE=true` in Secrets Manager. Increase `cascade_auto_execute_timeout_seconds` to 10 minutes for first live week (conservative). Reduce to 5 minutes after 2 weeks.

### Key Technical Decisions

- **GPT-4o-mini over GPT-4o for SMS generation:** The task (generate a 155-character SMS) does not require the reasoning capacity of GPT-4o. GPT-4o-mini at $0.15/1M input tokens vs GPT-4o at $2.50/1M input tokens produces no meaningful quality difference for this use case. At 2,800 proactive contacts/day, the cost difference is ~$15/month vs ~$250/month.
- **Feature flag for auto-execute:** The first production deployment uses shadow mode (alerts generated, actions recommended, but not autonomously executed). This builds operator trust before autonomous actions are enabled. The feature flag `FEATURE_AUTO_EXECUTE` in AWS Systems Manager Parameter Store allows instant rollback without redeployment.
- **Celery Beat for periodic tasks (not cron):** Celery Beat runs within the same container infrastructure and uses the same Redis broker. It does not require separate cron infrastructure or EC2 instances. It also provides task history and visibility via Flower monitoring UI.
- **S3 + PostgreSQL dual audit log:** PostgreSQL is fast for operational queries (recent interventions, active incidents). S3 Object Lock provides immutable compliance archiving for any regulated operator clients. Writing to both adds minimal overhead (S3 write is async via Celery background task).

### Definition of Done

- [ ] Proactive SMS is sent to a test recipient when their stop crosses 40% breach probability
- [ ] Inbound reply "1" correctly classifies as `confirm_home` and updates stop record
- [ ] Inbound reply "3" triggers reschedule flow, removes stop from active route, sends confirmation SMS
- [ ] STOP reply correctly opts out recipient and prevents future contacts
- [ ] SLARiskGauge updates in real time and accurately reflects portfolio economic risk
- [ ] Dashboard loads to fully operational state within 3 seconds on a fresh browser session
- [ ] Responsive layout works on 1024×768 tablet (tested in Chrome DevTools responsive mode)
- [ ] Datadog dashboard shows all 8 operational panels with live data
- [ ] PagerDuty receives test alert for each configured monitor
- [ ] Load test: 35 simulated drivers at 5-second intervals, p99 GPS ingest < 50ms, no consumer group lag > 5 seconds
- [ ] Shadow mode operates for 5 business days with zero false auto-execute events
- [ ] Operator can toggle auto-execute on/off via dashboard settings (backed by Secrets Manager parameter)

### Risks and Mitigations

| Risk | Probability | Mitigation |
|---|---|---|
| LLM API outage during proactive comms | Low-Medium | Fallback to Jinja2 template string (pre-written, validated against 160-char limit) implemented in `llm_client.generate_sms()`. Outage is transparent to recipients (message quality degrades slightly, not silently). |
| Twilio message delivery rate below 90% | Medium | Monitor Twilio delivery receipts in Datadog. For undelivered messages, retry once after 5 minutes. Flag undeliverable phone numbers in `stop_executions` for dispatcher awareness. |
| LLM generating non-compliant messages (too long, inappropriate content) | Low | Validate output length and run through a blocklist before sending. Log all LLM generations for review. Pydantic validation on LLM output; reject and fall back to template on validation failure. |
| Operator staff resistance to autonomous actions | High | Start with shadow mode and conservative timeout (10 minutes). Show operator the auto-execute action log prominently in the dashboard. Make it easy to revert auto-executed actions via `POST /api/v1/interventions/{id}/revert`. Frame auto-execute as "we caught it for you" not "we acted without you". |
| Datadog cost spike from high GPS event volume | Medium | Use Datadog custom metrics sparingly. Use log-based metrics for high-volume events (GPS ingest rate) rather than submitting each ping as a metric. Estimate cost at $0.08 per 100 custom metric-hours before launch. |
