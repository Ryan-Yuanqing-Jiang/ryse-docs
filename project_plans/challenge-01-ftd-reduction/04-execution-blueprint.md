# Challenge 1: Real-Time Route Intelligence & FTD Reduction — Execution Blueprint

---

## Phase 1: Foundation (Weeks 1–4)

### Goals

Stand up the complete data infrastructure, GPS streaming pipeline, and a working v0 risk scoring model trained on historical data. By end of Phase 1, the dispatcher can see per-parcel risk flags on a pre-dispatch manifest screen, and the engineering team has a tested data pipeline that all subsequent AI modules depend on.

No production delivery decisions are changed by Phase 1 output — this is instrument-first, intervene-second.

---

### Atomic Tasks

**Week 1: Infrastructure and Database**

1. **Provision AWS infrastructure via Terraform.** Create `infra/terraform/` directory. Define the following resources in separate `.tf` files: `vpc.tf` (VPC, 3 public + 3 private subnets across ap-southeast-2a/b/c, NAT gateway), `rds.tf` (PostgreSQL 16 RDS Multi-AZ instance, `db.t3.large`, 100GB gp3, automated backups 7 days, read replica in ap-southeast-2b), `timescaledb.tf` (separate RDS PostgreSQL 16 instance with TimescaleDB 2.x extension, `db.t3.medium`), `elasticache.tf` (Redis 7 ElastiCache cluster, two nodes `cache.t3.medium`, one for feature store / API cache, one for Celery), `msk.tf` (AWS MSK Kafka 3.6 cluster, 3 brokers, `kafka.t3.small`, 100GB EBS per broker), `ecr.tf` (one ECR repository per service: `risk-scoring-service`, `sms-service`, `route-optimizer-service`, `cascade-agent-service`, `learning-loop-service`, `api-gateway`), `ecs.tf` (ECS cluster, Fargate task definitions for each service), `s3.tf` (buckets: `odocs-models`, `odocs-training-data`, `odocs-manifests`, `odocs-reports`), `secrets.tf` (Secrets Manager entries: `prod/db`, `prod/redis`, `prod/kafka`, `prod/twilio`, `prod/google_maps`, `prod/openai`). Run `terraform init && terraform plan` against a staging workspace. Do not apply to production until Week 2.

2. **Create PostgreSQL schema.** In `services/shared/db/migrations/`, initialise Alembic: `alembic init alembic`. Write initial migration `alembic/versions/001_initial_schema.py` creating all tables defined in `02-system-architecture.md`: `clients`, `depots`, `drivers`, `vehicles`, `addresses`, `apartment_complexes`, `recipients`, `recipient_preferences`, `manifests`, `deliveries`, `delivery_attempts`, `risk_scores`, `feature_log`, `routes`, `route_stops`, `sms_conversations`, `sms_messages`, `ftd_events`, `cascade_events`, `training_data`, `model_registry`, `model_accuracy_log`, `postcode_ftd_rates`. Include all indexes defined in the architecture doc. Write a second migration `002_timescaledb_setup.py` on the TimescaleDB instance creating the `gps_positions` hypertable: `SELECT create_hypertable('gps_positions', 'time', chunk_time_interval => INTERVAL '1 day')`. Run migrations against staging RDS.

3. **Set up shared Python service boilerplate.** Create `services/shared/` directory with: `db/session.py` (SQLAlchemy async engine using `asyncpg` driver, connection pool size 10, timeout 30s; reads `DATABASE_URL` from env), `db/models.py` (SQLAlchemy ORM models for all tables), `cache/redis_client.py` (aioredis client, reads `REDIS_URL` from env), `kafka/producer.py` (confluent-kafka `Producer` wrapper with auto-retry and dead-letter logging), `kafka/consumer.py` (confluent-kafka `Consumer` wrapper with offset commit management), `auth/jwt.py` (JWT decode middleware, reads `JWT_SECRET` from env), `logging/datadog.py` (structlog configuration with Datadog JSON format), `config.py` (Pydantic `BaseSettings` class reading all env vars). All services import from `services/shared/` via a local package install (`pip install -e ../shared`).

4. **Bootstrap GitHub Actions CI pipeline.** Create `.github/workflows/ci.yml`. Steps: `checkout`, `setup-python 3.12`, `pip install -e services/shared`, `pip install -r services/{service}/requirements.txt`, `pytest services/{service}/tests/ --cov --cov-report=xml`, `docker build`, `docker push ECR` (on `main` branch only). Run pipeline and confirm it completes on a skeleton service.

**Week 2: GPS Streaming Pipeline**

5. **Create Kafka topics.** Write `infra/kafka/topics.sh`: use `kafka-topics.sh` CLI against MSK to create: `gps.driver.positions` (6 partitions, replication factor 3, retention 24 hours), `delivery.stop.events` (6 partitions, retention 7 days), `route.optimizer.events` (3 partitions, retention 7 days), `sms.events` (3 partitions, retention 7 days). Partition count set to match expected throughput: 35 drivers × 1 msg/sec × safety factor 10 = 350 msg/sec, well within 6-partition capacity.

6. **Build GPS ingestion endpoint in `api-gateway` service.** Create `services/api-gateway/` FastAPI app. Install: `fastapi`, `uvicorn[standard]`, `confluent-kafka`, `pydantic`. Implement `POST /api/v1/gps/position` endpoint. Request body: `GPSPositionPayload(driver_id: UUID, lat: float, lng: float, heading: float, speed: float, accuracy: float, timestamp: datetime)`. Validate lat/lng bounds (within Southeast Queensland bounding box: lat -28.5 to -26.5, lng 152.5 to 154.0). Publish to Kafka topic `gps.driver.positions` with key `str(driver_id)`. Write to TimescaleDB `gps_positions` table asynchronously (fire-and-forget via background task). Return `{status: "ok"}` in <10ms. Add Datadog APM instrumentation: `ddtrace-run uvicorn`. Write unit tests in `services/api-gateway/tests/test_gps_endpoint.py` covering: valid payload, out-of-bounds coordinates rejected, missing fields rejected, Kafka publish called once.

7. **Build driver mock GPS simulator.** Create `tools/gps-simulator/simulate.py`. Script takes a route file (JSON array of lat/lng waypoints) and a driver ID, interpolates positions at 2-second intervals, and POSTs to `POST /api/v1/gps/position`. Runs 35 concurrent simulated drivers using `asyncio`. Use this throughout development to generate realistic GPS streams without needing physical driver devices. Store sample route files in `tools/gps-simulator/routes/brisbane_route_01.json` through `brisbane_route_35.json` (generate from Google Maps API, 80–100 stops each, covering Logan, Northside, Bayside, Inner South zones).

8. **Implement GPS position consumer in TimescaleDB writer.** In `services/learning-loop-service/consumers/gps_consumer.py`, implement a Kafka consumer (consumer group `timescale-writer-cg`) for `gps.driver.positions`. Batch-insert to TimescaleDB `gps_positions` table using `COPY` via `asyncpg` for throughput (target: 1,000 inserts/second). Commit Kafka offsets every 5 seconds or 1,000 messages. Write integration test that publishes 1,000 GPS events and confirms they appear in TimescaleDB within 10 seconds.

9. **Set up Datadog monitoring baseline.** Install Datadog agent on ECS tasks via sidecar container. Create Datadog dashboard `FTD Platform - Infrastructure` with widgets: Kafka consumer lag per topic, RDS connection count and query latency, Redis memory usage, ECS task CPU/memory per service. Set alert: Kafka consumer lag >10,000 messages → PagerDuty P2.

**Week 3: Historical Data Import and Feature Engineering**

10. **Build manifest ingest API.** In `api-gateway`, implement `POST /api/v1/manifests` endpoint. Accept JSON payload: `{client_id, delivery_date, parcels: [{recipient_name, address_raw, phone, parcel_type, client_reference, window_start?, window_end?}]}`. Authenticate via `x-api-key` header (client API key, stored in `clients.api_key` with bcrypt hash). For each parcel: normalise phone to E.164 via `phonenumbers` library, upsert `recipients` by `phone_e164`, geocode address via Google Maps API (cache in `addresses.geocode_cache`), create `deliveries` record. Return `{manifest_id, parcel_count, geocode_failures: []}`. Target latency: <5 seconds for 500-parcel manifest.

11. **Import 90 days of historical delivery data.** Work with the operator to export historical records from their current routing software (likely CSV export of completed delivery records with address, date, outcome). Write a one-time import script `tools/data-import/import_historical.py` that: reads the CSV, geocodes each address via Google Maps API (rate-limited to 10 req/sec), classifies address type using a simple heuristic (unit/apt/level in address string → apartment; street address → residential), creates `addresses` and `recipients` records, creates `deliveries` and `delivery_attempts` records with outcome and failure_reason mapped from the CSV's outcome column to our enum values. Log geocoding failures to `tools/data-import/geocode_failures.csv` for manual review. Target: import 90 days × ~3,000 parcels/day = 270,000 records. Expect 5–8% geocoding failures on first pass; manually resolve the top 50 most common failure patterns.

12. **Build Feature Engineering Service.** Create `services/risk-scoring-service/feature_engineering.py`. Implement `FeatureBuilder` class with methods:
    - `build_address_features(address_id, delivery_date)`: Query `delivery_attempts` for last 90 days at this address. Compute: `address_ftd_rate_90d`, `address_delivery_count_90d`, `is_apartment` (bool), `failure_mode_not_home_pct`, `failure_mode_access_pct`.
    - `build_recipient_features(recipient_id)`: Query `sms_conversations` and `delivery_attempts`. Compute: `recipient_engagement_rate`, `recipient_prior_failures`, `recipient_confirmed_window_pct`.
    - `build_temporal_features(delivery_date, window_start, window_end)`: Compute: `day_of_week` (0–6), `is_weekend` (bool), `window_hour_start` (int), `window_hour_end` (int), `is_school_day` (bool, use Queensland school terms calendar in `data/qld_school_terms.json`), `is_public_holiday` (bool, use `data/au_public_holidays.json`).
    - `build_driver_features(driver_id, address_type)`: Query `drivers` table. Return: `driver_ftd_rate_overall`, `driver_ftd_rate_{address_type}`, `driver_experience_months`, `driver_deliveries_today` (from today's route).
    - `build_postcode_features(postcode)`: Lookup `postcode_ftd_rates` table (or Redis cache). Return: `postcode_ftd_rate_30d`, `postcode_delivery_volume_30d`.
    - `build_weather_features(postcode, delivery_date)`: Query `weather_cache` Redis key `weather:{postcode}:{date}` (populated at 5:30am by daily weather fetch job). Return: `rain_probability`, `temperature_max`.
    - `assemble_vector(delivery_id)`: Call all build methods, return `np.ndarray` of shape (47,) and `feature_names: list[str]`.

**Week 4: Risk Scoring Model v0**

13. **Train baseline GBDT model v0.** Create `services/risk-scoring-service/training/train_v0.py`. Load all 270,000 historical records from `training_data` table (or construct them from `delivery_attempts` + address/recipient features). Features: use `FeatureBuilder.assemble_vector()` for each record. Label: `1` if `outcome != 'success'` else `0`. Expected class imbalance: ~87% success, ~13% failure — use `scale_pos_weight = 87/13 ≈ 6.7` in XGBoost config to address. Split 80/20 train/test by `delivery_date` (NOT random split — use temporal split to prevent data leakage; test set = last 18 days of the 90-day window). Train XGBoost with initial config: `n_estimators=300, max_depth=6, learning_rate=0.05, subsample=0.8, colsample_bytree=0.8, scale_pos_weight=6.7`. Evaluate: AUC-ROC, precision/recall at various thresholds, confusion matrix. Expected baseline AUC-ROC: 0.70–0.75 on first run. Log to MLflow. Save artifact to `s3://odocs-models/risk-scoring/v0_initial/model.joblib`. Insert record in `model_registry` table with `status = active`.

14. **Build Risk Scoring Service inference API.** Create `services/risk-scoring-service/main.py`. Implement:
    - Model loading on startup: read active model version from `model_registry`, download `model.joblib` from S3 via `boto3`, load into memory with `joblib.load()`. Store as module-level singleton. Log model version and AUC-ROC to stdout on startup.
    - `POST /api/v1/risk/score/single`: Accept `{delivery_id: UUID}`. Call `FeatureBuilder.assemble_vector(delivery_id)`. Run `model.predict_proba([vector])[0][1]`. Multiply by 100 for score. Compute SHAP values via `shap.TreeExplainer(model).shap_values(vector)`. Map to top 3 factors. Write `risk_scores` record to PostgreSQL. Return `RiskScore` response. Total latency target: <200ms.
    - `POST /api/v1/risk/score/batch`: Accept `{manifest_id: UUID}`. Enqueue Celery task `score_manifest_batch`. Return `{job_id}`. Celery task fetches all `delivery_ids` for the manifest, calls `FeatureBuilder` in parallel (ThreadPoolExecutor with 8 workers for DB-bound feature fetching), runs batch `model.predict_proba()`, writes all `risk_scores` records, marks job complete.
    - Install: `xgboost==2.1`, `shap==0.45`, `joblib`, `boto3`, `celery[redis]`.

15. **Build pre-dispatch risk dashboard (read-only v0).** In the React frontend (`frontend/src/`), create `pages/PreDispatch.tsx`. Components:
    - `ManifestRiskSummary`: Shows total parcels, count by risk tier (colour-coded chips), estimated FTD rate, projected re-delivery cost (use `count_high_risk × 0.65 × $10` as placeholder calculation).
    - `ParcelRiskTable`: Table with columns: Client Ref, Address, Recipient, Risk Score (badge), Risk Tier (colour), Top Risk Factors (tooltip with SHAP factors), Recommended Action. Sortable by risk score. Filterable by tier.
    - `RiskScoreHistogram`: Recharts histogram of risk score distribution for the day's manifest.
    - Wire to `GET /api/v1/dashboard/manifest/{manifest_id}/risk`. Use React Query with 60-second stale time.
    - No action buttons yet — read-only display only.

16. **End-to-end test of Phase 1 pipeline.** Upload a 500-parcel test manifest via the ingest API. Verify: all parcels geocoded, `deliveries` records created, risk scoring batch job runs, `risk_scores` records written, pre-dispatch dashboard displays risk tiers and SHAP factors. Measure end-to-end latency from manifest upload to risk scores visible on dashboard. Target: <90 seconds.

### Key Technical Decisions

- **Temporal train/test split** (not random): Non-negotiable. Random splits for time-series data create data leakage (future patterns leaking into training set) and produce falsely optimistic accuracy metrics. All model evaluation uses a held-out test set from the most recent dates in the training window.
- **Feature logging at inference time**: The `feature_log` table captures the exact feature vector used to generate each score. This is critical for the learning loop — if the feature engineering logic changes between a scoring run and the later outcome, we must train on the features that were actually used, not recomputed features.
- **Avoid geocoding the same address twice**: The `addresses` table with `geocode_cache` column is the canonical geocode store. Every manifest ingest checks for address match before calling the Google Maps API. Match logic: normalise string (lowercase, strip punctuation, expand "St" → "Street" etc.) and use PostgreSQL `pg_trgm` trigram similarity matching with threshold 0.85.

### Definition of Done

- [ ] All Terraform resources provisioned in staging environment; `terraform plan` shows no changes.
- [ ] All database migrations applied; schema matches `02-system-architecture.md` exactly.
- [ ] GPS simulator running 35 concurrent drivers; positions visible in TimescaleDB.
- [ ] 270,000 historical delivery records imported; geocoding failure rate <10%.
- [ ] Risk scoring model v0 AUC-ROC >= 0.68 on temporal test set.
- [ ] Batch scoring API processes 500-parcel manifest in <90 seconds end-to-end.
- [ ] Pre-dispatch dashboard displaying risk scores, tiers, and SHAP factors.
- [ ] CI pipeline passing on all services.
- [ ] Datadog infrastructure dashboard live with no open P1 alerts.

### Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Historical data quality is poor (inconsistent outcome codes, missing addresses) | High | High | Allocate dedicated data cleaning sprint in Week 3. Build `data_quality_report.py` tool to surface issues before import. Accept 15% data loss as acceptable if patterns are consistent. |
| Google Maps geocoding API costs spike during 270,000-record import | Medium | Low | Use `GEOCODING_BATCH_MODE=true` env flag to run historical import at 5 req/sec (vs. 10 for live). Estimated cost: $270,000 × $0.005 = $1,350 one-time. Pre-approve budget. |
| XGBoost model AUC-ROC below 0.68 on first run | Medium | Medium | If below threshold: (1) review feature completeness (null rates per feature), (2) check temporal split boundary for concept drift, (3) try LightGBM as alternative. Do not proceed to Phase 2 with model below 0.65. |
| MSK Kafka cluster provisioning takes longer than expected | Low | Medium | Pre-provision MSK in Week 1. Use a local Kafka (Docker Compose) as fallback for development in Weeks 1–2. |

---

## Phase 2: Core AI (Weeks 5–8)

### Goals

Activate the production risk scoring model with real features (live data, not just historical import), build the full Two-Way SMS Engine, and integrate both into the pre-dispatch workflow. By end of Phase 2, the platform is actively preventing deliveries from failing: high-risk parcels are identified before dispatch, recipients are contacted, and confirmed time windows are locked into the route plan.

---

### Atomic Tasks

**Week 5: Production Feature Engineering**

17. **Build daily weather fetch job.** Create `services/learning-loop-service/jobs/fetch_weather.py`. Celery Beat task `fetch_weather_daily` scheduled at 05:30 AEST. Queries Bureau of Meteorology JSON API (`http://www.bom.gov.au/fwo/IDQ60901/IDQ60901.94575.json` for Brisbane region) and OpenWeatherMap API (`https://api.openweathermap.org/data/2.5/forecast`) for each postcode cluster (group 580 Southeast Queensland postcodes into 12 weather zones). Writes to Redis key `weather:{postcode}:{date}` with TTL of 25 hours. Writes backup to `weather_cache` PostgreSQL table. Test: mock API responses in unit test, verify Redis write and correct postcode grouping.

18. **Build Queensland school terms and public holidays data files.** Create `services/shared/data/qld_school_terms.json` with term start/end dates for 2025–2027 (source: Queensland Department of Education website, manually transcribed). Create `services/shared/data/au_public_holidays.json` with Queensland public holidays for 2025–2027 (source: qld.gov.au). Load both files as module-level constants in `services/shared/calendars.py`. Expose `is_school_day(date: date) -> bool` and `is_public_holiday(date: date) -> bool` functions.

19. **Add postcode FTD rate nightly aggregation job.** Create `services/learning-loop-service/jobs/update_postcode_rates.py`. Celery Beat task `update_postcode_ftd_rates` at 01:00 AEST. SQL: `INSERT INTO postcode_ftd_rates (postcode, ftd_rate_30d, delivery_count_30d, computed_at) SELECT a.postcode, COUNT(CASE WHEN da.outcome != 'success' THEN 1 END)::float / COUNT(*), COUNT(*), NOW() FROM delivery_attempts da JOIN deliveries d ON d.id = da.delivery_id JOIN addresses a ON a.id = d.address_id WHERE da.attempted_at >= NOW() - INTERVAL '30 days' AND da.attempt_number = 1 GROUP BY a.postcode ON CONFLICT (postcode) DO UPDATE SET ftd_rate_30d = EXCLUDED.ftd_rate_30d, computed_at = EXCLUDED.computed_at`. Seed Redis cache for each postcode: `redis.set(f"ftd_rate:postcode:{postcode}", rate, ex=86400)`.

20. **Run full feature completeness audit on 47 features.** Create `tools/feature-audit/audit.py`. For the last 30 days of historical records, compute null rate per feature. Any feature with >20% null rate is flagged. For flagged features: check if nullability is expected (e.g., `recipient_engagement_rate` is null for first-time recipients) — if so, add an `is_first_time_recipient` binary feature and impute with 0.5 (neutral prior). If null rate is unexpected (e.g., `postcode_ftd_rate_30d`), fix the underlying data job. Document all imputation decisions in `services/risk-scoring-service/FEATURE_SPEC.md`.

**Week 6: GBDT Model v1 with Full Feature Set**

21. **Retrain model v1 on full 47-feature set with live data.** Create `services/risk-scoring-service/training/train_v1.py`. Extend `train_v0.py`: add weather features, school day/holiday flags, postcode FTD rate, recipient engagement rate (now computed from live data for the last 90 days). Run Optuna hyperparameter search: 50 trials, search space `{n_estimators: [100, 500], max_depth: [3, 8], learning_rate: [0.01, 0.1], subsample: [0.6, 1.0], colsample_bytree: [0.6, 1.0], min_child_weight: [1, 10]}`. Use 5-fold time-series cross-validation (sklearn `TimeSeriesSplit`). Objective: maximise AUC-ROC. Log all trials to MLflow. Best model expected AUC-ROC: 0.73–0.78. Calibrate probabilities with `CalibratedClassifierCV(method='sigmoid')`. Save to `s3://odocs-models/risk-scoring/v1_{timestamp}/model.joblib`.

22. **Validate model v1 and promote via A/B test.** Set A/B split: 20% of scoring traffic to v1, 80% to v0. Run for 3 operating days (minimum 9,000 scored deliveries per version). Use `POST /api/v1/risk/model/ab-config` to configure split. After 3 days, call `GET /api/v1/learning/ab-test/results` to review. If v1 AUC-ROC > v0 + 0.02 with p < 0.05, promote v1 to active via `POST /api/v1/learning/models/v1_{timestamp}/promote`.

**Week 6–7: Two-Way SMS Engine**

23. **Set up Twilio account for Australian A2P messaging.** Register for Twilio A2P 10DLC (Australian carrier compliance). Register brand and campaign in Twilio console. Obtain Messaging Service SID. Store `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_MESSAGING_SERVICE_SID` in AWS Secrets Manager key `prod/twilio`. Configure Twilio inbound webhook URL: `https://api.platform.com/webhooks/twilio/inbound`. Configure status callback URL: `https://api.platform.com/webhooks/twilio/status`. Test with a single SMS send and receive.

24. **Build SMS service core.** Create `services/sms-service/`. Install: `twilio>=9.0`, `openai>=1.0`, `phonenumbers`, `celery[redis]`. Implement:
    - `sms_service/composer.py`: `compose_day_before_message(delivery, recipient, window_start, window_end) -> str`. Template: "Hi {name}, your {client_name} delivery is scheduled for tomorrow, {date} between {window_start}–{window_end}. Reply: 1=I'll be home, 2=Need different time, 3=Leave at door/provide access, 4=Pick up from depot. Delivery notification from {operator_name}."
    - `sms_service/composer.py`: `compose_pre_arrival_message(delivery, recipient, eta_time) -> str`. Template with 2-hour ETA.
    - `sms_service/composer.py`: `compose_cascade_message(delivery, recipient, new_eta, options) -> str`. Cascade-specific template.
    - `sms_service/sender.py`: `send_sms(to_phone, body, conversation_id) -> twilio_message_sid`. Checks opt-out Redis set before sending. Checks quiet hours. Creates `sms_messages` record. Calls Twilio API.
    - `sms_service/scheduler.py`: `schedule_day_before(delivery_id)` and `schedule_pre_arrival(delivery_id, eta)`. Create Celery tasks `send_day_before_sms` and `send_pre_arrival_sms` with `eta` (datetime) parameter for delayed execution.

25. **Build inbound reply processor.** Implement `sms_service/reply_processor.py`:
    - `process_inbound(twilio_payload: dict) -> None`: Entry point for webhook.
    - Acquire Redis lock `sms:lock:{conversation_id}` (SETNX, 30s TTL). Return 200 if lock not acquired (duplicate webhook).
    - Find open `sms_conversations` record by `recipients.phone_e164 = twilio_payload['From']` and `status NOT IN ('expired', 'opted_out')`.
    - If no open conversation: check for opt-out keywords (STOP/OPTOUT/UNSUBSCRIBE). If yes, process opt-out. Otherwise, ignore.
    - Call `StructuredParser.parse(body)`. Returns `Intent` dataclass or `None`.
    - If `None`: call `LLMParser.parse(body)`. LLM prompt (in `sms_service/prompts/intent_extraction.txt`): "Extract delivery intent from this customer SMS reply. The customer was asked to confirm their delivery or provide instructions. Return ONLY a JSON object: {\"intent\": \"confirm|reschedule|access_instructions|redirect|unclear\", \"details\": \"extracted time/instructions/null\", \"confidence\": 0.0-1.0}. SMS: {body}".
    - Map intent to action: `confirm` → update `route_stops.status` context, update conversation outcome. `reschedule` → parse new time window from `details`, publish `route.optimizer.events` Kafka message. `access_instructions` → write to `recipient_preferences` and `route_stops.driver_notes`. `redirect` → update delivery status and notify dispatcher.
    - Write `sms_messages` record for inbound message. Update `sms_conversations.status = 'replied'`, `outcome`. Release Redis lock.

26. **Build structured SMS reply parser.** Implement `sms_service/structured_parser.py`. `StructuredParser.parse(body: str) -> Intent | None`:
    - Strip and lowercase the body.
    - Match numeric: `re.match(r'^\s*[1-4]\s*$', body)` → map 1→confirm, 2→reschedule, 3→access_instructions, 4→redirect.
    - Match affirmative keywords: `{'yes', 'yep', 'yeah', 'ok', 'okay', 'sure', 'confirm', 'confirmed', 'will be home', "i'll be there"}` → confirm.
    - Match negative/reschedule: `{'no', 'not home', 'away', "won't be", 'reschedule', 'change', 'different time'}` → reschedule (no details).
    - Match opt-out: `{'stop', 'optout', 'opt out', 'unsubscribe', 'cancel'}` → special opt-out handling.
    - All other inputs: return `None` (triggers LLM fallback).
    - Unit tests: 30+ test cases in `tests/test_structured_parser.py` covering all branches and edge cases (mixed case, trailing whitespace, emoji in message, etc.).

27. **Integrate SMS scheduling into pre-dispatch workflow.** In `api-gateway`, modify the manifest ingest completion flow: after risk scoring batch job completes, automatically call `POST /api/v1/sms/schedule/batch` with the manifest ID. This triggers SMS scheduling for all HIGH/CRITICAL risk parcels. Add env var `SMS_AUTO_SCHEDULE_THRESHOLD=65` — only schedule for parcels above this risk score. Add env var `SMS_REQUIRE_DISPATCHER_APPROVAL=false` for initial rollout (auto-schedule without dispatcher action). Add a toggle on the pre-dispatch dashboard to pause SMS scheduling for a specific manifest if the dispatcher wants to review first.

**Week 8: Integration Testing and First Live Run**

28. **Build SMS conversation view on pre-dispatch dashboard.** Add `SMSStatusPanel` component to `PreDispatch.tsx`. Shows: total SMS scheduled, delivered, replied, confirmed, rescheduled, no-reply, failed delivery. Clickable rows expand to show full conversation thread (inbound + outbound messages). Update route stop `driver_notes` field rendered as a yellow callout banner on each stop in the route view. Wire to `GET /api/v1/sms/conversations` and `GET /api/v1/sms/stats/{date}`.

29. **Soft launch Phase 2 with a single client and 5 drivers.** Select the client with the highest FTD rate for the pilot. Assign 5 drivers. Run for 2 weeks with full Phase 2 features active. Measure: SMS delivery rate (target >95%), reply rate (target >50% for HIGH risk), FTD rate vs. prior 4-week average for the same client/zone. Track LLM fallback rate (target <15% of replies require LLM). Document all edge cases encountered in reply processing for structured parser improvement.

### Key Technical Decisions

- **LLM fallback is last resort**: The structured parser must handle >85% of replies. LLM adds ~300ms and costs money. Only invoke if structured parser returns `None`. Monitor LLM fallback rate daily — if it exceeds 20%, improve the structured parser first.
- **Celery task ETA for SMS scheduling**: Use Celery's built-in `eta` parameter for scheduling future SMS sends rather than cron or APScheduler. This means SMS tasks are durable (survive service restarts) and cancellable by task ID if a delivery is removed or rescheduled before the SMS fires.
- **Idempotent inbound webhook handling**: Twilio retries webhooks if no 200 response received within 15 seconds. The Redis lock + conversation deduplication logic must make repeated webhook calls safe. Test this explicitly with a mock that sends the same webhook 3 times in rapid succession.

### Definition of Done

- [ ] Weather features, school day flags, and postcode FTD rates populated and feeding into feature vectors.
- [ ] Model v1 AUC-ROC >= 0.72 on temporal test set.
- [ ] A/B test completed; v1 promoted to active if criteria met.
- [ ] Twilio account registered for Australian A2P messaging; test SMS send/receive working.
- [ ] Structured reply parser handles >85% of test cases without LLM.
- [ ] Inbound webhook processes reply and updates `route_stops.driver_notes` within 2 seconds of SMS receipt.
- [ ] Pre-dispatch workflow: manifest ingest → risk scoring → SMS scheduling runs end-to-end without manual intervention.
- [ ] 2-week soft launch data collected; FTD improvement measurable.

### Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Twilio A2P 10DLC registration delayed (carrier approval takes 2–4 weeks in Australia) | High | High | Begin Twilio registration in Week 5, parallel with development. Use Twilio test credentials for development. If delayed, launch Phase 2 without SMS and extend SMS launch to Phase 3. |
| Recipients reply in ways the structured parser doesn't handle | High | Medium | Launch with conservative structured parser. Log all LLM-handled replies. After 1 week, review top 20 unhandled patterns and add to structured parser. Continuous improvement cycle. |
| LLM API rate limits during morning batch | Low | Medium | gpt-4o-mini has generous rate limits (10M TPM on Tier 2). At 100 LLM calls/day average, this is trivial. If usage spikes, add Redis-based rate limiter in `LLMParser`. |
| Recipient confusion about opt-out affecting standard delivery | Low | High | Ensure opt-out message is clear: "You've been unsubscribed from proactive delivery notifications. Your parcels will still be delivered." Never opt out a recipient from all communications — only from proactive risk-based SMS. |

---

## Phase 3: Dynamic Routing (Weeks 9–12)

### Goals

Activate dynamic mid-day route re-optimisation and the Cascade Prevention Agent. By end of Phase 3, the platform is a complete real-time operational system: routes adapt to actual conditions throughout the day, cascade failures are detected and interrupted before they happen, and the dispatcher dashboard shows a live operational picture with triage tools.

---

### Atomic Tasks

**Week 9: Route Optimization Service**

30. **Build Route Optimization Service skeleton and OR-Tools VRP wrapper.** Create `services/route-optimizer-service/`. Install: `ortools==9.10`, `googlemaps`, `celery[redis]`, `confluent-kafka`. Implement `vrp/solver.py`:
    - `RoutingProblem` dataclass: `drivers: list[Driver]`, `stops: list[Stop]`, `depot: Depot`, `travel_times: np.ndarray` (n×n matrix of seconds), `time_windows: list[tuple[int,int]]` (seconds from start of day), `capacities: list[int]`.
    - `VRPSolver.solve(problem: RoutingProblem, time_limit_seconds: int = 30) -> RoutingSolution`: Creates `ortools.constraint_solver.RoutingIndexManager` and `RoutingModel`. Adds vehicle capacity dimension. Adds time window constraints. Sets arc cost as `travel_times[from][to]`. Uses `AUTOMATIC` first solution strategy and `GUIDED_LOCAL_SEARCH` metaheuristic. Returns `RoutingSolution(routes: list[list[StopSequence]], total_distance, total_time, solve_status)`.
    - Unit test: build a 10-stop, 3-driver test problem and verify solution is valid (all stops served, no capacity violations, no time window violations).

31. **Build travel time matrix service.** Implement `vrp/travel_times.py`. `TravelTimeMatrix.build(stops: list[Stop], departure_time: datetime) -> np.ndarray`:
    - Partition stops into batches of 10×10 (Google Maps Distance Matrix limit: 100 elements = 10 origins × 10 destinations).
    - Call `googlemaps.distance_matrix(origins, destinations, mode='driving', departure_time=departure_time, traffic_model='best_guess')` for each batch.
    - Extract `duration_in_traffic.value` (seconds) from each element.
    - Cache results in Redis: key `ropt:ttm:{date}:{sorted_stop_ids_hash}`, TTL 30 minutes.
    - Handle API errors: if a cell returns `ZERO_RESULTS` (no route found), set travel time to 99999 seconds (effectively prohibits this arc in the solver).
    - Cost estimate: for 35 drivers × 100 stops avg = 3,500 stops. Full matrix = 3,500² = 12.25M elements. This is impractical. Instead, build per-driver matrices (100 stops × 100 stops = 10,000 elements per driver) at route build time. For re-optimisation, only rebuild the matrix for the affected driver's remaining stops.

32. **Implement risk-weighted route ordering.** In `vrp/route_builder.py`, implement `RiskWeightedRouteBuilder.build_constraints(deliveries, risk_scores) -> RoutingProblem`:
    - For CRITICAL risk parcels with no SMS confirmation: apply a soft preference to sequence them earlier in the route (add a small time penalty for late-day delivery in the cost function).
    - For HIGH risk parcels with SMS-confirmed time windows: apply tight time window constraints (tighter than the original window to ensure driver arrives within the confirmed slot).
    - For CRITICAL risk parcels with SMS confirmation + access instructions: no special ordering needed (access issue resolved).
    - For healthcare parcels with SLA windows: apply hard time window constraints. Set `RoutingModel.AddTimeWindow(node, lower, upper)` with `upper = sla_end_time_seconds`. Solver will fail if it cannot find a solution satisfying all hard constraints — in this case, log infeasibility, relax the constraint for the lowest-priority SLA stop, and alert dispatcher.

33. **Build pre-dispatch route building API and Celery job.** Implement `POST /api/v1/routes/build` endpoint and `build_routes_for_date` Celery task. Flow:
    - Load all `deliveries` for `delivery_date` with `status = 'scheduled'`.
    - Load driver roster and vehicle assignments from `routes` records (if pre-created) or from request payload.
    - Fetch risk scores from `risk_scores` table.
    - Fetch confirmed time windows from `recipient_preferences` (any `preference_type = 'time_window'` for today's deliveries).
    - Build travel time matrices per driver zone (zone assignment: cluster stops geographically using KMeans, assign one driver per cluster).
    - Run `VRPSolver.solve()` per zone with 30-second time limit.
    - Write `route_stops` records with `sequence_order` from solution.
    - Publish `route.built` event to `route.optimizer.events` Kafka topic.
    - Mark job complete, return `{route_ids: []}`.

**Week 10: Real-Time Re-Optimisation**

34. **Build Kafka event consumer for re-optimisation triggers.** In `services/route-optimizer-service/consumers/`, implement `ReoptimiseTriggerConsumer`. Subscribes to `route.optimizer.events` topic. Event handler:
    - `handle_driver_delay(event)`: If `delay_minutes > REOPT_TRIGGER_MINUTES` (env var, default 15) AND remaining stops > 5: check Redis lock `ropt:lock:{route_id}`. If not locked: acquire lock, enqueue `reoptimise_route` Celery task.
    - `handle_stop_failed(event)`: If stop has risk tier HIGH/CRITICAL: check nearby drivers, enqueue `find_reassignment` task.
    - `handle_time_window_update(event)`: Always trigger re-opt for affected driver.
    - `handle_new_stop_added(event)`: If driver has capacity (current stop count < `vehicle.capacity_parcels`): insert stop and trigger soft re-opt.

35. **Implement real-time ETA computation and push.** In `route-optimizer-service`, implement `ETAComputer`. Celery Beat task `compute_etas` runs every 60 seconds. For each active route:
    - Get driver's current GPS position from TimescaleDB: `SELECT lat, lng FROM gps_positions WHERE driver_id = $1 ORDER BY time DESC LIMIT 1`.
    - Get remaining stops in sequence order from `route_stops WHERE status = 'pending' ORDER BY sequence_order`.
    - For each remaining stop: ETA = driver_current_time + travel_time(current_position, next_stop) + sum of travel_times for preceding stops + avg_stop_service_time (2 min default, configurable). avg_stop_service_time from `drivers.avg_stop_time_minutes` column (computed by learning loop).
    - Update `route_stops.predicted_arrival` in batch SQL.
    - Publish ETA update event to WebSocket channel via Redis Pub/Sub (key `ws:route:{route_id}`): `{type: "eta_update", route_id, stops: [{stop_id, predicted_arrival}]}`.

36. **Implement re-optimisation Celery task.** `reoptimise_route(route_id: UUID, trigger_event: str) -> None`:
    - Load remaining unserved stops for this route.
    - Get driver's current GPS position.
    - Rebuild travel time matrix for remaining stops (cache hit likely — 30-min TTL).
    - Build `RoutingProblem` with single driver (current position as depot), remaining stops, updated time windows.
    - Run `VRPSolver.solve(time_limit_seconds=10)` — tighter time limit for in-flight re-opt.
    - Compare new sequence to current sequence. If identical (no improvement found), skip update.
    - If changed: update `route_stops.sequence_order` and `sequence_version` in PostgreSQL (transaction). Publish updated sequence to driver app via WebSocket. Increment `routes.reoptimisation_count`. Log re-opt event to `reoptimisation_log` table.
    - Release Redis lock.

**Week 11: Cascade Prevention Agent**

37. **Build Cascade Prevention Agent service.** Create `services/cascade-agent-service/`. Core loop in `cascade_agent/monitor.py`:
    - Subscribe to `gps.driver.positions` and `delivery.stop.events` Kafka topics (consumer group `cascade-agent-cg`).
    - Maintain in-memory `DriverDelayState` dict: `{driver_id: {current_delay_minutes, last_updated, cascade_status}}`.
    - On GPS event: update `last_position` for driver. Compute delay by comparing actual completed stop count vs. planned schedule.
    - On stop event: update completed count. Recompute delay.
    - Every 60 seconds (Celery Beat `check_cascades` task): for each active driver, evaluate delay against threshold (`CASCADE_TRIGGER_MINUTES=20`). If threshold exceeded and not already in cascade state: set cascade status, call `at_risk_detector.identify_at_risk_stops(driver_id, delay_minutes)`.
    - Persist driver delay state to Redis every 30 seconds: `redis.set(f"cascade:driver:{driver_id}", json.dumps(state), ex=3600)`.

38. **Implement at-risk stop detection and cascade probability scoring.** In `cascade_agent/at_risk_detector.py`:
    - `identify_at_risk_stops(driver_id, delay_minutes) -> list[AtRiskStop]`: Load remaining stops in sequence. For each stop, compute `predicted_arrival` (from `route_stops.predicted_arrival`). Compare against stop's `planned_window_end` or recipient confirmed window. If `predicted_arrival > window_end - 30min`: flag as at-risk.
    - `CascadeProbabilityModel.score(stop, driver_delay, time_of_day) -> float`: Logistic regression model (4 features: `delay_minutes`, `remaining_stops_count`, `window_tightness_minutes`, `stop_is_residential_afternoon`). Train on `cascade_events` table outcomes. Serialised to `s3://odocs-models/cascade-probability/active/model.joblib`. Loaded on service startup.
    - Return list of `AtRiskStop` objects with `cascade_probability`, `window_end`, `stop_type`, `sms_preference`.

39. **Implement automated recipient outreach from cascade agent.** In `cascade_agent/outreach.py`:
    - `CascadeOutreach.send(at_risk_stops: list[AtRiskStop]) -> None`: For each stop with `cascade_probability > CASCADE_SMS_THRESHOLD` (default 0.6):
      - Check Redis key `cascade:outreach:{recipient_id}` (4-hour throttle). Skip if set.
      - Call `POST /api/v1/sms/send/cascade` (internal endpoint in sms-service). Payload: `{delivery_id, message_template: 'cascade', new_eta_time, options: list[str]}`.
      - Set Redis key `cascade:outreach:{recipient_id}` with TTL 14400 (4 hours).
      - Write to `cascade_events` table: `{driver_id, stop_id, delay_minutes, cascade_probability, action: 'outreach_sent', timestamp}`.

40. **Build dispatcher cascade triage UI.** Add `CascadeAlertPanel` to the live operations dashboard. Panel renders on right sidebar when any cascade alert is active. Shows: driver card (name, zone, delay badge), list of at-risk stops (address, cascade probability bar, recipient SMS status), action buttons:
    - "Approve Reassignment" (calls `POST /api/v1/cascade/approve-reassignment`).
    - "Dismiss Alert" (calls `POST /api/v1/cascade/dismiss/{driver_id}`).
    - "View Conversations" (expands inline to show SMS thread for each at-risk stop).
    - Cascade alert badge on driver map marker: flashing orange for warning, red for critical (>4 at-risk stops).

**Week 12: Integration and Full Fleet Go-Live**

41. **Load test the full real-time pipeline.** Use `tools/gps-simulator/simulate.py` to run 35 concurrent simulated drivers with 100 stops each for a full simulated operating day (8 hours compressed to 30 minutes using 16x time acceleration). Measure: Kafka consumer lag (target <500 messages), ETA computation latency (target <5 seconds end-to-end from GPS event to dashboard update), cascade agent detection latency (target <90 seconds from delay onset to alert displayed). Fix bottlenecks before live fleet deployment.

42. **Go-live with full fleet.** Deploy all Phase 3 services to production. Brief all 42 drivers on the driver app changes (updated stop sequence may change mid-shift — add in-app notification: "Your route has been updated. Tap to see new order."). Brief dispatchers on cascade triage panel. Monitor for first 5 operating days with on-call engineer available. Collect FTD metrics daily.

### Key Technical Decisions

- **Event-triggered re-optimisation, not continuous**: Running the VRP solver continuously would be computationally expensive and would push constant route changes to drivers. Re-optimise only when a meaningful event warrants it. The `REOPT_TRIGGER_MINUTES=15` threshold is configurable — set higher (20 minutes) initially to reduce noise, tune down as the fleet gains comfort with route changes.
- **Driver app UX for route changes**: Mid-shift route reordering can be disorienting for drivers. Implement "soft reorder" — show the new sequence with a diff indicator (green up arrow / red down arrow next to each stop showing it moved), not a completely new list. Give drivers 60 seconds to acknowledge before the navigation updates.
- **OR-Tools Python bindings performance**: The Python bindings add overhead. For the initial 35-vehicle fleet, a single ECS task with 2 vCPUs is sufficient. If fleet grows >100 vehicles, consider running OR-Tools in a subprocess via `subprocess.Popen` with a Go/Java wrapper for better GIL management.

### Definition of Done

- [ ] VRP solver builds valid 35-driver route in <60 seconds.
- [ ] Re-optimisation completes in <10 seconds for a single driver's remaining stops.
- [ ] ETA updates pushed to dispatcher dashboard within 5 seconds of driver GPS update.
- [ ] Cascade agent detects delay onset within 60 seconds of trigger event.
- [ ] Cascade outreach SMS sent within 90 seconds of at-risk stop identification.
- [ ] Load test passes: 35 simulated drivers, 3,500 stops, full day, no Kafka consumer lag >1,000.
- [ ] Full fleet go-live completed; no P1 incidents in first 5 operating days.
- [ ] FTD rate measurably below 10% within first 2 weeks of full deployment.

### Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| OR-Tools solver times out on large re-optimisation instances | Medium | Medium | Enforce 10-second hard limit via `RoutingModel.CloseModelWithParameters()`. If solver reaches time limit with a feasible solution, use it. If no feasible solution found, keep current sequence and alert dispatcher. Log all timeout events. |
| Drivers reject mid-shift route changes (UX friction) | Medium | High | Run driver briefing before go-live. Show "before/after" comparison in app. Monitor driver app acknowledgement rate. If <70% of drivers acknowledge route change within 60s, consider adding an audible alert. |
| Cascade agent produces too many false alarms | Medium | Medium | Start with conservative thresholds (`CASCADE_TRIGGER_MINUTES=25`, `CASCADE_SMS_THRESHOLD=0.7`). Review false positive rate daily in first 2 weeks. Tune thresholds down only after establishing baseline false positive rate. |

---

## Phase 4: Intelligence Layer (Weeks 13–16)

### Goals

Activate the Continuous Learning Loop to make the platform self-improving. Implement full A/B testing framework for ongoing model iteration, accuracy monitoring with automated alerts, and the advanced operations intelligence dashboard. By end of Phase 4, the platform requires minimal ML engineering oversight for day-to-day operation — models retrain nightly, promote automatically when improved, and alert the team when degradation is detected.

---

### Atomic Tasks

**Week 13: Outcome Capture and Training Pipeline**

43. **Implement delivery outcome capture trigger.** Create PostgreSQL trigger function `capture_training_record()` in migration `alembic/versions/005_training_capture_trigger.py`. The trigger fires AFTER INSERT on `delivery_attempts`. Trigger body: query `feature_log` for the matching `delivery_id` (find the most recent `feature_log` entry before the `attempted_at` timestamp). Insert into `training_data`: `{delivery_id, delivery_date, features (from feature_log.feature_snapshot), label (0 if outcome='success', 1 otherwise), failure_reason, model_version_scored (from risk_scores.model_version)}`. Test: insert a mock `delivery_attempts` record and verify `training_data` record created within 100ms.

44. **Build nightly retraining Celery task.** In `services/learning-loop-service/jobs/retrain.py`, implement `retrain_risk_model()` Celery task:
    - Pull 90-day training window: `SELECT features, label FROM training_data WHERE delivery_date >= NOW() - INTERVAL '90 days'`.
    - Convert `features` JSONB to numpy array using `feature_names` list from current active model's `model_registry` entry (ensures consistent feature ordering).
    - Temporal train/test split: last 20% of days as test set.
    - Run Optuna study `risk_model_retrain_{timestamp}` with 50 trials. Each trial: train XGBoost with sampled hyperparams, evaluate AUC-ROC on test set via 5-fold `TimeSeriesSplit`. Return AUC-ROC as objective.
    - Best trial: retrain on full training set (excluding test). Calibrate with `CalibratedClassifierCV`.
    - Evaluate on held-out test set: AUC-ROC, precision/recall at HIGH threshold (score>65).
    - Write artifact to `s3://odocs-models/risk-scoring/v{timestamp}/`. Register in `model_registry` with `status = 'candidate'`.
    - Apply promotion logic: if AUC-ROC > active_model.auc_roc + 0.005 AND precision_high >= 0.65: auto-promote (update `is_active` flag, call risk-scoring-service model hot-swap endpoint). Else: require manual review.
    - Log entire run as MLflow experiment run. Send summary to Datadog: `learning.retrain_completed`, `learning.new_model_auc_roc`, `learning.promoted` (bool).

45. **Build feature drift detection.** In `services/learning-loop-service/jobs/drift_detection.py`, implement `DriftDetector.run()` (Celery Beat, hourly):
    - Load active model's `training_feature_stats` from `model_registry` (JSONB column with per-feature mean, std, percentile values from training data, populated at training time).
    - Compute per-feature mean and std from last 7 days of `feature_log` entries.
    - For each feature: compute `population_stability_index` (PSI) comparing training distribution to current distribution. PSI > 0.2 is significant drift.
    - Write results to `feature_drift_log` table: `{feature_name, psi, is_drifting, checked_at}`.
    - If any feature PSI > 0.2: create Datadog event `risk_model.feature_drift_detected`, trigger PagerDuty P2.
    - Dashboard endpoint: `GET /api/v1/learning/drift` returns current PSI values per feature.

**Week 14: Accuracy Monitoring and Model Performance Dashboard**

46. **Build daily accuracy computation job.** In `services/learning-loop-service/jobs/accuracy_check.py`, implement `AccuracyMonitor.compute_daily()` (Celery Beat, 20:30 AEST):
    - Fetch today's completed `delivery_attempts` with outcome (label) and their associated `risk_scores` record.
    - Compute: AUC-ROC (sklearn `roc_auc_score`), Precision at HIGH threshold (score>65), Recall at HIGH threshold, Brier score (calibration quality).
    - Write to `model_accuracy_log`: `{date, model_version, auc_roc, precision_high, recall_high, brier_score, total_scored, total_high_risk_flagged, total_high_risk_actually_failed}`.
    - Publish to Datadog: `risk_model.daily_auc_roc`, `risk_model.daily_precision_high`.
    - Alert if AUC-ROC < 0.65 (degradation threshold): PagerDuty P2. Alert if AUC-ROC < 0.60: PagerDuty P1.

47. **Build model performance dashboard page.** Create `frontend/src/pages/ModelPerformance.tsx`. Components:
    - `AUCROCTrend`: Line chart of daily AUC-ROC over last 30 days, with model version promotion markers as vertical lines. Data from `GET /api/v1/learning/accuracy/history`.
    - `PrecisionRecallByTier`: Bar chart showing for each risk tier: what % of flagged deliveries actually failed (precision) and what % of all failures were caught (recall). Data from `GET /api/v1/learning/accuracy`.
    - `FeatureDriftHeatmap`: Grid of features vs. PSI value. Cells coloured green (PSI<0.1), yellow (0.1–0.2), red (>0.2). Data from `GET /api/v1/learning/drift`.
    - `ModelVersionTable`: List of all model versions with training date, data size, AUC-ROC, status, promotion date. `Promote` and `Retire` action buttons for admin users. Data from `GET /api/v1/learning/models`.
    - `TrainingDataStats`: Data volume trend chart (new labelled examples per week), class balance chart (success vs. failure ratio). Data from `GET /api/v1/learning/training-data/stats`.

48. **Build address and driver performance update jobs.** In `services/learning-loop-service/jobs/update_performance.py`, implement:
    - `update_address_ftd_rates()` (Celery Beat, 01:00 AEST): Run SQL to update `addresses.ftd_rate_30d` and `addresses.total_deliveries`, `total_failures` from rolling 30-day `delivery_attempts` data. Update Redis cache for feature store.
    - `update_driver_performance()` (Celery Beat, 01:30 AEST): Update `drivers.ftd_rate_overall`, `ftd_rate_apartment`, `ftd_rate_residential`, `ftd_rate_commercial`, `avg_stop_time_minutes` from `delivery_attempts` data. Join with `addresses.address_type` for the type-specific rates.
    - Both jobs should run in under 60 seconds on the production dataset (index-optimised queries on partitioned tables).

**Week 15: Advanced Dashboard and Client Reporting**

49. **Build SLA compliance tracking.** Create `services/api-gateway/routes/sla.py`. Implement:
    - `GET /api/v1/dashboard/sla/{client_id}`: Fetch all SLA-governed deliveries for the client (those with `deliveries.planned_window_end` set and `clients.sla_threshold` configured). Compute: breach count (actual delivery after `planned_window_end`), breach rate, SLA threshold compliance (is breach_rate <= `clients.sla_threshold`?), estimated penalty total (`ftd_events.sla_penalty_amount` sum).
    - Nightly report generation: `services/learning-loop-service/jobs/generate_client_report.py`. Celery Beat task at 20:00 AEST. For each client: generate PDF report using `weasyprint` library (HTML template in `templates/client_report.html`). Save to `s3://odocs-reports/{client_id}/{date}/daily_report.pdf`. Send via AWS SES to client's registered email address.

50. **Build SMS funnel analytics.** Add `SMSFunnelReport` to the pre-dispatch dashboard. Shows daily funnel: Deliveries scored → HIGH/CRITICAL flagged → SMS scheduled → SMS delivered → Reply received → Positive outcome (confirm/access/redirect) → Delivery success. Conversion rates at each stage. Wire to `GET /api/v1/sms/stats/{date}`. Add 30-day SMS funnel trend chart.

51. **Build driver performance intelligence page.** Create `frontend/src/pages/DriverPerformance.tsx`. Components:
    - `FleetOverview`: Table of all drivers sorted by FTD rate (worst first). Columns: name, total deliveries (30d), FTD rate (30d), on-time rate, cascade events caused, trend arrow.
    - `DriverDetailCard`: Click on driver to expand detail panel: FTD rate by address type (apartment vs. residential vs. commercial), zone heatmap (Mapbox layer showing delivery success rate by address for this driver's historical stops), SMS engagement rate for their deliveries.
    - `DriverComparisonChart`: Select 2–4 drivers to compare FTD rates over time.

**Week 16: System Hardening and Production Readiness**

52. **Implement comprehensive alerting.** Create Datadog monitor definitions in `infra/datadog/monitors/`:
    - `kafka_consumer_lag.json`: Alert if consumer lag >5,000 messages on any topic for >5 minutes.
    - `model_auc_degradation.json`: Alert if `risk_model.daily_auc_roc` < 0.65.
    - `sms_delivery_failure.json`: Alert if SMS delivery failure rate >5% in any 1-hour window.
    - `route_optimizer_timeout.json`: Alert if VRP solver timeout rate >10% of solves.
    - `cascade_agent_unhealthy.json`: Alert if `cascade.active_drivers` metric stops reporting for >10 minutes (agent died).
    - `database_replication_lag.json`: Alert if read replica lag >30 seconds.
    - All P1 alerts go to PagerDuty. P2 alerts go to Slack `#platform-alerts`. All monitors defined as Terraform resources in `infra/terraform/monitoring.tf`.

53. **Implement data retention policies.** Create `services/learning-loop-service/jobs/data_retention.py`. Celery Beat task `enforce_data_retention` weekly (Sunday 03:00 AEST):
    - Archive `training_data` records older than 365 days to S3 Parquet: `s3://odocs-training-data/archive/{year}/training_data_{year}.parquet`. Delete from PostgreSQL after successful S3 write.
    - Delete `gps_positions` TimescaleDB records older than 90 days (raw positions). Keep 1-minute downsampled aggregates (via TimescaleDB continuous aggregate) for 365 days.
    - Delete `feature_log` records older than 365 days.
    - Delete `sms_messages.body` content (nullify) for messages older than 730 days (Australian Privacy Act: don't retain SMS body longer than necessary). Keep metadata (timestamps, status).

54. **Conduct security review.** Review all API endpoints for:
    - Authentication: every endpoint (except Twilio webhooks) requires valid JWT. Twilio webhooks use Twilio signature validation middleware.
    - SQL injection: confirm all queries use parameterised statements (SQLAlchemy ORM or explicit `$1` params — no string concatenation in SQL).
    - API key storage: confirm no API keys in code or environment variables. All via AWS Secrets Manager.
    - Phone number PII: confirm `recipients.phone_e164` is not logged in plain text to application logs (use masked format `+61XXXXXXX34` in logs).
    - Rate limiting: confirm `POST /api/v1/gps/position` has per-driver rate limiting (max 5 req/second per `driver_id`) to prevent runaway driver apps flooding Kafka.

55. **Write runbooks for on-call team.** Create `docs/runbooks/` with the following runbooks:
    - `01-kafka-consumer-lag.md`: How to diagnose and resolve high consumer lag.
    - `02-risk-model-degradation.md`: Steps to investigate accuracy drop, how to roll back model version.
    - `03-sms-delivery-failures.md`: Twilio error codes, how to re-queue failed SMS sends.
    - `04-route-optimizer-timeout.md`: How to manually trigger re-optimisation, how to fall back to static route.
    - `05-database-failover.md`: RDS failover procedure, application connection string update steps.

### Key Technical Decisions

- **PSI for drift detection over KL divergence**: Population Stability Index is interpretable by engineers without a statistics background (PSI > 0.2 = "significant drift" is an accepted industry rule of thumb). KL divergence provides the same information but requires more context to interpret. Use PSI as the primary metric; log KL divergence as a secondary metric.
- **Auto-promotion threshold of +0.005 AUC-ROC**: This is intentionally conservative. A model with slightly higher AUC-ROC but worse calibration could increase the false positive rate for SMS sends (contacting recipients unnecessarily). Only auto-promote when improvement is statistically meaningful. Require manual review for marginal improvements.
- **Weasyprint for PDF reports**: Lighter weight than headless Chrome for server-side PDF generation. HTML/CSS templates are maintainable by non-engineers. For complex multi-page reports, consider switching to `reportlab` in a future iteration.

### Definition of Done

- [ ] Nightly retraining job completes within 45 minutes end-to-end (data pull, Optuna search, evaluation, artifact save).
- [ ] Auto-promotion logic tested: mock a candidate model with AUC-ROC +0.01 above active; verify it promotes automatically without manual intervention.
- [ ] Feature drift detection: mock a drifting feature (set all values to 0 for one day); verify Datadog alert fires within 1 hour.
- [ ] Daily accuracy log populating; `model_accuracy_log` has entries for all days since Phase 2 go-live (backfill if needed).
- [ ] Client SLA report PDF generated successfully for all active clients; emailed by 8pm.
- [ ] All Datadog monitors active; test each by triggering the alert condition in staging.
- [ ] Data retention policies run successfully without data loss (verify S3 archive before PostgreSQL delete).
- [ ] Security review completed; all critical findings resolved.
- [ ] Runbooks written and reviewed by on-call engineer.
- [ ] FTD rate at or below 8% sustained over 2 consecutive weeks.

### Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Optuna hyperparameter search takes >45 minutes (blocking nightly window) | Low | Medium | Set wall-clock limit on Optuna study: `study.optimize(..., timeout=2400)` (40 minutes). If not converged, use best trial found so far. Alert if best trial AUC-ROC is below expectation. Consider parallelising Optuna trials across 2 ECS tasks if single-threaded search is too slow. |
| Training data class imbalance worsens as FTD rate improves | Medium | Medium | As the platform drives FTD rate from 13% down to 7%, the training set becomes more imbalanced (7% positives). Monitor `scale_pos_weight` parameter — recalculate from training data class balance at each retraining run. Use SMOTE oversampling if class balance drops below 5% positive rate. |
| A/B test causes inconsistent driver experience | Low | Low | A/B testing is on the risk scoring model, not the route or SMS logic. Two drivers on the same day may have their deliveries scored by different models. This is acceptable — the variation is in risk score magnitude, not operational behaviour. Document this for the operations team. |
| Model training data volume too small after initial ramp | Medium | Medium | At 3,000 parcels/day with 13% failure rate = 390 positive examples/day. After 90 days = 35,100 positive examples. This is sufficient for GBDT. If operator fleet is smaller or failure rate has already improved, use a 180-day window instead of 90-day. Configure `TRAINING_WINDOW_DAYS` env var. |
