# Challenge 1: Real-Time Route Intelligence & FTD Reduction — Modular Breakdown

---

## Module 1: Risk Scoring Engine

### Purpose

The Risk Scoring Engine is the predictive core of the platform. It generates a calibrated failure probability score (0–100) for every parcel in each day's manifest, before routes are built and before drivers leave the depot. It enables every downstream module — SMS outreach, route ordering, cascade prevention, and dispatcher triage — to act on predicted outcomes rather than reacting to actual failures.

### Key Functions

- **Feature Extraction**: For each parcel, assemble a 47-dimensional feature vector by joining address history, recipient history, temporal context, driver assignment, and external signals (weather, postcode FTD rate).
- **Batch Scoring**: Accept a manifest batch of up to 3,500 parcels and return risk scores within 45 seconds. Parallelise across 4 inference workers using Python multiprocessing.
- **On-Demand Re-Scoring**: Score individual parcels added to manifests after the initial batch run (<20ms per parcel, synchronous).
- **SHAP Explainability**: For each score, compute the top 3 SHAP feature contributions and return as human-readable factors for dispatcher UI display.
- **Risk Tier Assignment**: Map continuous score to LOW/MEDIUM/HIGH/CRITICAL tiers with configurable threshold overrides per client (some clients want more aggressive intervention).
- **Intervention Recommendation**: Based on risk tier, top factors, and recipient history, recommend the appropriate intervention: NONE / SMS_CONFIRM / DISPATCHER_REVIEW / MANUAL_RESCHEDULE.
- **Model Registry Management**: Serve the currently active model version; support warm A/B traffic splitting between two model versions; hot-swap to a new model version without service restart.
- **Feature Logging**: Write the complete feature vector for every scored parcel to `feature_log` table at scoring time, enabling exact reproduction of the inference context for later supervised learning.

### API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/risk/score/batch` | Score all parcels in a manifest. Body: `{manifest_id: UUID}`. Triggers async batch job; returns `{job_id, estimated_completion_seconds}`. |
| `GET` | `/api/v1/risk/score/batch/{job_id}` | Poll batch job status. Returns `{status, progress_pct, completed_at}`. |
| `GET` | `/api/v1/risk/scores/{manifest_id}` | Fetch all risk scores for a manifest once batch is complete. Returns array of `RiskScore` objects with factors and recommendations. |
| `POST` | `/api/v1/risk/score/single` | Score a single parcel synchronously. Body: `{delivery_id: UUID}`. Returns `RiskScore` within 200ms. |
| `GET` | `/api/v1/risk/scores/delivery/{delivery_id}` | Fetch the active risk score for a specific delivery. |
| `GET` | `/api/v1/risk/model/active` | Return active model version metadata: version, training date, AUC-ROC, feature count. |
| `POST` | `/api/v1/risk/model/activate/{version}` | Promote a candidate model version to active. Requires `dispatcher_admin` role. |
| `GET` | `/api/v1/risk/model/ab-config` | Return current A/B split configuration (version_a, version_b, split_pct). |
| `PUT` | `/api/v1/risk/model/ab-config` | Update A/B split configuration. Body: `{version_a, version_b, split_pct_b}`. |

### State Management

- **Redis feature store** (`features:{parcel_id}`): Hash set of computed feature values. Written by Feature Engineering Service before scoring. TTL: 24 hours. Eviction policy: `allkeys-lru`.
- **Redis model cache** (`model:active`, `model:candidate`): Stores loaded XGBoost model objects serialised with pickle (in-process) — model object is loaded once at service startup, not per-request.
- **PostgreSQL `risk_scores` table**: Persistent record of every score. `is_active = false` on old scores when delivery is re-scored.
- **PostgreSQL `feature_log` table**: Immutable append-only log. Never updated, only inserted. Retention policy: 365 days.
- **PostgreSQL `model_registry` table**: Tracks all model versions with metadata and `is_active` flag.

### Dependencies

- PostgreSQL (`deliveries`, `addresses`, `recipients`, `drivers`, `feature_log`, `risk_scores`, `model_registry`)
- Redis (feature store, result cache)
- S3 (`s3://odocs-models/risk-scoring/{version}/model.joblib` — model artifact store)
- Feature Engineering Service (internal service-to-service call or shared library)
- SMS Service (publishes `recommended_intervention` for consumption by SMS scheduling)
- Learning Loop Service (consumes `feature_log` for retraining)
- Datadog (custom metrics: `risk_scoring.batch_duration_seconds`, `risk_scoring.score_distribution`, `risk_scoring.model_version`)

---

## Module 2: Two-Way SMS Engine

### Purpose

The Two-Way SMS Engine converts the risk scoring output into proactive recipient engagement. It schedules and sends structured SMS messages to recipients of high-risk deliveries, processes their replies, extracts structured preferences and instructions, and updates downstream records to convert probable failures into successful deliveries. It is the primary human-in-the-loop mechanism — connecting the AI's predictions to the actual recipient's real-world situation.

### Key Functions

- **Outbound Scheduling**: For each parcel with risk tier >= HIGH and recipient not opted out, schedule two outbound SMS touches: day-before (4:00–6:00pm, day prior) and pre-arrival (2 hours before predicted driver arrival at stop).
- **Quiet Hours Enforcement**: No SMS sent before 8:00am or after 9:00pm AEST. Reschedule if calculation falls outside window.
- **Message Personalisation**: Compose message body with recipient name, delivery date, planned time window, and structured response menu. Vary message copy by parcel type (healthcare parcels use different urgency framing than standard e-commerce).
- **Inbound Reply Processing**: Receive Twilio POST webhook for every inbound SMS. Match to open conversation by recipient phone number. Parse response.
- **Structured Response Parser**: Regex-first parser for numeric responses (1/2/3/4). Handles common variants ("yes", "yep", "ok", "1" → confirm; "no", "can't", "not home" → reschedule signal).
- **LLM Fallback Intent Classification**: For responses not matched by structured parser, call GPT-4o-mini with a constrained extraction prompt. Extract intent (confirm / reschedule / access_instructions / redirect / unclear) and details (time window, access note, neighbour name, collection point preference).
- **Preference Persistence**: Write extracted preferences to `recipient_preferences` table. Push access instructions to route stop's `driver_notes` field in real time.
- **Time Window Update**: For recipients who request a different delivery time, publish a `route.optimizer.events` Kafka message with `{event_type: "time_window_update", delivery_id, new_start, new_end}` to trigger route re-optimisation.
- **Opt-Out Handling**: Any reply containing "STOP", "OPTOUT", "UNSUBSCRIBE" (case-insensitive) immediately adds recipient phone to `sms:optouts` Redis set AND updates `recipients.sms_opt_out = true`. No further SMS ever sent to this number. Confirmation SMS sent: "You've been unsubscribed from delivery notifications. Reply START to resubscribe."
- **Delivery Receipt Tracking**: Process Twilio status callbacks. Mark messages as `delivered`. If `failed` or `undelivered`, log to `sms_messages`, update conversation status, and trigger fallback notification (push notification if driver app is installed by client, or dispatcher alert).
- **Conversation Expiry**: Conversations with no reply within 18 hours of first send are marked `expired`. No further outbound messages sent for that delivery.

### API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/sms/schedule/batch` | Schedule SMS for all high-risk parcels in a manifest. Body: `{manifest_id}`. Enqueues Celery tasks. |
| `POST` | `/api/v1/sms/schedule/single` | Schedule SMS for a single delivery immediately. Body: `{delivery_id}`. |
| `GET` | `/api/v1/sms/conversations/{delivery_id}` | Fetch the SMS conversation for a delivery, including all messages and extracted preference. |
| `GET` | `/api/v1/sms/conversations` | List conversations with filters: `?date=2026-03-14&status=replied&outcome=confirmed`. Paginated. |
| `POST` | `/webhooks/twilio/inbound` | Twilio webhook endpoint for inbound SMS. Validated via Twilio signature. Triggers reply processing pipeline. Returns 200 OK with empty TwiML. |
| `POST` | `/webhooks/twilio/status` | Twilio webhook for delivery status updates. Updates `sms_messages.status`. |
| `DELETE` | `/api/v1/sms/optout/{phone_e164}` | Manual opt-out registration by dispatcher or client portal. |
| `POST` | `/api/v1/sms/optout/{phone_e164}/resubscribe` | Re-subscribe a previously opted-out recipient (only if recipient initiated). |
| `GET` | `/api/v1/sms/stats/{date}` | Daily SMS performance stats: sent, delivered, replied, confirmed, rescheduled, opted_out, llm_fallback_rate. |
| `POST` | `/api/v1/sms/send/manual` | Dispatcher manually triggers an SMS to a specific recipient. Body: `{delivery_id, message_body}`. Logged to conversation. |

### State Management

- **Redis opt-out set** (`sms:optouts`): Set of E.164 phone numbers. Checked before every outbound send. Populated by opt-out processing and bulk seed from `recipients` table on service startup.
- **Redis conversation lock** (`sms:lock:{conversation_id}`): 30-second advisory lock during reply processing to prevent duplicate processing if Twilio retries the webhook.
- **PostgreSQL `sms_conversations`** and **`sms_messages`**: Persistent record of all conversations and messages.
- **Celery task state** (Redis backend): Task IDs for scheduled sends tracked in Redis. Allows cancellation if delivery is cancelled or rescheduled before the SMS fires.
- **PostgreSQL `recipient_preferences`**: Updated in real time as replies are processed. Read by Route Optimizer and Driver App on stop detail screen.

### Dependencies

- Twilio SDK (`twilio-python>=8.0`) + Twilio Messaging Service SID (from `AWS_SECRETS_MANAGER` key `prod/twilio`)
- OpenAI API (gpt-4o-mini, for LLM fallback; key from `AWS_SECRETS_MANAGER` key `prod/openai`)
- PostgreSQL (`sms_conversations`, `sms_messages`, `recipients`, `recipient_preferences`, `deliveries`, `route_stops`)
- Redis (opt-out set, conversation lock, Celery broker/backend)
- Kafka producer (publishes `route.optimizer.events` for time window updates)
- Risk Scoring Engine (consumes `recommended_intervention` from `risk_scores` table)
- Route Optimizer Service (receives time window update events)
- Datadog (metrics: `sms.sent_count`, `sms.reply_rate`, `sms.llm_fallback_rate`, `sms.opt_out_count`)

### TCPA and Australian Spam Act Compliance

- All outbound SMS must be sent only between 8:00am and 9:00pm in the recipient's local timezone (Queensland: AEST, no DST).
- Opt-out processing must complete within the same session as receipt — Redis write happens synchronously in the webhook handler before returning 200 OK.
- All outbound messages must include identification: "Delivery notification from [Operator Name]."
- A/B copy testing must not change the opt-out mechanism or obscure the sender identity.
- Message logs retained for 7 years per Australian Privacy Act requirements.

---

## Module 3: Dynamic Route Optimizer

### Purpose

The Dynamic Route Optimizer builds the initial optimised route plan from risk-adjusted parcel data and continuously re-optimises in-flight routes in response to real-world events throughout the operating day. It transforms static overnight planning into a living, adaptive delivery plan that improves as the day progresses.

### Key Functions

- **Initial Route Build (Pre-Dispatch)**: Given the full day's parcel list, driver roster, and vehicle assignments, run the VRP solver to produce optimised routes. Incorporate risk scores into stop sequencing (high-risk confirmed stops positioned optimally; unconfirmed high-risk stops front-loaded in route to allow re-intervention time).
- **Travel Time Matrix Construction**: Query Google Maps Distance Matrix API for all stop-to-stop travel time pairs, using current traffic conditions and predicted traffic for the morning window. Cache results in Redis.
- **Event Consumption**: Consume events from Kafka `route.optimizer.events` topic. Event types: `driver_delay`, `stop_completed`, `stop_failed`, `time_window_update`, `new_stop_added`, `driver_unavailable`, `manual_reoptimise`.
- **Re-Optimisation Trigger Logic**: Not all events trigger re-optimisation. Trigger rules: driver delay >15 minutes AND >5 remaining stops; stop_failed for a HIGH/CRITICAL risk parcel (attempt reassignment to nearby driver); time_window_update for any stop; new_stop_added to depot's active manifest; driver_unavailable (full reassignment of their remaining stops).
- **Constraint Enforcement**: Hard constraints: vehicle capacity (parcel count and weight), driver shift end time, client-specified delivery windows (SLA time windows are hard), driver HOS limits. Soft constraints: recipient preferred windows (can be violated with penalty in cost function), driver zone preferences, vehicle type requirements for oversized parcels.
- **Route Push**: After re-optimisation, publish updated stop sequence to `gps.driver.positions` topic (picked up by api-gateway WebSocket push) AND write updated `route_stops` records to PostgreSQL.
- **Nearby Driver Matching**: For failed stops that need reassignment, query currently active drivers within a 5km radius of the failed stop's address, check their remaining capacity, and recommend the best reassignment option to the dispatcher.
- **ETA Prediction**: Real-time ETA for each remaining stop computed from current driver GPS position + remaining stop sequence + current traffic travel times. Updated every 60 seconds per active driver and pushed via WebSocket to dispatcher dashboard.
- **Route Lock Management**: Dispatcher can lock individual stops (prevent re-ordering during re-optimisation). Locked stops are treated as fixed nodes in the VRP.

### API Endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/routes/build` | Trigger full route build for a delivery date. Body: `{delivery_date, driver_ids[], depot_id}`. Returns `{job_id}`. Async. |
| `GET` | `/api/v1/routes/build/{job_id}` | Poll route build job. Returns status and estimated completion. |
| `GET` | `/api/v1/routes/{route_id}` | Fetch route with all stops, current driver position, ETAs. |
| `GET` | `/api/v1/routes/active` | List all active routes for today with summary stats (completed stops, delay minutes, at-risk count). |
| `POST` | `/api/v1/routes/{route_id}/reoptimise` | Manually trigger re-optimisation for a specific route. Dispatcher action. |
| `GET` | `/api/v1/routes/{route_id}/stops` | List all stops for a route with sequence, status, predicted arrival, risk flag. |
| `PUT` | `/api/v1/routes/stops/{stop_id}/sequence` | Manually override stop sequence position. Body: `{new_position: int}`. Triggers soft re-optimise of affected stops. |
| `PUT` | `/api/v1/routes/stops/{stop_id}/lock` | Lock a stop's sequence position. Prevents re-optimisation from reordering it. |
| `POST` | `/api/v1/routes/stops/{stop_id}/reassign` | Reassign a specific stop to a different driver. Body: `{target_driver_id}`. |
| `GET` | `/api/v1/routes/nearby-drivers` | Given a location, find drivers with capacity within a radius. Query: `?lat=&lng=&radius_km=&min_capacity=`. |
| `GET` | `/api/v1/routes/travel-matrix` | Fetch cached travel time matrix for a route (debug/admin). |
| `POST` | `/api/v1/gps/position` | Driver app GPS position submit. Body: `{driver_id, lat, lng, heading, speed, accuracy, timestamp}`. Forwarded to Kafka. |

### State Management

- **Kafka `route.optimizer.events`**: Topic consumed by this service's internal Kafka consumer group (`route-optimizer-cg`). Event processing is idempotent — each event carries an `event_id` UUID; processed event IDs stored in Redis set (`ropt:processed_events`) with 24h TTL to deduplicate Kafka redeliveries.
- **Redis travel time cache** (`ropt:ttm:{route_date}:{origin_id}:{dest_id}`): Pairwise travel times. 30-minute TTL for real-time traffic accuracy. Warmed in background 30 minutes before re-optimisation.
- **Redis route lock** (`ropt:lock:{route_id}`): Mutex during active re-optimisation. Prevents concurrent solver runs on the same route. 120-second TTL (solver timeout upper bound).
- **PostgreSQL `routes` and `route_stops`**: Source of truth. Updated transactionally after each re-optimisation. `sequence_version` incremented on each re-opt.
- **Driver app state**: Pushed via WebSocket after each re-optimisation. Driver app stores the current stop sequence locally and reconciles with server version on reconnect.

### Dependencies

- Google OR-Tools 9.x (`ortools` Python package)
- Google Maps Platform (Distance Matrix API, Directions API; key from `AWS_SECRETS_MANAGER` key `prod/google_maps`)
- Apache Kafka / AWS MSK (consumer for `route.optimizer.events`; producer for driver app push events)
- PostgreSQL (`routes`, `route_stops`, `deliveries`, `drivers`, `vehicles`, `addresses`)
- Redis (travel time cache, event deduplication, route lock)
- Datadog (metrics: `route_optimizer.reoptimise_count`, `route_optimizer.solve_duration_ms`, `route_optimizer.stops_reassigned`)

---

## Module 4: Cascade Prevention Agent

### Purpose

The Cascade Prevention Agent is a continuously running monitoring service that detects emerging driver delay scenarios before they compound into multiple delivery failures. It acts proactively — contacting recipients of at-risk stops and offering alternatives, notifying dispatchers with triage options — before the driver arrives and fails. It is the defensive layer that catches failures the risk scoring model and SMS engine did not prevent.

### Key Functions

- **Real-Time Delay Detection**: Consume GPS events from Kafka `gps.driver.positions` and stop completion events from `delivery.stop.events`. For each active driver, maintain a running delay estimate: `current_delay = actual_position_time - planned_position_time`. Compute via comparison of actual completed stops against planned schedule.
- **At-Risk Stop Identification**: When a driver's delay exceeds configurable threshold (default: 20 minutes, configurable per operator), identify which remaining stops are at risk of not being attempted within their planned or preferred time windows. Apply a severity model: stops with hard SLA windows are CRITICAL; residential stops after 5:30pm are HIGH; commercial stops after 4:00pm are HIGH.
- **Cascade Probability Model**: A lightweight logistic regression model (trained on historical cascade events) that predicts the probability each at-risk stop will result in a failure given the driver's current delay trajectory. Inputs: delay minutes, stop count remaining, time of day, stop type, driver historical completion rate when behind schedule. Outputs: cascade probability 0–1.
- **Automated Recipient Outreach**: For stops with cascade probability >0.6, trigger an SMS via SMS Engine with a cascade-specific template: honest, proactive, and offering concrete alternatives. Do not send if recipient has already received an SMS within 4 hours (throttle).
- **Alternative Options Delivery**: Present recipients with four options in the SMS: (1) Wait for driver (estimated new arrival time provided), (2) Leave with neighbour (if neighbour preference exists), (3) Safe drop (if safe drop authorised on delivery record), (4) Reschedule to next available slot. Process reply via SMS Engine inbound handler.
- **Dispatcher Triage Notification**: Simultaneously alert dispatcher via WebSocket push to dashboard. Triage panel shows: driver name, delay minutes, list of at-risk stops with cascade probability, available nearby drivers with capacity, recommended action (reassign / extend shift / accept reschedule).
- **Reassignment Execution**: If dispatcher approves reassignment (or if auto-reassign is enabled for operators above a configurable tier), call Route Optimizer to execute the stop transfer to the nearest available driver.
- **Outcome Recording**: Record each cascade prevention event to `cascade_events` table: driver, at-risk stops, action taken (outreach/reassignment/none), outcome (prevented/failed despite intervention), for use in learning loop.

### API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/cascade/active` | List all currently active cascade situations (drivers with delay >threshold). Returns driver, delay, at-risk stop count, recommended actions. |
| `GET` | `/api/v1/cascade/driver/{driver_id}` | Detailed cascade status for a specific driver: delay trajectory, at-risk stops with probabilities, outreach status. |
| `POST` | `/api/v1/cascade/approve-reassignment` | Dispatcher approves a recommended stop reassignment. Body: `{stop_ids[], target_driver_id}`. Calls Route Optimizer. |
| `POST` | `/api/v1/cascade/dismiss/{driver_id}` | Dispatcher dismisses cascade alert for a driver (manual override — driver is recovering). |
| `GET` | `/api/v1/cascade/config` | Fetch cascade agent configuration thresholds. |
| `PUT` | `/api/v1/cascade/config` | Update thresholds. Body: `{delay_trigger_minutes, cascade_prob_threshold, auto_reassign_enabled}`. Requires admin role. |
| `GET` | `/api/v1/cascade/events` | Historical cascade event log with outcomes. Filterable by date, driver, outcome. |

### State Management

- **In-memory driver delay state** (service instance memory + Redis for HA failover): Each active driver's current delay, last known position, and cascade status maintained as a `DriverDelayState` dataclass. Serialised to Redis key `cascade:driver:{driver_id}` every 30 seconds for failover recovery.
- **Redis cascade alert set** (`cascade:active`): Set of driver IDs currently in cascade state. Used for dashboard polling and alert deduplication.
- **Redis outreach throttle** (`cascade:outreach:{recipient_id}`): TTL key set when SMS is sent. Prevents duplicate cascade outreach within 4-hour window.
- **PostgreSQL `cascade_events`**: Immutable log of all cascade detection and resolution events.
- **Kafka consumer group** (`cascade-agent-cg`): Stateful consumer on `gps.driver.positions` and `delivery.stop.events`. Committed offsets in Kafka. Consumer group lag monitored via Datadog.

### Dependencies

- Apache Kafka / AWS MSK (consumer: `gps.driver.positions`, `delivery.stop.events`; producer: dispatcher alert events)
- SMS Engine (internal service call to trigger outreach SMS)
- Route Optimizer Service (call to execute reassignment)
- PostgreSQL (`cascade_events`, `route_stops`, `routes`, `drivers`, `deliveries`)
- Redis (driver state, alert set, outreach throttle)
- WebSocket server in `api-gateway` (publishes dispatcher triage notifications)
- Datadog (metrics: `cascade.active_drivers`, `cascade.stops_at_risk`, `cascade.outreach_sent`, `cascade.reassignment_count`, `cascade.prevented_failures`)

---

## Module 5: Delivery Intelligence Dashboard

### Purpose

The Delivery Intelligence Dashboard is the primary operational interface for dispatchers and operations managers. It provides a unified real-time view of pre-dispatch risk, live route performance, and predictive FTD outcomes — giving dispatchers the information they need to intervene before failures happen, not after.

### Key Functions

**Pre-Dispatch Panel (visible from manifest upload until routes depart)**
- Display today's manifest with risk score overlay: each parcel listed with risk tier colour coding (green/yellow/orange/red), top risk factors, recommended intervention.
- Aggregate risk summary: count of HIGH/CRITICAL parcels, estimated FTD rate for today's manifest based on risk distribution, projected re-delivery cost.
- SMS status board: count of outreach sent, replied, confirmed, pending, opted out.
- Risk score drill-down: click any parcel to see full feature breakdown, SHAP contributions, and conversation history.
- Bulk action panel: select multiple HIGH-risk parcels and batch-trigger manual review, SMS, or time window adjustment.

**Live Operations Map (visible once routes depart)**
- Mapbox GL JS map with 35 driver markers, colour-coded by current status (on time / delayed / cascade alert / offline).
- All route stops plotted with colour-coded status (pending / completed / failed / at-risk).
- Driver detail panel on click: current stop, delay minutes, remaining stop count, estimated shift completion time.
- Cascade alert sidebar: active cascade situations with priority ranking, triage options, one-click approval buttons.
- Risk heatmap layer: toggle to show address-level FTD risk heatmap (postcode-level colour gradient from historical data).

**FTD Prediction vs. Actual (end-of-day and historical)**
- Today's running FTD rate (updates in real time as deliveries complete).
- Predicted FTD rate from pre-dispatch risk scoring vs. actual developing rate — gap narrows throughout the day.
- Failure reason breakdown: pie chart of not_home / access_failure / wrong_address / refused / out_of_time.
- SLA breach tracker: count of breaches vs. contract thresholds, by client.

**Driver Performance Overlay**
- Per-driver FTD rate (rolling 30 days) vs. fleet average.
- Driver heatmap: zones where each driver has high vs. low success rates (informs future route assignment).
- On-time arrival rate, average stop completion time, cascade events caused.

**Weekly and Monthly Intelligence Reports**
- FTD trend chart: week-over-week improvement since platform deployment.
- SMS engagement funnel: sent → delivered → replied → positive outcome.
- Model accuracy: predicted risk tier distribution vs. actual outcome distribution.
- Client SLA compliance calendar heat map.

### API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/dashboard/pre-dispatch/{date}` | Pre-dispatch summary: risk distribution, SMS status counts, manifest overview. |
| `GET` | `/api/v1/dashboard/manifest/{manifest_id}/risk` | Full risk-annotated parcel list for a manifest. Supports sort/filter by risk tier, postcode, client. Paginated (50 per page). |
| `GET` | `/api/v1/dashboard/live` | Live operations state: all active drivers with positions, delays, stop counts, cascade flags. |
| `GET` | `/api/v1/dashboard/ftd/today` | Real-time FTD metrics for today: attempted, succeeded, failed, failure_reasons breakdown, current rate. |
| `GET` | `/api/v1/dashboard/ftd/history` | Historical FTD rate by day. Query: `?from=2026-01-01&to=2026-03-14&granularity=day`. |
| `GET` | `/api/v1/dashboard/drivers` | Driver performance summary: all drivers with FTD rates, on-time rates, stop counts for the date range. |
| `GET` | `/api/v1/dashboard/drivers/{driver_id}` | Individual driver performance detail and zone heatmap data. |
| `GET` | `/api/v1/dashboard/sla/{client_id}` | SLA compliance for a specific client: breaches, penalties, trend. |
| `GET` | `/api/v1/dashboard/sla/summary` | SLA compliance across all clients for a date range. |
| `GET` | `/api/v1/dashboard/model-accuracy` | Risk model accuracy metrics: AUC-ROC, precision/recall at HIGH threshold, by week. |
| `WebSocket` | `/ws/dashboard` | Real-time push channel for dispatcher dashboard. Messages: `driver_position_update`, `stop_completed`, `cascade_alert`, `route_updated`, `sms_reply_received`. |

### State Management

- All dashboard data served from PostgreSQL read replica (separate RDS read replica instance; prevents analytical queries from impacting OLTP write performance).
- WebSocket connection state managed by `api-gateway` service using an in-process connection registry (suitable for single-instance; scale-out requires Redis Pub/Sub fan-out for multi-instance).
- Redis cache for pre-dispatch manifest summaries: key `dashboard:predispatch:{date}:{manifest_id}`, 5-minute TTL, invalidated on new risk score batch completion.
- Frontend Zustand store holds: selected driver, selected stop, active panels, map viewport state, filter state (risk tier filter, client filter, date).
- React Query cache holds: manifest risk list (stale after 2 min during pre-dispatch, 30 seconds during live ops), driver list (stale 10 seconds during live ops).

### Dependencies

- All backend microservices (reads from their respective PostgreSQL tables via read replica)
- WebSocket server in `api-gateway`
- Mapbox GL JS (frontend, Mapbox token from `VITE_MAPBOX_TOKEN` env var)
- PostgreSQL read replica
- Redis (dashboard cache)
- Datadog RUM (Real User Monitoring for frontend performance tracking)

---

## Module 6: Continuous Learning Loop

### Purpose

The Continuous Learning Loop closes the feedback cycle between the platform's predictions and real-world outcomes. It ensures the risk scoring model improves with every operating day — learning from each delivery success and failure, detecting when the model's accuracy is degrading, and autonomously retraining and promoting improved models. Without this module, the risk scoring model would be a static snapshot that degrades as delivery patterns, address stock, and recipient behaviour evolve.

### Key Functions

- **Outcome Capture**: At the moment a driver marks a stop as completed or failed, write the outcome and reason to `delivery_attempts`. A PostgreSQL trigger (`capture_training_record`) joins the new `delivery_attempts` record with the corresponding `feature_log` entry (matched by `delivery_id`) and inserts a labelled training record into `training_data` table: `{features: JSONB, label: int (0=success, 1=failure), failure_reason: varchar, delivery_date}`.
- **Feature Drift Detection (Hourly)**: `DriftDetector.run()` computes the mean absolute deviation of each feature's distribution over the last 7 days versus the model's training period distribution (stored in `model_registry.training_feature_stats`). If any feature's MAD exceeds `DRIFT_THRESHOLD` (default 0.2), publish a Datadog event and alert on-call ML engineer.
- **Score Distribution Monitoring (Daily)**: Compare today's predicted risk score distribution (mean, std, percentile bands) with the rolling 30-day distribution stored in `model_accuracy_log`. Sudden shift (>10% change in mean or >15% in std) triggers an alert.
- **Accuracy Monitoring (Daily, post-operations)**: After 8:00pm when >95% of deliveries are completed, compute: Precision at HIGH tier threshold, Recall at HIGH tier threshold, AUC-ROC on today's labelled outcomes. Write to `model_accuracy_log`. Publish to Datadog dashboard metric `risk_model.daily_auc_roc`.
- **Nightly Retraining (2:00am AEST, Celery Beat)**: Pull last 90 days of labelled training data from `training_data` table. Train new XGBoost model with Optuna hyperparameter search (50 trials, 10-fold CV, optimising AUC-ROC). If new model meets promotion criteria (see below), write artifact to S3 and register in `model_registry` as `status = candidate`.
- **Model Promotion Logic**: Auto-promote candidate to active if new model AUC-ROC > current active AUC-ROC + 0.005 AND precision at HIGH threshold >= 0.65 AND no feature drift alerts active. Require manual approval if improvement is <0.005 (marginal gain). Reject and alert if new model AUC-ROC < current - 0.01 (regression).
- **A/B Testing Framework**: Support routing a configurable percentage (1–50%) of scoring requests to a `candidate` model version. Log which model version scored each parcel in `risk_scores.model_version`. After 7 days of A/B operation, compute statistical significance of outcome difference between groups (two-proportion z-test). If p < 0.05 and candidate is better, auto-promote.
- **Address-Level Performance Tracking**: Nightly job updates `addresses.ftd_rate_30d` by computing the rolling 30-day actual FTD rate at each address with >5 delivery attempts. This column is a direct input to the risk scoring feature set.
- **Driver Performance Tracking**: Similar nightly job updates `drivers.ftd_rate_overall`, `drivers.ftd_rate_apartment`, `drivers.ftd_rate_residential`, `drivers.ftd_rate_commercial` from `delivery_attempts` outcomes.
- **Postcode FTD Rate Cache**: Nightly aggregation writes postcode-level FTD rates to `postcode_ftd_rates` table and seeds Redis cache (key `ftd_rate:postcode:{postcode}`, TTL 25 hours) used by Feature Engineering Service.

### API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/learning/accuracy` | Current model accuracy metrics and 30-day trend. |
| `GET` | `/api/v1/learning/accuracy/history` | Historical daily accuracy log. Query: `?from=&to=&model_version=`. |
| `GET` | `/api/v1/learning/drift` | Current feature drift status: per-feature MAD values, drift flags, last check time. |
| `POST` | `/api/v1/learning/retrain` | Manually trigger a retraining run outside the nightly schedule. Requires admin role. Returns `{job_id}`. |
| `GET` | `/api/v1/learning/retrain/{job_id}` | Poll retraining job status. |
| `GET` | `/api/v1/learning/models` | List all model versions: version, training date, training data size, AUC-ROC, status (active/candidate/retired). |
| `POST` | `/api/v1/learning/models/{version}/promote` | Manually promote a candidate model to active. Requires admin role. |
| `POST` | `/api/v1/learning/models/{version}/retire` | Retire a model version. Updates `model_registry.status = retired`. |
| `GET` | `/api/v1/learning/ab-test/results` | Current A/B test results: traffic split, outcome rates per version, statistical significance. |
| `GET` | `/api/v1/learning/training-data/stats` | Training dataset statistics: total records, class balance, date range, feature completeness rates. |

### State Management

- **PostgreSQL `training_data`**: Append-only table. Schema: `{id UUID, delivery_id UUID, delivery_date DATE, features JSONB, label INT, failure_reason VARCHAR, model_version_scored VARCHAR, created_at TIMESTAMPTZ}`. Partitioned by `delivery_date` (monthly partitions) for efficient rolling-window queries. Retention: 365 days (older records archived to S3 in Parquet format for potential future retraining).
- **PostgreSQL `model_accuracy_log`**: Daily accuracy record per model version. Append-only.
- **PostgreSQL `model_registry`**: Current state of all model versions. Columns: `version VARCHAR, status VARCHAR (active/candidate/retired), training_started_at, training_completed_at, training_data_size_rows, auc_roc DECIMAL(5,4), precision_high DECIMAL(5,4), recall_high DECIMAL(5,4), artifact_s3_path, training_feature_stats JSONB, promoted_at, promoted_by`.
- **S3 model artifact store** (`s3://odocs-models/risk-scoring/{version}/`): `model.joblib`, `feature_names.json`, `training_config.json`, `optuna_study.db`. Versioned by timestamp: `v20260314_0200`.
- **MLflow tracking server** (self-hosted ECS task, metadata in PostgreSQL, artifacts in S3): Each Optuna trial logged as an MLflow run. Enables experiment comparison UI.
- **Celery Beat schedule** (defined in `learning_loop_service/celery_config.py`): `retrain_nightly` at 02:00 AEST, `accuracy_check_daily` at 20:30 AEST, `drift_check_hourly` every hour, `update_address_ftd_rates` at 01:00 AEST, `update_driver_performance` at 01:30 AEST, `seed_postcode_cache` at 01:45 AEST.

### Dependencies

- PostgreSQL (`training_data`, `model_accuracy_log`, `model_registry`, `delivery_attempts`, `feature_log`, `addresses`, `drivers`, `postcode_ftd_rates`)
- S3 (model artifact storage and Parquet archive)
- MLflow (experiment tracking)
- Optuna (hyperparameter search)
- XGBoost / scikit-learn (model training)
- Redis (postcode FTD rate cache; Celery broker)
- Celery Beat (scheduled job triggers)
- Risk Scoring Engine (model hot-swap after promotion)
- Datadog (metrics: `learning.daily_auc_roc`, `learning.drift_score`, `learning.training_data_size`, `learning.retraining_duration_minutes`)
- PagerDuty (alert on model degradation, drift detection, retraining failure)
