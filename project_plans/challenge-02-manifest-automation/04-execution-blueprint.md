# Challenge 2: Inbound Data Normalisation & Manifest Automation — Execution Blueprint

---

## Phase 1: Foundation (Weeks 1–3)

### Goals

Stand up the end-to-end pipeline skeleton for the two most common input formats (CSV and Excel) with manual column mapping, GNAF address validation, and the minimal ops manager review UI. By the end of Week 3, the ops manager can upload a CSV or Excel manifest, map columns through a UI, validate addresses against GNAF, and push clean records to the routing system. This replaces the morning copy-paste ritual with a structured, validated workflow even before any LLM intelligence is added.

The key principle for Phase 1: build the skeleton first, make it dumb but complete, then make it smart in Phase 2. Ship to production at end of Week 3 for internal testing with one pilot client.

---

### Atomic Tasks

#### Week 1: Backend Infrastructure + File Ingestion Gateway

**1.1 Project Scaffolding**
- Create repo structure:
  ```
  /backend
    /app
      /api/v1/routes/          # FastAPI routers
      /models/                 # SQLAlchemy models
      /schemas/                # Pydantic v2 schemas
      /services/               # Business logic
      /parsers/                # Format Parser Library
      /tasks/                  # Celery tasks
      /address/                # Address Intelligence Service
      /anomaly/                # Anomaly Detection Pipeline
      /llm/                    # LLM Schema Inference Engine
      /merge/                  # Late Manifest Merge Engine
    /migrations/               # Alembic migration scripts
    /prompts/                  # LLM prompt templates (text files)
    /tests/
    /models/                   # Serialised ML models (gitignored)
  /frontend
    /src/features/             # Feature-based directory structure
  /infrastructure
    /lambdas/
    /terraform/
  ```
- `pyproject.toml` with dependencies: `fastapi>=0.110`, `uvicorn[standard]`, `sqlalchemy>=2.0`, `alembic`, `pydantic>=2.0`, `celery>=5.3`, `redis>=5.0`, `httpx`, `boto3`, `python-multipart`, `chardet>=5.0`, `openpyxl>=3.1`, `xlrd>=2.0`, `python-magic`
- `.env.example` documenting all required env vars: `DATABASE_URL`, `REDIS_URL`, `AWS_S3_BUCKET_RAW`, `AWS_REGION`, `ANTHROPIC_API_KEY`, `GOOGLE_MAPS_API_KEY`, `HERE_API_KEY`, `GNAF_API_URL`, `ROUTING_WEBHOOK_URL`, `ROUTING_WEBHOOK_SECRET`, `SLACK_ANOMALY_WEBHOOK_URL`, `FCM_SERVER_KEY`, `CLAMAV_API_URL`
- `docker-compose.yml` for local dev: FastAPI (`uvicorn`), Celery worker, Redis, PostgreSQL 16
- GitHub Actions workflow: `ci.yml` — `ruff` lint, `mypy` type check, `pytest`, Docker build on PR

**1.2 Database Migrations (Alembic)**
- `alembic init migrations/` in backend directory
- Migration `001_initial_schema.py`: create tables `clients`, `client_configs`, `users`, `manifest_uploads`
- Migration `002_manifest_records.py`: create `manifest_raw_records`, `manifest_staging_records`, `field_mappings`
- Migration `003_validation_tables.py`: create `address_validation_results`, `anomaly_flags`, `delivery_records`, `delivery_zones`
- Migration `004_schema_profiles.py`: create `client_schema_profiles`
- Migration `005_audit.py`: create `audit_events` table
- Migration `006_late_merge.py`: create `late_merge_events` table
- Apply indexes as specified in `02-system-architecture.md` Data Models section
- Seed script `scripts/seed_dev.py`: insert 2 test clients, 1 test user, 10 delivery zones for Brisbane postcodes 4000–4999

**1.3 File Ingestion Gateway — REST Upload Endpoint**
- `app/api/v1/routes/manifests.py`: implement `POST /api/v1/manifests/upload`
- Implement auth middleware: JWT validation (use `python-jose`) for portal uploads; API key validation from `X-API-Key` header (stored in `client_configs.api_key` — bcrypt hashed)
- File pre-flight validation: size check (<50MB, configurable via `MAX_UPLOAD_SIZE_MB` env var); MIME type check using `python-magic` (reject non-CSV/Excel/JSON/EDI/PDF types)
- File hash: compute `hashlib.sha256(file_bytes).hexdigest()`; query `manifest_uploads` for duplicate within 24 hours; return 409 if found
- S3 upload: `boto3` async upload via `aioboto3` to `{AWS_S3_BUCKET_RAW}/{client_id}/{date}/{manifest_upload_id}/{filename}`; wait for S3 confirmation before writing DB record
- Write `ManifestUpload` record with status `RECEIVED`
- Enqueue Celery task: `from app.tasks.processing import process_manifest; process_manifest.delay(str(manifest_upload_id))`
- Return 202 with `{ manifest_upload_id, status: "RECEIVED", progress_url: "/api/v1/manifests/{id}/status" }`

**1.4 File Ingestion Gateway — Status + WebSocket**
- `GET /api/v1/manifests/{manifest_upload_id}/status`: query `manifest_uploads`; return current status, stage, counts
- `GET /api/v1/manifests?client_id=&date=&status=&limit=50&offset=0`: paginated manifest list
- WebSocket endpoint `WS /ws/manifests/{manifest_upload_id}` using FastAPI's `WebSocket`; store connected clients in a `dict[UUID, list[WebSocket]]` in-memory (Redis pub/sub for multi-instance later)
- `app/services/progress.py`: `publish_progress(manifest_upload_id, stage, pct, message)` — updates DB + sends WebSocket message

**1.5 Celery Task Chain Skeleton**
- `app/tasks/processing.py`: define `process_manifest(manifest_upload_id: str)` Celery task
- Implement as a sequential chain using Celery `chain()`: `detect_format.s() | parse_file.s() | apply_mapping.s() | validate_addresses.s() | detect_anomalies.s() | finalise_records.s()`
- Each task: load `ManifestUpload` from DB; call service function; update status; publish progress; return result for next task
- Error handling: wrap each task in try/except; on exception update `manifest_uploads.status = FAILED`; publish error progress event; log with `structlog`
- Celery config: `CELERY_TASK_SERIALIZER = 'json'`, `CELERY_RESULT_BACKEND = REDIS_URL`, `CELERY_BROKER_URL = REDIS_URL`, `CELERY_TASK_MAX_RETRIES = 3`, `CELERY_TASK_RETRY_BACKOFF = True`

#### Week 2: Format Parsers + Manual Schema Mapping

**1.6 Format Detection**
- `app/services/format_detector.py`: function `detect_format(file_bytes: bytes, filename: str) -> FileFormat`
- Check extension first: `.csv`, `.xlsx`, `.xls`, `.json`, `.jsonl`, `.pdf`, EDI files (no standard extension — detect by content)
- Use `python-magic` for MIME sniff: confirm extension matches MIME type
- EDI detection: check if `file_bytes[:3] == b'ISA'`
- Return `FileFormat` enum value

**1.7 CSV Parser**
- `app/parsers/csv_parser.py`: implement `CsvParser(BaseParser)`
- `chardet.detect(file_bytes[:10240])` for encoding; decode bytes with detected encoding (fallback UTF-8)
- Strip BOM: `content.lstrip('\ufeff')`
- Delimiter detection: try each of `[',', '\t', '|', ';']` with `csv.Sniffer` + fallback to measuring average column count consistency
- Parse with `csv.DictReader`; filter empty rows (all values empty after strip)
- Multi-header detection: if row index 1 also has all non-numeric values and >50% overlap with row 0 tokens, log warning and use row 0 only
- Return `ParseResult`
- Unit tests: `tests/parsers/test_csv_parser.py` — fixtures in `tests/fixtures/csv/`: `shopify_export.csv`, `woocommerce_export.csv`, `tab_delimited.tsv`, `pipe_delimited.csv`, `utf16_encoded.csv`, `bom_prefixed.csv`, `empty_rows.csv`, `single_column.csv`

**1.8 Excel Parser**
- `app/parsers/excel_parser.py`: implement `ExcelParser(BaseParser)`
- Detect format from MIME type: `.xlsx` → `openpyxl`; `.xls` → `xlrd`
- `openpyxl`: `load_workbook(BytesIO(file_bytes), data_only=True)`; select first sheet or configured sheet name
- Merged cell handling: iterate `ws.merged_cells.ranges`; unmerge and fill forward
- Subtotal row detection: check first cell of each row for subtotal keywords; skip if matched
- Date cell handling: `isinstance(cell.value, datetime)` → `.strftime('%Y-%m-%d')`
- Numeric formatting: `f"{value:.10g}"` to avoid scientific notation on large numbers
- Return `ParseResult`
- Unit tests: fixtures in `tests/fixtures/excel/`: `merged_headers.xlsx`, `subtotals_mixed.xlsx`, `date_formatted.xlsx`, `multisheet.xlsx`

**1.9 Manual Schema Mapping — Backend**
- `app/services/schema_mapper.py`: function `create_manual_mapping(manifest_upload_id, headers) -> list[FieldMapping]`
- Creates placeholder `FieldMapping` records with `confidence=0.0`, `human_confirmed=False` for each header
- `POST /api/v1/manifests/{id}/schema-mapping/approve` endpoint: validate request body; write `FieldMapping` records with `human_confirmed=True`; transition status to `SCHEMA_CONFIRMED`; resume Celery chain
- `GET /api/v1/manifests/{id}/schema-mapping` endpoint: return field mappings with sample data preview (first 5 unique non-null values per column, drawn from `manifest_raw_records`)
- Define canonical field enum in `app/schemas/canonical_fields.py`: list of all 23 canonical field names with human-readable labels and descriptions (used to populate the mapping dropdown in UI)

**1.10 Record Transformation Service**
- `app/services/transformer.py`: function `transform_records(manifest_upload_id, field_mappings) -> int`
- Load `ManifestRawRecord` rows for manifest; apply `FieldMapping` (rename columns, discard unmapped columns)
- Normalise phone numbers: `phonenumbers.parse(value, 'AU')`; format as E.164; catch `NumberParseException` → store raw value + flag transformation warning
- Normalise dates: try `dateparser.parse(value)` (install `dateparser` library); fallback to common formats `['%d/%m/%Y', '%Y-%m-%d', '%m/%d/%Y', '%d-%m-%Y']`; store as `YYYY-MM-DD`
- Normalise state codes: map `['Queensland', 'Qld', 'QLD', 'qld'] → 'QLD'` etc. via lookup dict in `app/data/state_codes.py`
- Normalise postcodes: zero-pad to 4 digits (`str(int(value)).zfill(4)`)
- Bulk insert transformed records into `manifest_staging_records`; return inserted count

#### Week 3: GNAF Address Validation + Review UI Foundation

**1.11 GNAF Integration**
- Provision self-hosted GNAF on RDS PostgreSQL (separate from application DB): download GNAF data from `data.gov.au/dataset/geocoded-national-address-file-g-naf`; load into RDS using provided SQL scripts; install PostGIS extension
- `app/address/gnaf_client.py`: async GNAF client using `httpx` (or direct PostgreSQL queries via `asyncpg` to the GNAF RDS instance — preferred for latency)
- `exact_match(address_line_1, suburb, state, postcode)`: full-text search query against GNAF `address_detail` view; return first result or None
- `fuzzy_search(address_string, limit=5)`: PostgreSQL trigram similarity (`pg_trgm` extension) query; return candidates with similarity scores
- `app/address/validator.py`: `validate_batch(records)` — asyncio with semaphore 50; call GNAF; fall back to Google Maps; write `AddressValidationResult` records
- Integration test: `tests/integration/test_gnaf_client.py` — test 10 known Brisbane addresses; verify GNAF PID returned; test known bad address; verify fuzzy suggestion returned

**1.12 Google Maps Fallback**
- `app/address/google_maps_client.py`: async `httpx` client; `geocode(address, country='AU')` function
- Parse response: extract lat/lng, formatted_address, address_components
- Rate limiter: `asyncio.Semaphore(50)` (50 QPS)
- Redis counter for daily call tracking: `INCR google_maps:calls:{date}`; if >35,000 (below 40,000 free tier limit) set `SETEX google_maps:quota_exceeded 3600 1` to activate HERE fallback
- `app/address/here_client.py`: same pattern; activated when `google_maps:quota_exceeded` key exists in Redis

**1.13 Basic Rule-Based Anomaly Detection**
- `app/anomaly/rules/`: implement 6 critical rules for Phase 1:
  - `MissingAddressRule` (CRITICAL)
  - `MissingPostcodeRule` (CRITICAL)
  - `ExactDuplicateParcelIdRule` (CRITICAL)
  - `UndeliverableAddressRule` (CRITICAL — uses `is_deliverable` from address validation result)
  - `MissingPhoneRule` (WARNING)
  - `NearDuplicateAddressRule` (WARNING)
- `app/anomaly/pipeline.py`: `run_rule_engine(manifest_upload_id)` — run all rules; write `AnomalyFlag` records; update manifest anomaly counts
- Remaining rules and Isolation Forest deferred to Phase 3

**1.14 Clean Record Finalisation + Routing Webhook**
- `app/tasks/finalise.py`: `finalise_records(manifest_upload_id)` Celery task
- Select staging records with `status = APPROVED` (or all non-rejected if no anomaly review configured); bulk insert into `delivery_records`
- Update `manifest_uploads.status = PROCESSED`, `processing_completed_at = now()`
- Fire routing webhook: `httpx.post(ROUTING_WEBHOOK_URL, json={...}, headers={'X-Webhook-Signature': hmac_sign(...)})`; retry 5x with exponential backoff using Celery `self.retry()`

**1.15 Frontend Foundation**
- Scaffold React + TypeScript + Vite project in `/frontend`
- Install dependencies: `@tanstack/react-query`, `zustand`, `react-dropzone`, `@tanstack/react-table`, `tailwindcss`, `shadcn/ui` (init)
- `src/lib/api.ts`: typed API client using `fetch` (not axios — keep dependencies minimal); auto-attaches JWT from localStorage; handles 401 with redirect to login
- `src/features/upload/UploadPage.tsx`: react-dropzone UI; calls `POST /api/v1/manifests/upload`; on success, redirects to status page for that `manifest_upload_id`
- `src/features/manifest-progress/ManifestProgressPage.tsx`: polls `GET /api/v1/manifests/{id}/status` every 3 seconds; opens WebSocket connection; renders stage timeline component
- `src/features/schema-review/SchemaReviewPage.tsx`: renders mapping table; `<ConfidenceBadge>` component; canonical field dropdown; sample data preview panel; Approve button
- `src/features/anomaly-review/AnomalyReviewPage.tsx`: anomaly list; per-severity filter tabs; resolve action buttons
- `src/features/dashboard/DashboardPage.tsx`: manifest history table with TanStack Table; status badges; quick action buttons

---

### Key Technical Decisions — Phase 1

**Decision: Self-hosted GNAF vs PSMA API**
- Decision: Self-hosted GNAF on RDS PostgreSQL
- Rationale: At 3,000 records/day, PSMA API per-call costs would be significant. Self-hosted RDS instance (~$80/month) is cheaper within weeks, and eliminates the latency of external API calls. Query latency: <10ms vs ~200ms per API call. Data freshness: quarterly update acceptable for address validation.
- Implementation: RDS PostgreSQL 16 `db.t3.medium` instance; `pg_trgm` extension for fuzzy search; PostGIS for future geo queries

**Decision: Celery chain vs Celery canvas (chord)**
- Decision: Sequential chain (`chain()`) in Phase 1; upgrade to chord for parallel validation in Phase 2
- Rationale: Simpler error handling and debugging in Phase 1. Parallel address validation (chord) will be added in Phase 2 when the address validation bottleneck is confirmed.

**Decision: JWT vs API key auth**
- Decision: Both — JWT for portal UI sessions (short-lived, 1 hour expiry); API key (hashed with bcrypt in `client_configs.api_key_hash`) for SFTP and programmatic clients
- Rationale: Portal users need session management; programmatic clients need stable credentials. Both validated in a single FastAPI dependency `app/dependencies/auth.py`

---

### Definition of Done — Phase 1

- [ ] Ops manager can upload a CSV or Excel file via the portal UI
- [ ] File is archived to S3 before any processing response is sent
- [ ] Duplicate file submission returns 409 with reference to original upload
- [ ] Pipeline completes CSV/Excel parsing in <5 seconds for a 3,000-row file
- [ ] Manual schema mapping UI displays all columns with sample values; ops manager can map each column to a canonical field; approval writes field mappings to DB
- [ ] All address fields validated against GNAF; unresolvable addresses fall back to Google Maps; `AddressValidationResult` written for every staging record
- [ ] Rule-based anomaly detection runs for all 6 Phase 1 rules; `AnomalyFlag` records written; manifest anomaly counts updated
- [ ] Clean records written to `delivery_records` after anomaly review
- [ ] Routing webhook fires on `PROCESSED`; HMAC signature verified by routing system
- [ ] End-to-end pipeline tested with 3 real client manifest files (CSV + Excel formats)
- [ ] All API endpoints have OpenAPI docs (automatic via FastAPI)
- [ ] `pytest` test suite: >80% line coverage on parsers and transformation service
- [ ] Deployed to staging environment (ECS Fargate); accessible by ops manager for UAT

---

### Risks & Mitigations — Phase 1

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| GNAF RDS setup takes longer than estimated | Medium | High (blocks address validation) | Spike task in Week 1 Day 1 to provision and load GNAF. If blocked, use Google Maps as primary for Phase 1 and migrate to GNAF in Phase 2. |
| Client manifest fixtures not representative of actual files | Medium | Medium (parsers may fail on real files) | Collect actual sample manifests from all 4 current clients in Week 1; use as test fixtures throughout Phase 1. Ops manager to provide sanitised samples. |
| ClamAV sidecar adds latency to upload endpoint | Low | Low | Run ClamAV scan async after S3 archive; do not block HTTP response. If virus found, set manifest status to QUARANTINED and alert. |
| HMAC webhook signature mismatch with routing system | Low | High (no records reach routing) | Integration test the webhook against a mock routing endpoint before wiring to real system. Document the signing algorithm in `infrastructure/docs/webhook-integration.md`. |

---

## Phase 2: LLM Intelligence (Weeks 4–6)

### Goals

Replace the manual column mapping workflow with Claude-powered schema inference. After Phase 2, the ops manager's interaction with schema mapping is reduced to reviewing LLM-suggested mappings (2–3 minutes for a new format) and confirming with a click. Known client formats auto-process. Introduce the per-client schema learning system so that after 3 confirmed uploads, clients are fully automated.

---

### Atomic Tasks

#### Week 4: Claude API Integration + Schema Inference

**2.1 Prompt Engineering**
- Create `prompts/schema_inference_system.txt`
- System prompt content:
  - Role definition: "You are an expert at mapping delivery manifest file formats to a canonical delivery schema."
  - Canonical field definitions: for each of the 23 canonical fields, provide: field name, description, data type, 5–8 example column names it might appear as (e.g., `recipient_name` → `["Name", "Customer Name", "Recipient", "Deliver To", "ship_to_name", "contact_name", "addressee"]`)
  - Output format instruction: "Return ONLY a JSON object conforming to the following schema: ..."
  - JSON schema definition for `FieldMappingResult` (see below)
  - 5 few-shot examples: each example is a markdown table of headers + 3 sample rows, followed by the correct JSON mapping output
  - Edge case instructions: columns that are clearly irrelevant (SKU, product ID, inventory code) → `canonical_field: null`; multiple columns mapping to same field → choose most complete-looking, flag others as REDUNDANT; phone numbers that appear to be landlines → map to `phone_landline` not `phone_mobile`
- Test prompt against all 4 client manifest samples; iterate until accuracy ≥ 95% on known formats

**2.2 Pydantic Schema for LLM Response**
- `app/schemas/llm_responses.py`:
  ```python
  class FieldMappingItem(BaseModel):
      source_column: str
      canonical_field: str | None  # None = ignore this column
      confidence: float = Field(ge=0.0, le=1.0)
      reasoning: str
      action: Literal['MAP', 'IGNORE', 'REDUNDANT', 'AMBIGUOUS']
      ambiguous_alternatives: list[str] | None = None  # for AMBIGUOUS fields

  class FieldMappingResult(BaseModel):
      mappings: list[FieldMappingItem]
      overall_confidence: float
      unmapped_required_fields: list[str]  # canonical fields not found in source
      notes: str | None  # free-text notes from LLM about the file
  ```

**2.3 LLM Schema Inference Service**
- `app/llm/schema_inference.py`:
  - `infer_schema(headers, sample_rows, client_id, existing_profile=None) -> SchemaInferenceResult`
  - Build user message: format headers as JSON array; format 10 sample rows as a markdown table with header row; if `existing_profile` provided, include a note: "This client has previously uploaded files with this header set. Previous mapping: [json]. Verify whether the same mapping applies."
  - Anthropic SDK call: `client.messages.create(model='claude-sonnet-4-6', max_tokens=2000, system=system_prompt, messages=[{'role': 'user', 'content': user_message}])`
  - Parse response text as JSON; validate against `FieldMappingResult` Pydantic model
  - On JSON parse error: retry once with added instruction "Your previous response was not valid JSON. Return only the JSON object, no other text."
  - On second failure: raise `LLMInferenceError`; set manifest status to `LLM_INFERENCE_FAILED`; alert via Slack; do NOT block the pipeline — fall back to manual mapping
  - Return `SchemaInferenceResult` with per-field confidence and overall confidence

**2.4 Schema Cache Check**
- `app/llm/schema_cache.py`:
  - `check_cache(client_id, header_fingerprint) -> ClientSchemaProfile | None`
  - If profile found AND `auto_process_enabled = True` AND current headers match stored `header_fingerprint` → return profile (skip LLM call)
  - If profile found BUT `auto_process_enabled = False` → pass profile to LLM as context (use as few-shot hint, not as direct answer)
  - `header_fingerprint(headers: list[str]) -> str`: `hashlib.sha256('|'.join(sorted(h.lower().strip() for h in headers)).encode()).hexdigest()`

**2.5 Modify Celery Chain for LLM Stage**
- Update `app/tasks/processing.py`: insert `infer_schema` task between `parse_file` and `apply_mapping`
- `infer_schema` task logic:
  1. Load `ManifestUpload` + `ParseResult` from DB/Redis
  2. Check schema cache; if cache hit → write `FieldMapping` records from profile → status `SCHEMA_AUTO_APPLIED` → continue chain
  3. If cache miss → call `infer_schema()` → write `FieldMapping` records → evaluate confidence
  4. If all confidence ≥ threshold → status `SCHEMA_CONFIRMED` → continue chain
  5. If any confidence < threshold → status `AWAITING_SCHEMA_REVIEW` → publish WebSocket event → **pause chain**

**2.6 Chain Resume Mechanism**
- When `approve_mapping` API endpoint is called:
  1. Validate user auth and role (`ops_manager` or `admin`)
  2. Update `FieldMapping` records with human confirmations/overrides
  3. Update `ClientSchemaProfile` with corrections
  4. Set `manifest_uploads.status = SCHEMA_CONFIRMED`
  5. Enqueue the remaining chain steps as a new Celery chain starting from `transform_records`
  6. Return 200 with updated status
- This is simpler than trying to "resume" a paused Celery chord — the chain is simply re-created from the checkpoint

#### Week 5: Confidence Scoring + Review UI Enhancement

**2.7 Confidence Scoring Logic**
- Per-field confidence from LLM response is the primary score
- Apply multipliers for data validation: after transformation, check if the mapped field's values conform to the expected type (e.g., `postcode` should be 4 digits; `phone` should parse as Australian number). If >20% of values fail type check → multiply confidence by 0.8 and add a note
- Computed `adjusted_confidence` field on `FieldMapping` records (stored alongside raw LLM confidence)
- Overall manifest confidence = `min(adjusted_confidence values for non-IGNORE fields)`

**2.8 Enhanced Schema Review UI**
- Update `src/features/schema-review/SchemaReviewPage.tsx`:
  - Show LLM reasoning in expandable tooltip per field
  - Show `adjusted_confidence` (post-type-check) not raw LLM confidence for cleaner signal
  - Add "Why low confidence?" button for fields with confidence <0.8: shows LLM reasoning + type check failure count
  - Add sample value preview inline in the mapping table (3 values shown without clicking)
  - For `AMBIGUOUS` fields, show the alternative options as a quick-select alongside the dropdown
  - For `REDUNDANT` fields, collapse them under an "Unmapped columns" section (visible but not requiring action)
  - Track review time: `reviewed_at - schema_inference_completed_at`; send to Datadog as `manifest.schema_review_duration_seconds`

**2.9 Client Schema Learning**
- `app/llm/schema_learner.py`: called at the end of each `SCHEMA_CONFIRMED` transition
  - `record_successful_upload(client_id, header_fingerprint, field_mappings, was_llm_result)`
  - Increment `successful_upload_count` on `ClientSchemaProfile`
  - If `successful_upload_count >= 3` AND `min(field_confidences) >= threshold` → set `auto_process_enabled = True`
  - Log learning event to `audit_events` table
  - When `auto_process_enabled` flips to True: send notification to ops manager: "Client X now auto-processing. You'll be notified of anomalies only."
- Handle format drift: if a new upload's `header_fingerprint` does not match stored profile → create a new profile record (do not update the existing one); mark old profile as `status = SUPERSEDED`; require LLM inference for the new format

#### Week 6: Integration Testing + Client Onboarding Flow

**2.10 LLM Integration Tests**
- `tests/integration/test_llm_schema_inference.py`:
  - Test all 4 client manifest formats (using sanitised fixtures)
  - Assert: all required fields mapped; overall confidence ≥ 0.90 for all known formats; no CRITICAL unmapped required fields
  - Test malformed LLM response handling: mock `anthropic.messages.create` to return invalid JSON; assert retry logic fires; assert fallback to manual mapping on second failure
  - Test prompt cost: measure token usage per call; assert `prompt_tokens < 3000` (controls cost)

**2.11 Schema Profile Admin UI**
- `src/features/schema-profiles/SchemaProfilesPage.tsx`:
  - List all profiles per client: profile name, header fingerprint (first 8 chars), auto-process status, upload count, last used
  - "Disable auto-processing" toggle per profile (calls `PATCH /api/v1/schema-profiles/{id}`)
  - "Delete profile" button (forces LLM re-inference on next upload)
  - Link to view the confirmed mappings for a profile in read-only mode

**2.12 New Client Onboarding Flow**
- `src/features/onboarding/NewClientOnboardingPage.tsx`: wizard UI
  - Step 1: Client details (name, contact, config options)
  - Step 2: Upload a sample manifest → triggers LLM inference → shows schema review in-page (same `SchemaReviewPage` component, embedded)
  - Step 3: Confirm the mapping → saves as the initial `ClientSchemaProfile` with `successful_upload_count = 1`
  - Step 4: Configure client options (anomaly review settings, webhook URL, export format)
  - Target: new client setup time <15 minutes end-to-end

---

### Key Technical Decisions — Phase 2

**Decision: Retry count for LLM API calls**
- Decision: Retry 3x with 5-second linear backoff; on all retries exhausted, fall back to manual mapping
- Rationale: The Claude API is highly reliable, but occasional timeouts occur. The fallback to manual mapping is essential — LLM inference must never be a hard dependency that blocks the ops morning workflow. The manual mapping path must always work.

**Decision: Temperature = 0 for schema inference**
- Decision: `temperature=0` (deterministic)
- Rationale: Schema inference is a structured extraction task, not a creative generation task. Deterministic output means the same file always maps the same way, which is essential for building trust with the ops manager. Non-zero temperature could produce different mappings on re-runs, which would be confusing.

**Decision: Store LLM reasoning in DB**
- Decision: Yes — store `llm_reasoning` per field in `field_mappings` table
- Rationale: The reasoning is the key trust-building element for the ops manager. When they hover over a confidence indicator and read "I mapped this column to `phone_mobile` because the values match Australian mobile number format (04XX XXX XXX) and the column label contains 'mobile'," they understand why and can quickly confirm or override. Without stored reasoning, the review UI cannot show this.

---

### Definition of Done — Phase 2

- [ ] LLM schema inference runs automatically after file parsing; `FieldMapping` records written with confidence scores and reasoning
- [ ] Files from all 4 current clients infer schema with overall confidence ≥ 0.90
- [ ] Low-confidence fields trigger human review notification (WebSocket + Slack)
- [ ] Review UI shows LLM reasoning tooltips, adjusted confidence, sample values
- [ ] Schema approval via UI resumes the Celery chain within 10 seconds
- [ ] After 3 confirmed uploads from a client, `auto_process_enabled` flips to True; subsequent uploads from that client process without any human review
- [ ] LLM fallback to manual mapping tested and verified to work on API timeout/error
- [ ] LLM API cost per manifest is tracked in Datadog; average <$0.01 per manifest
- [ ] New client onboarding flow tested with one new client; setup time <15 minutes

---

### Risks & Mitigations — Phase 2

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| LLM inference accuracy drops below 90% for one of the 4 client formats | Medium | High | Run prompt engineering iterations in Week 4 before wiring to pipeline. Add client-specific few-shot examples to system prompt for any format that underperforms. |
| Claude API rate limits triggered on high-volume upload days | Low | Medium | The number of LLM calls per day is low (1 per manifest, 5–8 manifests/day). Rate limits not a realistic concern at this scale. Monitor `anthropic_api:call_count:{date}` in Redis anyway. |
| Ops manager distrust of LLM mappings leads to reviewing everything manually | Medium | Medium | Track and report the accuracy of auto-approved mappings vs. manually corrected ones in Datadog. Share weekly accuracy metric with ops manager to build trust. Set auto-process threshold high (0.95) so only genuinely confident mappings auto-approve. |
| LLM reasoning text is confusing or too technical for ops manager | Low | Low | Review all reasoning outputs with the ops manager during Week 6 UAT. Adjust prompt to produce simpler, action-oriented reasoning. E.g., "I think this is the mobile phone number column because..." not "Column `c7` exhibits token patterns consistent with E.164 format..." |

---

## Phase 3: Advanced Validation (Weeks 7–9)

### Goals

Complete the anomaly detection pipeline with ML-based detection (Isolation Forest), add the remaining rule-based checks, integrate FTD history cross-reference, add EDI 204 support for the enterprise client, add PDF extraction, and complete the full anomaly review UI. By end of Phase 3, the pipeline handles all 5 input formats and provides comprehensive pre-routing data quality validation.

---

### Atomic Tasks

#### Week 7: Isolation Forest + Complete Rule Engine

**3.1 Feature Engineering for Isolation Forest**
- `app/anomaly/ml/features.py`: `build_feature_matrix(records: list[StagingRecord], client_id) -> pd.DataFrame`
- Features:
  - `weight_kg_normalised`: `weight_kg / MAX_WEIGHT` (MAX_WEIGHT = 30kg, from ClientConfig)
  - `delivery_zone_encoded`: label-encode from `delivery_zones` table
  - `service_type_encoded`: label-encode
  - `upload_hour`: `int(manifest.created_at.hour)`
  - `postcode_prefix`: `int(str(postcode)[:2])`
  - `has_special_instructions`: `1 if special_instructions else 0`
  - `requires_signature`: `int(requires_signature)`
  - `item_count_normalised`: `item_count / 10` (capped)
- Handle missing values: impute with median for numeric features; mode for categorical

**3.2 Isolation Forest Training Pipeline**
- `app/anomaly/ml/trainer.py`:
  - `train_client_model(client_id: UUID)`: query `delivery_records` for last 6 months for this client; build feature matrix; fit `IsolationForest(n_estimators=100, contamination=0.05, random_state=42)`; serialise with `joblib.dump()` to `/tmp/models/isolation_forest_{client_id}.joblib`; upload to `s3://manifest-models/isolation_forest_{client_id}.joblib`
  - `train_global_model()`: same but using all clients' data; used as fallback for clients with <500 records
  - CLI command `python -m app.cli train-anomaly-models` for initial training
  - Celery beat task `retrain_anomaly_models` scheduled weekly at 02:00 Sunday

**3.3 Isolation Forest Scoring**
- `app/anomaly/ml/scorer.py`:
  - `load_model(client_id) -> IsolationForest`: check local cache first; download from S3 if not present; fallback to global model if client model not found
  - `score_records(records, client_id) -> list[tuple[UUID, float]]`: build feature matrix; `model.score_samples(X)` returns negative scores (lower = more anomalous); normalise to 0.0–1.0 range; pair with record IDs
  - Score ≥ 0.7 → create `AnomalyFlag` with `anomaly_type=STATISTICAL_OUTLIER`, `severity=INFO`
  - Score ≥ 0.9 → `severity=WARNING`

**3.4 Complete Rule Engine**
- Add remaining rules not built in Phase 1:
  - `InvalidPostcodeQldRule` (`WARNING`)
  - `InvalidPhoneFormatRule` (`INFO`)
  - `SuspiciousVolumeRule` (`WARNING`) — requires cross-record context; include in `RuleContext`
  - `ZeroWeightRule` (`INFO`)
  - `HighFtdRateRule` (`WARNING`) — requires `AddressValidationResult.ftd_rate_historical`
- Refactor `app/anomaly/pipeline.py` to run: rule engine → ML scoring → FTD check → deduplicate flags → write to DB

**3.5 FTD History Integration**
- `app/address/ftd_history.py` (as specified in Module 4): `get_ftd_rate(gnaf_pid, address_line_1, postcode) -> float`
- Query: `SELECT COUNT(*) FILTER (WHERE outcome = 'FAILED'), COUNT(*) FROM delivery_records WHERE (gnaf_pid = $1 OR (address_line_1_normalised = $2 AND postcode = $3)) AND delivery_date >= NOW() - INTERVAL '12 months'`
- Index required: `CREATE INDEX CONCURRENTLY delivery_records_ftd_lookup ON delivery_records (gnaf_pid) INCLUDE (outcome)` — migration `007_ftd_index.py`
- Return 0.0 if no history found (not penalise new addresses)

**3.6 Complete Anomaly Review UI**
- `src/features/anomaly-review/`: additions:
  - Near-duplicate comparison view: side-by-side two records with token-level diff highlighting using `diff` npm package
  - `SuspiciousVolumeCard.tsx`: shows all records at the flagged address grouped together; "Mark as expected" action (creates an exception rule for this address/client combination)
  - Bulk resolve by anomaly type: "Accept all missing-phone warnings" button
  - Anomaly resolution audit trail panel (read-only, shows previous actions on this manifest)
  - Export anomaly report as CSV: `GET /api/v1/manifests/{id}/anomalies/export`

#### Week 8: EDI 204 Support

**3.7 EDI X12 204 Parser**
- `app/parsers/edi_parser.py`: implement `EDI204Parser(BaseParser)`
- Use `pyx12` to parse ISA/GS/ST/SE/GE/IEA envelope
- Transaction set 204 loop structure:
  - `B2` segment: manifest info (carrier ID, shipment date)
  - `L11` loop: reference numbers (Bill of Lading, PO number → map to `order_id`, `client_reference`)
  - `N1` loop with qualifier `SH` (shipper) and `CN` (consignee): name, address → map to `recipient_name`, `address_line_1`
  - `N3`, `N4` segments: street address, city/state/zip → `address_line_2`, `suburb`, `state`, `postcode`
  - `S5` loop (stop off details): sequence number, delivery instructions → `special_instructions`, `stop_sequence`
  - `AT8` segment: weight → `weight_kg`
  - `OID` segment: order information, piece count → `item_count`, `parcel_id`
- Handle multiple stops per 204: one `ParseResult` record per S5 loop
- `ParseResult.metadata['edi_requires_ack'] = True` for records needing a 990 response

**3.8 EDI 990 Response Generation**
- `app/parsers/edi_response.py`: `generate_990(edi_204_meta: dict, accepted: bool, errors: list[str]) -> bytes`
- Generate X12 990 (Response to a Load Tender) accepting or rejecting the tender
- Called from `finalise_records` task when `manifest_uploads.detected_format == EDI_X12` and `edi_requires_ack`
- Transmit via SFTP back to client's EDI endpoint (configured in `ClientConfig.edi_sftp_host`, `edi_sftp_path`, `edi_sftp_credentials_secret`)

**3.9 EDI Client Configuration**
- Add `edi_sender_id`, `edi_receiver_id`, `edi_sftp_host`, `edi_sftp_path`, `edi_sftp_credentials_secret` fields to `client_configs` (migration `008_edi_config.py`)
- Admin UI: `src/features/client-config/EdiConfigSection.tsx` for configuring EDI parameters

#### Week 9: PDF Extraction + Integration Testing

**3.10 PDF Table Extractor**
- `app/parsers/pdf_parser.py`: implement `PDFParser(BaseParser)`
- Primary extraction: `pdfplumber.open(BytesIO(file_bytes))` → iterate pages → `page.extract_table()` → collect non-None tables
- Multi-page: concatenate all page tables; detect repeated header rows (compare first row of each page to first page's header using `RapidFuzz` similarity > 0.95 → skip as repeated header)
- Fallback to `camelot`: if `pdfplumber` returns empty tables → try `camelot.read_pdf(filepath, flavor='lattice')`, then `flavor='stream'`; `camelot` requires a file path, not bytes → write to `tempfile.NamedTemporaryFile(suffix='.pdf')` first
- Cell value cleaning: strip `\n` within cells; strip leading `•`, `-`, `*` (bullet points in PDF text cells)
- Log warning for each page yielding no table; if entire PDF yields no tables → `ParseError(reason='NO_TABLES_FOUND')`
- Note: PDF parsing is inherently imperfect for non-structured PDFs. Add `ParseResult.metadata['pdf_extraction_confidence']` score (ratio of pages that successfully yielded tables). If <0.8 → flag for human review regardless of LLM mapping confidence.

**3.11 Full Pipeline Integration Tests**
- `tests/integration/test_full_pipeline.py`:
  - Test 1: CSV manifest from Client A → auto-processed after schema learning → 3,000 records in `delivery_records` within 90 seconds
  - Test 2: Excel manifest with merged headers → manual schema review triggered → approval resumes pipeline → clean records written
  - Test 3: EDI 204 manifest → parsed correctly → 990 response generated and written to SFTP mock
  - Test 4: PDF manifest → pdfplumber extraction → schema inference → records written
  - Test 5: Manifest with 5 duplicates and 2 bad addresses → CRITICAL anomalies block routing → resolution clears block
  - Test 6: GNAF unavailable (mock) → Google Maps fallback triggered → HERE fallback triggered → all addresses resolved
- Performance test: `tests/performance/test_processing_time.py` — 3,000-record CSV manifest processed end-to-end in <90 seconds (measure with `time.perf_counter`)

---

### Key Technical Decisions — Phase 3

**Decision: camelot-py vs tabula-py for PDF fallback**
- Decision: `camelot-py[cv]`
- Rationale: Camelot has better handling of borderless tables (`stream` flavor) which are common in delivery manifests printed from legacy systems. Tabula requires Java (JRE) as a dependency, which adds container weight. camelot's OpenCV dependency is heavier but avoids a JRE.

**Decision: Isolation Forest contamination parameter**
- Decision: `contamination=0.05` (assume 5% anomaly rate in training data)
- Rationale: Conservative estimate. The actual anomaly rate in delivery records is low (most historical records are clean). Start at 5% and tune downward if false positive rate is too high after 4 weeks of production data. Make this a configurable `ClientConfig` field (`isolation_forest_contamination`) for per-client tuning.

**Decision: pyx12 vs x12 library for EDI parsing**
- Decision: `pyx12`
- Rationale: `pyx12` is more mature and has better support for 204 transaction sets with the stop-off loops common in multi-drop delivery manifests. The alternative `x12` library is newer but has less community validation for 204 specifically.

---

### Definition of Done — Phase 3

- [ ] Isolation Forest model trained on at least 1,000 historical records; scoring runs in <5 seconds for 3,000-record manifest
- [ ] All 11 anomaly detection rules implemented and tested
- [ ] FTD history lookup integrated into address validation result
- [ ] EDI 204 manifest from test fixture parses correctly; all stops extracted; 990 response generated
- [ ] PDF manifest extraction works for table-based PDFs; multi-page handled; fallback to camelot verified
- [ ] Full pipeline integration tests pass for all 5 formats
- [ ] Performance test: 3,000-record manifest processes in <90 seconds on staging hardware
- [ ] Anomaly review UI complete with near-duplicate comparison, bulk resolve, and audit trail

---

### Risks & Mitigations — Phase 3

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| PDF extraction fails for the actual client's PDF format | High | Medium | Test pdfplumber against actual client PDF samples in Week 9 Day 1. PDF parsing is inherently format-dependent. If pdfplumber + camelot both fail, document as a known limitation and flag PDF uploads for manual pre-processing until a purpose-built PDF profile can be developed. |
| Isolation Forest produces too many false positives in Week 7 testing | Medium | Medium | Start with `contamination=0.02` (2%) if false positive rate is high. The rule engine provides deterministic safety net; ML scoring is INFO/WARNING severity only, so false positives do not block dispatch. |
| EDI 990 response not accepted by enterprise client's AS2 system | Low | High | Validate 990 output against an EDI validator (free tool: `stedi.com` EDI validation) before connecting to real client. Test SFTP drop of 990 in staging before going live. |
| `camelot-py[cv]` OpenCV dependency causes Docker image size bloat | Low | Low | Use multi-stage Docker build; camelot + OpenCV in a separate build stage. If image size is problematic, run PDF extraction as a separate Lambda function (invoked by the main pipeline). |

---

## Phase 4: Automation & Learning (Weeks 10–12)

### Goals

Complete the Late Manifest Merge Engine, achieve full automation for all learned client formats, build the ops manager monitoring dashboard, and prepare for production launch. By end of Phase 4, the daily morning workflow is: receive Slack notification "All manifests processed. 3 warnings flagged. Ready for dispatch," click to review 3 warnings, click approve, done — in under 5 minutes.

---

### Atomic Tasks

#### Week 10: Late Manifest Merge Engine

**4.1 Late Manifest Detection**
- `app/tasks/finalise.py`: in `finalise_records` task, after records are written to `delivery_records`, check:
  - `manifest.created_at.time() > ClientConfig.late_manifest_cutoff_time`
  - AND `SELECT COUNT(*) FROM routes WHERE delivery_date = manifest.manifest_date AND status IN ('OPTIMISED', 'IN_PROGRESS') > 0`
  - If both true → set `manifest.is_late_manifest = True` → enqueue `merge_late_manifest` task instead of `dispatch_to_routing`

**4.2 Route Matcher**
- `app/merge/route_matcher.py`: `match_stops_to_routes(new_records, manifest_date) -> dict[UUID, UUID]`
- Load all routes for `manifest_date` with status `OPTIMISED` or `IN_PROGRESS`; load all future stops per route (stops where `route_stop.status != 'COMPLETED'`)
- For each new record, compute centroid of each route's future stops: `(mean_lat, mean_lng)`
- Compute Haversine distance from new record's `(validated_latitude, validated_longitude)` to each route centroid
- Check constraints: capacity check (`SELECT SUM(weight_kg) FROM delivery_records WHERE route_id = $1`) vs `vehicle.max_weight_kg`; time check (estimate +10 minutes per additional stop; verify `route.estimated_end_time + additional_time <= driver.max_shift_end_time + timedelta(minutes=30)`)
- Assign to minimum-distance valid route; if no valid route → append to `unassignable_stops` list
- Use `geopy.distance.distance` for Haversine computation (install `geopy`)

**4.3 Partial Re-optimisation**
- `app/merge/reoptimiser.py`: `reoptimise_route_segment(route_id, new_record_ids) -> list[tuple[UUID, int]]`
- Load future stops for route (not yet completed); add new records
- Build OR-Tools distance matrix: `google.ortools.constraint_solver` — for each pair of stops, compute travel time estimate using straight-line distance × 1.3 (urban detour factor, configurable)
- Routing model: single vehicle; fixed start = driver's current location (or depot if not started); minimise total distance
- Time limit for solver: `search_parameters.time_limit.seconds = 5` (hard limit — this is partial optimisation, not exhaustive solve)
- Return list of `(delivery_record_id, new_stop_sequence)` pairs

**4.4 Impact Calculator**
- `app/merge/impact_calculator.py`: `calculate_impact(route_id, new_stops, new_stop_sequences) -> RouteImpact`
- Estimate time added: `len(new_stops) × 10 minutes` (configurable average stop time) + `routing_delta_km × 2.5 minutes/km` (configurable speed)
- Compute revised completion ETA: `route.current_eta + timedelta(minutes=time_added_minutes)`
- Compute capacity utilisation: `(current_weight + new_weight) / vehicle.max_weight_kg`
- Flag if `time_added_minutes > 30` (warning threshold, configurable)

**4.5 Driver App Push**
- `app/merge/driver_notifier.py`: `notify_driver(route_id, updated_stops, impact) -> bool`
- Build FCM notification payload: `{ title: "Route Updated", body: "X new stops added. Revised ETA: HH:MM", data: { route_id, new_stops_json, updated_sequence_json } }`
- `firebase_admin.messaging.send(message)` → log success/failure
- Store `late_merge_event` record: `(route_id, manifest_upload_id, stops_added, notified_at, driver_acknowledged_at=None)`
- Celery beat task `check_driver_acknowledgements`: every 2 minutes, check for merge events where `notified_at < now() - 2 minutes` and `driver_acknowledged_at IS NULL` → send follow-up Slack alert

**4.6 Merge Report API + UI**
- `GET /api/v1/manifests/{id}/merge-report` (as specified in Module 6)
- `src/features/late-merge/LateMergeApprovalPage.tsx`:
  - Route impact table with sortable columns
  - Map view: existing stops in blue, new stops in red, affected route paths highlighted
  - "Approve" button → POST approve endpoint → `useQueryClient().invalidateQueries(['manifest', id])` → show confirmation toast
  - Auto-approval: if `ClientConfig.auto_approve_late_merge = True` AND `unassignable_stops.length === 0`, merge is applied without showing approval screen; ops manager sees only a toast notification

#### Week 11: Automation Monitoring + Performance Optimisation

**4.7 Auto-Processing Reliability Monitor**
- `app/services/automation_monitor.py`:
  - Daily Celery beat task `generate_automation_report` at 08:00 local time
  - Metrics computed: manifests processed today (total, auto-processed, required human review, failed); average processing time; anomaly rates by type; schema auto-apply rate per client
  - Send to Datadog custom metrics: `manifest.auto_processed_count`, `manifest.human_review_count`, `manifest.processing_seconds_p95`, `manifest.anomaly_rate`
  - Send daily summary Slack message to `#dispatch-ops` channel: "Daily Manifest Summary: 6 manifests processed. 5 auto-processed. 1 required review (Client B — new column format detected). 47 total anomalies (2 critical, resolved). All clear for dispatch."

**4.8 Datadog Dashboard**
- `infrastructure/datadog/manifest_automation_dashboard.json`: create Datadog dashboard with:
  - Pipeline processing time histogram (P50, P95, P99)
  - Auto-process rate over time (line chart; target trend toward 95%)
  - Anomaly rate by type (stacked bar chart)
  - LLM API call latency and cost (from custom metrics)
  - GNAF / Google Maps / HERE fallback rates (pie chart)
  - Address validation coverage (% GNAF exact, GNAF fuzzy, Google fallback, unresolvable)
  - Late manifest merge frequency and average stops merged

**4.9 Performance Optimisation**
- Address validation batch size: tune async semaphore from 50 to optimal value based on GNAF RDS performance testing; target <20 seconds for 3,000 records
- LLM call caching: if two manifests from the same client in the same day have identical header fingerprints, reuse the `FieldMappingResult` from the first call (cache in Redis with key `llm_mapping:{header_fingerprint}:{client_id}`, TTL 24 hours)
- Staging record bulk insert: ensure SQLAlchemy bulk insert uses `executemany` with `insert().values()`; verify no N+1 patterns in transformation service using `sqlalchemy-utils` query counter in tests
- Celery task prioritisation: late manifest merge tasks get high priority queue (`CELERY_TASK_ROUTES = {'app.tasks.merge.*': {'queue': 'high_priority'}}`) to ensure sub-90-second processing

**4.10 SFTP Ingestion (Complete)**
- `infrastructure/terraform/sftp.tf`: AWS Transfer Family SFTP server with S3 backend
- One SFTP directory per client: `/uploads/{client_id}/`
- S3 event notification on `s3:ObjectCreated:*` in `sftp-drop-{env}` bucket → SNS → SQS → Lambda trigger
- `infrastructure/lambdas/sftp-trigger/handler.py`: extract `client_id` from S3 key; enqueue `process_manifest_from_s3.delay(s3_bucket, s3_key, client_id)`
- `app/tasks/s3_ingestion.py`: `process_manifest_from_s3(bucket, key, client_id)` — download file from S3; compute hash; check for duplicate; create `ManifestUpload` record; enqueue `process_manifest` chain

#### Week 12: Production Readiness + Launch

**4.11 Email Attachment Ingestion (Complete)**
- `infrastructure/lambdas/ses-attachment-handler/handler.py`:
  - Triggered by SES action rule: email to `manifests@*.odocs.app` → stored in S3 → Lambda triggered
  - Parse MIME with Python `email` stdlib
  - Extract attachments (skip inline images, skip attachments >10MB)
  - Identify client from `From:` address: query `client_configs.allowed_sender_emails` via DB call (Lambda uses same RDS via IAM auth)
  - Reject unknown senders: bounce email with explanation
  - Call FastAPI `/api/v1/manifests/upload` with extracted file bytes and `client_id` (using service-to-service API key stored in Lambda env vars)

**4.12 Security Review**
- Run `bandit -r app/` (Python security linter); resolve all HIGH severity findings
- OWASP checklist:
  - File upload: ClamAV scan, MIME type validation, size limit, file extension allowlist — confirmed
  - API auth: JWT expiry (1 hour), refresh token rotation, API key bcrypt hashing — confirm
  - SQL injection: all queries use SQLAlchemy parameterised queries — verify no raw string interpolation in queries (`grep -r "execute(" app/` + manual review)
  - S3 bucket policy: `manifest-raw-uploads` bucket must be private (no public access); confirm bucket ACL
  - Secrets: confirm all credentials in AWS Secrets Manager; confirm no secrets in `.env` committed to Git (`.gitignore` enforced + `git-secrets` pre-commit hook)

**4.13 Ops Manager Onboarding Documentation**
- `docs/ops-manager-guide.md` (internal): how to use the portal for daily workflow; how to read confidence indicators; when to accept vs. correct anomalies; what to do when a manifest fails processing
- Recorded Loom video walkthrough (5 minutes): upload → auto-process → review anomalies → approve → dispatch

**4.14 Load Testing**
- `tests/load/locust_manifest_upload.py`: Locust test simulating 10 concurrent manifest uploads; each manifest 1,000 rows CSV; target P95 processing time <90 seconds; no errors
- Run against staging: `locust -f tests/load/locust_manifest_upload.py --host=https://staging.odocs.app --users=10 --spawn-rate=2 --run-time=10m`
- Fix any bottlenecks identified before production launch

**4.15 Production Deployment**
- ECS Fargate task definitions: `manifest-api` (FastAPI + Uvicorn, 2 tasks, 1 vCPU / 2GB each); `manifest-worker` (Celery worker, 4 tasks, 2 vCPU / 4GB each — more CPU for ML scoring and address validation)
- ECS service auto-scaling: scale `manifest-worker` based on SQS queue depth (`manifest-processing` queue) — scale up when >5 messages queued
- RDS PostgreSQL: `db.t3.large` for application DB; `db.t3.medium` for GNAF DB — both Multi-AZ in production
- ALB health check: `GET /health` returns 200; checks DB connectivity and Redis connectivity
- CloudWatch alarms: `5xx rate > 1%`, `manifest processing time P95 > 120 seconds`, `Celery queue depth > 20` — all alert to `#alerts-critical` Slack channel

---

### Key Technical Decisions — Phase 4

**Decision: OR-Tools local search vs. full solve for late merge**
- Decision: Local search with 5-second time limit
- Rationale: The late merge needs to complete in <90 seconds total pipeline time. A full re-solve of a 60-stop route takes 15–30 seconds alone. Local search starting from the existing solution and inserting new stops is fast enough and produces routes that are within 5% of optimal. The ops manager cannot tell the difference between a 98% optimal route and a 100% optimal route; they need the update fast.

**Decision: Firebase FCM vs APNS/FCM direct for driver push**
- Decision: Firebase Admin SDK (FCM) — handles both Android and iOS in one SDK
- Rationale: Driver fleet uses a mix of Android and iOS devices. FCM handles fan-out to both platforms. Single SDK, single integration point.

**Decision: Auto-approval threshold for late manifest merge**
- Decision: Auto-approve if: (a) `ClientConfig.auto_approve_late_merge = True`, (b) 0 unassignable stops, (c) no route's `time_added_minutes > 45` (configurable `max_auto_approve_extension_minutes`)
- Rationale: Give the operator control. Default is manual approval (conservative). Operators can enable auto-approve once they trust the system. The time extension limit ensures the ops manager is alerted when a late manifest materially impacts the day's schedule.

---

### Definition of Done — Phase 4

- [ ] Late manifest merge engine processes <200 late parcels in <90 seconds end-to-end
- [ ] Affected drivers receive push notification within 30 seconds of merge approval
- [ ] Unassignable stops are clearly flagged in the merge approval UI
- [ ] Auto-approval works correctly when configured and all conditions are met
- [ ] SFTP ingestion end-to-end tested: file dropped to SFTP → processed → records in routing system
- [ ] Email ingestion end-to-end tested: email with attachment sent → attachment extracted → processed
- [ ] Daily automation report sending to Slack at 08:00
- [ ] Datadog dashboard showing all key metrics, accessible by ops manager and engineering
- [ ] All 4 current client formats processing automatically (auto_process_enabled = True) after learning period
- [ ] Load test: 10 concurrent uploads, P95 <90 seconds, 0 errors
- [ ] Security review complete; no HIGH severity bandit findings; OWASP checklist complete
- [ ] Ops manager has completed UAT and signed off on the daily workflow
- [ ] Production deployment verified: all ECS tasks healthy; alarms configured; CloudWatch logs flowing

---

### Risks & Mitigations — Phase 4

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| OR-Tools partial re-optimisation produces routes that drivers find confusing (out-of-sequence additions) | Medium | Medium | Add a human-readable explanation to the driver push notification: "New stop added between Stop 7 and Stop 8: [address]." Test with 3–5 drivers in Week 10 before full rollout. |
| FCM push notifications not received by drivers on poor mobile coverage | Medium | Medium | Implement in-app notification as fallback (fetched by driver app on next background sync). Also set `time_to_live=3600` on FCM messages so they are delivered when the device comes online. |
| Production load higher than staging load test revealed | Low | High | Set ECS auto-scaling to trigger at SQS queue depth >5 (aggressive scale-out). Monitor first 2 weeks of production closely; adjust scaling thresholds based on real patterns. |
| Ops manager resistance to relying on the automated system | Medium | Medium | Keep manual override accessible at every step. The automation should feel like a helpful assistant that can be overruled, not an opaque black box. Spend Week 12 training time on how to disable auto-processing for a client if needed. |
| Database migration time on `delivery_records` (large table in production) | Low | High | Run `CREATE INDEX CONCURRENTLY` for all new indexes (non-blocking). Test all migrations on a production-sized data copy before deployment window. Schedule production deployment during off-peak hours (Saturday night). |

---

## Cross-Cutting Concerns (All Phases)

### Environment Variables Reference

All environment variables documented in `.env.example`:

```bash
# Database
DATABASE_URL=postgresql+asyncpg://user:pass@host:5432/odocs
GNAF_DATABASE_URL=postgresql+asyncpg://user:pass@gnaf-host:5432/gnaf

# Redis
REDIS_URL=redis://host:6379/0

# AWS
AWS_S3_BUCKET_RAW=manifest-raw-uploads
AWS_S3_BUCKET_MODELS=manifest-models
AWS_REGION=ap-southeast-2

# LLM
ANTHROPIC_API_KEY=sk-ant-...

# Address APIs
GOOGLE_MAPS_API_KEY=AIza...
HERE_API_KEY=...

# Routing integration
ROUTING_WEBHOOK_URL=https://...
ROUTING_WEBHOOK_SECRET=...

# Notifications
SLACK_ANOMALY_WEBHOOK_URL=https://hooks.slack.com/...
FCM_SERVER_KEY=...

# Security
CLAMAV_API_URL=http://clamav:8080
JWT_SECRET_KEY=...
JWT_ALGORITHM=HS256

# Feature flags
MAX_UPLOAD_SIZE_MB=50
AUTO_PROCESS_CONFIDENCE_THRESHOLD=0.95
LATE_MANIFEST_CUTOFF_TIME=07:00
MAX_AUTO_APPROVE_EXTENSION_MINUTES=45
ISOLATION_FOREST_CONTAMINATION=0.05
```

### Testing Strategy

- **Unit tests** (`pytest`): all parser logic, transformation service, rule engine rules, feature engineering, address normalisation. Minimum 80% line coverage.
- **Integration tests** (`pytest` with `testcontainers`): PostgreSQL + Redis containers for DB-touching services; mock `anthropic`, `boto3`, GNAF, Google Maps APIs using `respx` (httpx mock library).
- **End-to-end tests** (staging environment): real Claude API calls with test manifests; real GNAF validation; real Google Maps. Run nightly via GitHub Actions against staging.
- **Performance tests** (`locust`): run on staging before each production release.
- **Fixture files** (`tests/fixtures/`): one sanitised sample manifest per input format per client. These are the golden test cases. Any pipeline change that breaks these fixtures is a regression.

### Logging Standards

All logs use `structlog` with JSON output. Every log entry for manifest processing includes:
- `manifest_upload_id` (UUID string)
- `client_id` (UUID string)
- `stage` (pipeline stage name)
- `correlation_id` (set at upload; propagated through Celery task chain via task kwargs)

This enables Datadog Log Management to filter all log entries for a single manifest's processing chain with one query: `@manifest_upload_id:"{uuid}"`.
