# Challenge 2: Inbound Data Normalisation & Manifest Automation — System Architecture

---

## Tech Stack

### Frontend

| Layer | Technology | Rationale |
|---|---|---|
| UI Framework | React 18 + TypeScript 5 | Component model suits the review interface pattern (per-field confidence indicators, inline editing). TypeScript prevents the class of field-mapping bugs that affect manifest integrity. |
| State Management | Zustand | Lightweight; the review UI is not complex enough to warrant Redux. Zustand's subscription model works well for real-time anomaly feed updates. |
| Data Fetching | React Query (TanStack Query v5) | Built-in loading/error states, background refetch for the processing status polling pattern. |
| UI Components | shadcn/ui + Tailwind CSS | Rapid, accessible component assembly without a heavy component library dependency. |
| Build Tooling | Vite | Faster dev server and build than CRA or Next.js for this use case (no SSR required). |
| File Upload | react-dropzone | Handles drag-and-drop multi-file upload with MIME type validation and progress feedback. |
| Table Rendering | TanStack Table v8 | Virtual scrolling required for 3,000-row manifest review. TanStack Table handles this natively. |

### Backend

| Layer | Technology | Rationale |
|---|---|---|
| API Framework | Python 3.12 + FastAPI 0.110+ | Async-native, automatic OpenAPI docs, Pydantic v2 for strict schema validation at every boundary. |
| Task Queue | Celery 5 + Redis 7 | Manifest processing is I/O-heavy (API calls to Claude, GNAF, Google Maps) and must not block the HTTP response. Celery with Redis broker handles this. Redis also serves as the result backend. |
| ORM | SQLAlchemy 2.0 (async) + Alembic | Async SQLAlchemy for non-blocking DB operations. Alembic for migration management. |
| Validation Layer | Pydantic v2 | Strict model definitions for every schema boundary. Used throughout: API request/response models, internal data transfer objects, LLM output parsing. |
| File Processing | See Format Parser Library section | |
| HTTP Client | httpx (async) | Async HTTP client for all outbound API calls (Claude, GNAF, Google Maps, HERE). |

### LLM

| Component | Technology | Notes |
|---|---|---|
| Provider | Anthropic Claude API | Model: `claude-sonnet-4-6` for schema inference (sufficient capability, lower latency and cost than Opus). |
| SDK | `anthropic` Python SDK 0.25+ | Official SDK; handles retry logic, streaming, structured output. |
| Structured Output | JSON mode with schema specification | Claude API supports specifying a JSON schema; responses validated against Pydantic models before use. |
| Prompt Management | Stored in `prompts/` directory as `.txt` files | Version-controlled prompt templates. Loaded at startup, not hardcoded. |

### File Parsing

| Format | Library | Version |
|---|---|---|
| CSV (any delimiter) | Python stdlib `csv` + `chardet` for encoding detection | stdlib |
| Excel (.xlsx, .xls) | `openpyxl` (xlsx) + `xlrd` (xls legacy) | openpyxl 3.1+, xlrd 2.0+ |
| PDF table extraction | `pdfplumber` (primary) + `camelot-py` (fallback for complex tables) | pdfplumber 0.10+, camelot-py 0.11+ |
| EDI X12 204 | `pyx12` (Python X12 parser) | pyx12 2.0+ |
| JSON / JSONL | Python stdlib `json` + `jsonlines` | stdlib + jsonlines 3.1 |

### Address Validation

| Service | Role | Fallback Order |
|---|---|---|
| GNAF / PSMA Data API | Primary Australian address authority. Exact and fuzzy match. Geocoordinates, suburb, state, postcode, LGA. | Primary |
| Google Maps Geocoding API | Fallback geocoding. Handles addresses not in GNAF (new subdivisions, rural, PO Boxes). | Secondary |
| HERE Geocoding API | Secondary fallback if Google Maps quota exceeded or unavailable. | Tertiary |
| RapidFuzz (local) | In-process fuzzy string matching against GNAF address tokens for correction suggestions. | Runs locally; no API call |

### Anomaly Detection

| Component | Technology |
|---|---|
| Statistical anomaly detection | scikit-learn 1.4+ `IsolationForest` |
| Model serialisation | `joblib` for model persistence |
| Rule engine | Custom Python rule classes; no external library required |
| Feature engineering | `pandas` 2.0+ for manifest-level feature computation |

### Infrastructure

| Component | Technology | Notes |
|---|---|---|
| Queue / Cache | Redis 7 | Celery broker + result backend + session cache |
| Database | PostgreSQL 16 | Primary data store. JSONB columns for flexible schema profile storage. |
| Raw file storage | AWS S3 | All uploaded files archived in original form. Bucket: `manifest-raw-uploads`. |
| Secrets management | AWS Secrets Manager | `ANTHROPIC_API_KEY`, `GOOGLE_MAPS_API_KEY`, `HERE_API_KEY`, `GNAF_API_KEY`, database credentials. |
| Containerisation | Docker + Docker Compose (dev) / ECS Fargate (prod) | |
| Monitoring | Datadog | APM traces on all FastAPI endpoints and Celery tasks. Custom metrics: manifest processing time, LLM call latency, address validation hit rates, anomaly rates. |
| Logging | structlog → Datadog Log Management | JSON-structured logs. Correlation ID on every manifest processing chain. |
| CI/CD | GitHub Actions | Lint (ruff), type check (mypy), test (pytest), build, push to ECR, deploy to ECS. |

---

## AI Architecture

The full manifest processing pipeline runs as a Celery task chain, triggered asynchronously after file upload. The HTTP endpoint returns immediately with a `manifest_upload_id` and a WebSocket or polling URL for status updates.

### Pipeline Overview

```
[File Upload]
     │
     ▼
[Format Detection]
     │
     ▼
[File Parser] ──────────────────────────────────────────► [S3 Archive]
     │
     ▼
[Raw Record Extraction]
     │
     ▼
[LLM Schema Inference] ◄──── [Client Schema Profile Cache]
     │
     ├── confidence ≥ 0.95 ──► [Auto-approved mapping]
     │
     └── confidence < 0.95 ──► [Human Review Queue] ──► [Mapping confirmed by ops manager]
               │
               ▼
     [Confirmed Field Mapping]
               │
               ▼
     [Record Transformation]  (apply mapping; normalise values)
               │
               ▼
     [Address Validation]  ◄──── [GNAF API / Google Maps / HERE / RapidFuzz]
               │
               ▼
     [Anomaly Detection]  ◄──── [Isolation Forest model / Rule engine / FTD history DB]
               │
               ├── CRITICAL anomalies ──► [Block routing; alert dispatcher]
               │
               └── WARNING/INFO ──► [Anomaly Report attached to manifest]
                         │
                         ▼
               [Human Review Queue (anomalies)]
                         │
                         ▼ (after review/approval)
               [Clean Delivery Records]
                         │
                         ▼
               [Downstream Routing System]
```

### Stage-by-Stage Specification

#### Stage 1: File Upload
- **Trigger**: POST to `/api/v1/manifests/upload` (multipart form data) OR SFTP drop detected by polling job OR email attachment received
- **Input**: Raw file bytes, `client_id`, `upload_source` (`portal`|`sftp`|`api`|`email`)
- **Operations**: Virus scan (ClamAV); MIME type sniff; file size check (<50MB limit); generate `manifest_upload_id` (UUID); write record to `manifest_uploads` table with status `RECEIVED`; enqueue Celery task `process_manifest.delay(manifest_upload_id)`
- **Output**: `manifest_upload_id`, status `RECEIVED`, WebSocket URL for progress
- **Latency target**: <500ms HTTP response
- **Error handling**: File too large → 413 with message; virus detected → quarantine and alert; DB write failure → retry 3x with exponential backoff

#### Stage 2: Format Detection
- **Celery task**: `detect_format` (first in chain)
- **Input**: File bytes from S3
- **Operations**: Check file extension; use `python-magic` for MIME type detection from file bytes (not extension); apply format-specific header checks (e.g., EDI files start with `ISA*`)
- **Output**: `format` enum: `CSV`|`EXCEL_XLSX`|`EXCEL_XLS`|`PDF`|`EDI_X12`|`JSON`|`JSONL`|`UNKNOWN`
- **Latency target**: <100ms
- **Error handling**: `UNKNOWN` format → status `FORMAT_UNRECOGNISED`; alert dispatcher; stop processing

#### Stage 3: File Parser
- **Celery task**: `parse_file` (second in chain)
- **Input**: File bytes, detected format
- **Operations**: Route to appropriate parser (see Format Parser Library section); parse into list of raw dicts (`list[dict[str, Any]]`); capture all original column names; record row count; detect encoding (`chardet`) for CSV; detect delimiter (try tab, comma, pipe, semicolon)
- **Output**: `raw_records: list[dict[str, str]]`, `original_headers: list[str]`, `row_count: int`, `parse_warnings: list[str]`
- **Latency target**: <5 seconds for 5,000-row manifest
- **Archive**: Write original file to S3 at `s3://manifest-raw-uploads/{client_id}/{date}/{manifest_upload_id}/{original_filename}`
- **Error handling**: Parse failure → status `PARSE_FAILED`; store error detail; alert dispatcher; stop processing

#### Stage 4: Raw Record Extraction
- **Celery task**: Inline within `parse_file` task
- **Input**: Parser output
- **Operations**: Normalise all values to strings; strip leading/trailing whitespace; detect empty rows and remove; detect header rows in multi-header Excel files; store raw records in `manifest_raw_records` table with `manifest_upload_id` FK
- **Output**: Clean `raw_records` written to staging table; `raw_record_count` updated on `manifest_uploads`

#### Stage 5: LLM Schema Inference
- **Celery task**: `infer_schema`
- **Input**: `original_headers`, sample of up to 10 raw records, `client_id`, existing `ClientSchemaProfile` if present
- **Operations**:
  1. Check `ClientSchemaProfile` for this `client_id`. If profile exists with overall confidence ≥ 0.95 and the current file's headers match the stored header fingerprint (normalised hash), skip LLM call and use stored mapping → status `SCHEMA_AUTO_APPLIED`
  2. Otherwise, build prompt: system prompt from `prompts/schema_inference_system.txt`; user message contains headers + 10 sample rows formatted as a markdown table; few-shot examples of 5 common manifest formats included in system prompt
  3. Call Claude API: `model=claude-sonnet-4-6`, `max_tokens=2000`, response format specifies JSON schema (see Data Models section for `FieldMappingResult`)
  4. Parse response with Pydantic; validate all required fields present; compute per-field confidence scores
  5. If any field confidence < 0.95 → enqueue in human review queue; status `AWAITING_SCHEMA_REVIEW`
  6. If all fields ≥ 0.95 → auto-approve; update/create `ClientSchemaProfile`; status `SCHEMA_CONFIRMED`
- **Latency target**: 1–3 seconds for LLM call; <500ms for cache hit
- **Error handling**: Claude API timeout → retry up to 3x with 5-second backoff; API error → status `LLM_INFERENCE_FAILED`; alert and fall back to manual mapping

#### Stage 6: Record Transformation
- **Celery task**: `transform_records`
- **Input**: `raw_records`, confirmed `FieldMapping`
- **Operations**: Apply column mapping (rename, reorder); normalise phone numbers to E.164 format (`phonenumbers` library); normalise dates to ISO 8601; normalise state codes to standard abbreviations (QLD, NSW, VIC, etc.); strip currency symbols from weight/value fields; write transformed records to `manifest_staging_records` table
- **Latency target**: <2 seconds for 3,000 records
- **Error handling**: Per-record transformation errors logged as `INFO` anomalies; record not dropped but flagged

#### Stage 7: Address Validation
- **Celery task**: `validate_addresses`
- **Input**: Transformed records with address fields
- **Operations**: For each record, call `AddressIntelligenceService.validate()` (see Module 4 spec); store `AddressValidationResult` per record; compute validation summary: total, exact_match, fuzzy_match, google_fallback, unresolvable
- **Latency target**: <30 seconds for 3,000 records (using asyncio batched GNAF calls, 50 concurrent)
- **Error handling**: GNAF API down → fall back to Google Maps; Google Maps quota exceeded → fall back to HERE; all APIs down → flag all addresses as unvalidated (warning), do not stop pipeline

#### Stage 8: Anomaly Detection
- **Celery task**: `detect_anomalies`
- **Input**: Transformed + address-validated records, `client_id`, `manifest_upload_id`
- **Operations**: Run rule-based checks (see Module 5); run Isolation Forest scoring; cross-reference addresses against FTD history table; generate `AnomalyFlag` records; compute manifest-level anomaly summary
- **Latency target**: <10 seconds for 3,000 records
- **Error handling**: Model load failure → run rule-based checks only; log `WARNING`; alert engineering on-call

#### Stage 9: Human Review or Auto-Approve
- **Decision point** (not a Celery task — a state evaluation):
  - If `CRITICAL` anomalies exist → status `BLOCKED_CRITICAL_ANOMALIES`; alert dispatcher; wait for human resolution
  - If `WARNING` anomalies exist and `ClientConfig.require_anomaly_review = True` → status `AWAITING_ANOMALY_REVIEW`
  - If no `CRITICAL`, and no `WARNING`, or `ClientConfig.auto_approve_warnings = True` → proceed to Stage 10
- **Human review interface**: React UI; ops manager reviews, edits, approves/rejects each anomaly; approval recorded in audit trail

#### Stage 10: Clean Delivery Records
- **Celery task**: `finalise_records`
- **Input**: Approved records from staging table
- **Operations**: Write to `delivery_records` table with `manifest_upload_id` FK; update `manifest_uploads.status` to `PROCESSED`; compute final record count; trigger downstream webhook
- **Latency target**: <2 seconds for 3,000 records (bulk insert)

#### Stage 11: Downstream Routing
- **Celery task**: `dispatch_to_routing`
- **Input**: `manifest_upload_id`, list of `delivery_record_id`s
- **Operations**: POST to routing system webhook (`ROUTING_WEBHOOK_URL` env var); or write directly to routing DB if same-platform deployment; send confirmation webhook to client system if `ClientConfig.webhook_url` set
- **Latency target**: <5 seconds
- **Error handling**: Webhook failure → retry 5x with exponential backoff; if all retries fail → alert dispatcher with manual action instructions

### End-to-End Latency Budget (3,000-record manifest)

| Stage | Latency |
|---|---|
| Upload + S3 archive | <1s |
| Format detection | <0.1s |
| File parsing | <3s |
| LLM schema inference (LLM call) | 1–3s |
| Record transformation | <2s |
| Address validation (batched async) | <25s |
| Anomaly detection | <10s |
| DB finalisation | <2s |
| **Total (auto-approve path)** | **<50 seconds** |
| **Total with human schema review** | **50 seconds + review time (~2–5 min)** |

Target of <90 seconds for 1,000-parcel manifest is comfortably met on the auto-approve path. For 3,000 parcels, the pipeline completes in under 50 seconds before any human review.

---

## Integration Layer

### Client Upload Methods

#### 1. Portal UI Upload (primary for most clients)
- **URL**: `POST /api/v1/manifests/upload`
- **Auth**: JWT bearer token (per-client API key mapped to `client_id` in DB)
- **Body**: `multipart/form-data` with `file` field and optional `manifest_date`, `client_reference`
- **Response**: `{ manifest_upload_id, status, progress_url }`
- **Progress updates**: WebSocket at `wss://api.odocs.app/ws/manifests/{manifest_upload_id}` emits `{ stage, progress_pct, message }` as each pipeline stage completes

#### 2. SFTP Drop Zone
- **Server**: AWS Transfer Family (managed SFTP)
- **Path structure**: `/uploads/{client_id}/` — each client has their own directory
- **Trigger**: S3 event notification on `PutObject` in `sftp-drop-{env}` bucket → triggers Lambda → enqueues Celery task
- **Auth**: SSH public key per client, managed in AWS Transfer Family console
- **Polling fallback**: Celery beat task runs every 5 minutes to check for unprocessed files if S3 event missed
- **Duplicate prevention**: SHA-256 hash of file content stored on `manifest_uploads`; reject if hash already processed within 24 hours

#### 3. REST API (programmatic client systems)
- **URL**: Same `POST /api/v1/manifests/upload` endpoint
- **Content-Type**: `multipart/form-data` (file upload) OR `application/json` (for structured JSON manifests — records passed directly, no file parsing needed)
- **Auth**: API key in `X-API-Key` header
- **Rate limiting**: 60 requests/hour per client (Redis-backed with `slowapi`)
- **JSON body format** (direct upload): `{ client_id, records: [{ ... }], schema_version: "1.0" }` — bypasses LLM inference if schema already known

#### 4. Email Attachment Parsing
- **Ingestion**: AWS SES receives email at `manifests@{client_domain}.odocs.app`; S3 stores raw email; Lambda parses MIME, extracts attachments, enqueues Celery task with `upload_source=email`
- **Client identification**: From address matched against `ClientConfig.allowed_sender_emails`
- **Security**: Only pre-registered sender addresses accepted; others quarantined and flagged
- **Limitations**: Max 10MB attachment; single attachment per email (multi-attachment emails flagged for manual handling)

#### 5. EDI 204 (enterprise clients)
- **Protocol**: AS2 (Applicability Statement 2) over HTTPS, or SFTP for clients without AS2 capability
- **Format**: ANSI X12 EDI 204 (Motor Carrier Load Tender)
- **Endpoint**: Dedicated EDI ingestion queue; AWS Transfer Family for SFTP; custom AS2 receiver for enterprise
- **Processing**: Routed to `EDI204Parser`; parsed into raw records; remainder of pipeline identical to CSV path

### Output Integrations

#### Routing System Webhook
- **Trigger**: On manifest status transition to `PROCESSED`
- **Method**: `POST {ROUTING_WEBHOOK_URL}` (env var)
- **Body**: `{ manifest_upload_id, delivery_record_ids: [...], record_count, client_id, manifest_date }`
- **Auth**: HMAC-SHA256 signature in `X-Webhook-Signature` header (shared secret in env var)
- **Retry**: 5 attempts, exponential backoff starting at 10 seconds

#### Direct Database Write (same-platform routing)
- **When**: Routing system shares the same PostgreSQL instance (Phase 1)
- **Mechanism**: `delivery_records` table written directly; routing system polls for new records with `status=READY_FOR_ROUTING`

#### CSV Export (external routing tools)
- **URL**: `GET /api/v1/manifests/{manifest_upload_id}/export?format=csv`
- **Auth**: JWT
- **Output**: CSV conforming to the routing tool's expected format (format specified by `ClientConfig.export_format`)
- **Use case**: Operators using OptimoRoute or Routific as their routing tool can export clean, validated records in the tool's required format

#### Client Webhook (processing confirmation)
- **Trigger**: On manifest `PROCESSED` or `FAILED`
- **Config**: `ClientConfig.webhook_url`, `ClientConfig.webhook_secret`
- **Body**: `{ manifest_upload_id, status, record_count, anomaly_summary, processed_at }`
- **Use case**: Client's WMS or order management system receives confirmation that their manifest was processed; can trigger order status updates in their system

#### EDI 990 Response (enterprise clients)
- **Trigger**: On successful processing of an EDI 204 manifest
- **Format**: ANSI X12 EDI 990 (Response to a Load Tender)
- **Transmission**: AS2 or SFTP back to client's EDI endpoint
- **Content**: Acceptance/rejection code, reference number, error details if rejected

---

## Data Models

All models defined as SQLAlchemy 2.0 declarative models with Pydantic v2 schema counterparts. JSONB used for flexible fields.

### ManifestUpload

```
manifest_uploads
├── id: UUID (PK)
├── client_id: UUID (FK → clients.id)
├── upload_source: ENUM('portal', 'sftp', 'api', 'email', 'edi')
├── original_filename: VARCHAR(255)
├── s3_key: VARCHAR(512)  -- path to raw file in S3
├── file_hash_sha256: VARCHAR(64)  -- for deduplication
├── file_size_bytes: BIGINT
├── detected_format: ENUM('CSV','EXCEL_XLSX','EXCEL_XLS','PDF','EDI_X12','JSON','JSONL','UNKNOWN')
├── status: ENUM('RECEIVED','PARSING','SCHEMA_INFERENCE','AWAITING_SCHEMA_REVIEW',
│               'SCHEMA_CONFIRMED','TRANSFORMING','VALIDATING_ADDRESSES',
│               'DETECTING_ANOMALIES','AWAITING_ANOMALY_REVIEW',
│               'BLOCKED_CRITICAL_ANOMALIES','PROCESSING','PROCESSED',
│               'PARSE_FAILED','LLM_INFERENCE_FAILED','FORMAT_UNRECOGNISED','FAILED')
├── raw_record_count: INTEGER
├── clean_record_count: INTEGER
├── anomaly_count_critical: INTEGER
├── anomaly_count_warning: INTEGER
├── anomaly_count_info: INTEGER
├── processing_started_at: TIMESTAMPTZ
├── processing_completed_at: TIMESTAMPTZ
├── manifest_date: DATE  -- the intended delivery date for this manifest
├── client_reference: VARCHAR(255)  -- client's own manifest ID
├── celery_task_id: VARCHAR(255)
├── created_at: TIMESTAMPTZ (default now())
└── updated_at: TIMESTAMPTZ (auto-updated)
```

### ManifestRawRecord

```
manifest_raw_records
├── id: UUID (PK)
├── manifest_upload_id: UUID (FK → manifest_uploads.id, CASCADE DELETE)
├── row_index: INTEGER  -- original row number in source file
├── raw_data: JSONB  -- { "original_column_name": "raw_value", ... }
└── created_at: TIMESTAMPTZ
```

### FieldMapping

```
field_mappings
├── id: UUID (PK)
├── manifest_upload_id: UUID (FK → manifest_uploads.id)
├── client_schema_profile_id: UUID (FK → client_schema_profiles.id, NULLABLE)
├── source_column: VARCHAR(255)  -- original column name from file
├── canonical_field: VARCHAR(100)  -- e.g. 'recipient_name', 'address_line_1'
├── confidence: FLOAT  -- 0.0–1.0 from LLM
├── llm_reasoning: TEXT  -- LLM's explanation for this mapping
├── human_confirmed: BOOLEAN (default FALSE)
├── human_override: VARCHAR(100)  -- if ops manager overrode the LLM mapping
├── confirmed_by: UUID (FK → users.id, NULLABLE)
├── confirmed_at: TIMESTAMPTZ (NULLABLE)
└── created_at: TIMESTAMPTZ
```

Canonical fields enum:
`recipient_name`, `address_line_1`, `address_line_2`, `suburb`, `state`, `postcode`, `country`, `phone_mobile`, `phone_landline`, `email`, `parcel_id`, `order_id`, `client_reference`, `weight_kg`, `dimensions_cm`, `item_count`, `special_instructions`, `delivery_window_start`, `delivery_window_end`, `service_type`, `is_fragile`, `requires_signature`, `cod_amount`

### ClientSchemaProfile

```
client_schema_profiles
├── id: UUID (PK)
├── client_id: UUID (FK → clients.id)
├── profile_name: VARCHAR(255)  -- e.g. 'Shopify Export v2' (auto-named by LLM or manually set)
├── header_fingerprint: VARCHAR(64)  -- SHA-256 of normalised sorted column names
├── column_mappings: JSONB  -- { "source_column": { "canonical_field": "...", "confidence": 0.98, ... } }
├── auto_process_enabled: BOOLEAN (default FALSE)
├── auto_process_confidence_threshold: FLOAT (default 0.95)
├── successful_upload_count: INTEGER (default 0)
├── last_used_at: TIMESTAMPTZ
├── created_at: TIMESTAMPTZ
└── updated_at: TIMESTAMPTZ
```

Auto-processing is enabled automatically when `successful_upload_count >= 3` and all field confidences remain above threshold.

### AddressValidationResult

```
address_validation_results
├── id: UUID (PK)
├── manifest_record_id: UUID (FK → manifest_staging_records.id)
├── input_address_raw: TEXT  -- original address string from manifest
├── validation_method: ENUM('GNAF_EXACT','GNAF_FUZZY','GOOGLE_MAPS','HERE','UNRESOLVED')
├── gnaf_pid: VARCHAR(20)  -- GNAF Persistent Identifier if matched
├── validated_address_line_1: VARCHAR(255)
├── validated_suburb: VARCHAR(100)
├── validated_state: VARCHAR(10)
├── validated_postcode: VARCHAR(10)
├── latitude: DECIMAL(10, 7)
├── longitude: DECIMAL(10, 7)
├── confidence: FLOAT  -- 0.0–1.0
├── correction_applied: BOOLEAN
├── original_vs_corrected_diff: JSONB  -- { "field": "postcode", "original": "4001", "corrected": "4000" }
├── delivery_zone: VARCHAR(50)  -- assigned from postcode/coord lookup
├── ftd_rate_historical: FLOAT  -- fraction of historical deliveries to this address that failed first attempt
├── is_deliverable: BOOLEAN
├── undeliverable_reason: TEXT (NULLABLE)
└── created_at: TIMESTAMPTZ
```

### AnomalyFlag

```
anomaly_flags
├── id: UUID (PK)
├── manifest_upload_id: UUID (FK → manifest_uploads.id)
├── manifest_record_id: UUID (FK → manifest_staging_records.id, NULLABLE)  -- NULL = manifest-level anomaly
├── anomaly_type: ENUM('EXACT_DUPLICATE','NEAR_DUPLICATE','MISSING_PHONE',
│                      'MISSING_POSTCODE','MISSING_ADDRESS','INVALID_PHONE_FORMAT',
│                      'INVALID_POSTCODE','HIGH_FTD_RATE','SUSPICIOUS_VOLUME',
│                      'ZERO_WEIGHT','UNDELIVERABLE_ADDRESS','STATISTICAL_OUTLIER',
│                      'SUSPICIOUS_PATTERN')
├── severity: ENUM('CRITICAL','WARNING','INFO')
├── score: FLOAT  -- 0.0–1.0; higher = more anomalous (from Isolation Forest or rule confidence)
├── description: TEXT  -- human-readable explanation
├── suggested_action: TEXT  -- what the ops manager should do
├── related_record_ids: UUID[]  -- for NEAR_DUPLICATE, points to the duplicate record
├── resolution_status: ENUM('UNRESOLVED','ACCEPTED','CORRECTED','REJECTED')
├── resolved_by: UUID (FK → users.id, NULLABLE)
├── resolved_at: TIMESTAMPTZ (NULLABLE)
├── resolution_note: TEXT (NULLABLE)
└── created_at: TIMESTAMPTZ
```

### ManifestStagingRecord

```
manifest_staging_records
├── id: UUID (PK)
├── manifest_upload_id: UUID (FK → manifest_uploads.id)
├── raw_record_id: UUID (FK → manifest_raw_records.id)
├── recipient_name: VARCHAR(255)
├── address_line_1: VARCHAR(255)
├── address_line_2: VARCHAR(255, NULLABLE)
├── suburb: VARCHAR(100)
├── state: VARCHAR(10)
├── postcode: VARCHAR(10)
├── country: VARCHAR(50) (default 'AU')
├── phone_mobile: VARCHAR(20, NULLABLE)
├── phone_landline: VARCHAR(20, NULLABLE)
├── email: VARCHAR(255, NULLABLE)
├── parcel_id: VARCHAR(100)
├── order_id: VARCHAR(100, NULLABLE)
├── client_reference: VARCHAR(100, NULLABLE)
├── weight_kg: DECIMAL(8, 3, NULLABLE)
├── item_count: INTEGER (default 1)
├── special_instructions: TEXT (NULLABLE)
├── delivery_window_start: TIME (NULLABLE)
├── delivery_window_end: TIME (NULLABLE)
├── service_type: VARCHAR(50, NULLABLE)
├── requires_signature: BOOLEAN (default FALSE)
├── is_fragile: BOOLEAN (default FALSE)
├── transformation_warnings: JSONB  -- list of non-fatal issues during transformation
├── status: ENUM('STAGING','VALIDATED','ANOMALY_FLAGGED','APPROVED','REJECTED')
└── created_at: TIMESTAMPTZ
```

### DeliveryRecord

```
delivery_records
├── id: UUID (PK)
├── manifest_upload_id: UUID (FK → manifest_uploads.id)
├── staging_record_id: UUID (FK → manifest_staging_records.id)
├── client_id: UUID (FK → clients.id)
├── delivery_date: DATE
├── -- All address and recipient fields mirrored from staging record --
├── -- (denormalised for performance; routing queries this table heavily) --
├── validated_latitude: DECIMAL(10, 7)
├── validated_longitude: DECIMAL(10, 7)
├── delivery_zone: VARCHAR(50)
├── routing_status: ENUM('PENDING_ROUTING','ASSIGNED','IN_ROUTE','COMPLETED','FAILED','CANCELLED')
├── route_id: UUID (FK → routes.id, NULLABLE)
├── stop_sequence: INTEGER (NULLABLE)
├── driver_id: UUID (FK → drivers.id, NULLABLE)
└── created_at: TIMESTAMPTZ
```

### ClientConfig

```
client_configs
├── id: UUID (PK)
├── client_id: UUID (FK → clients.id, UNIQUE)
├── auto_process_threshold: FLOAT (default 0.95)
├── require_anomaly_review: BOOLEAN (default TRUE)
├── auto_approve_warnings: BOOLEAN (default FALSE)
├── allowed_sender_emails: TEXT[]  -- for email ingestion
├── sftp_directory: VARCHAR(255, NULLABLE)
├── webhook_url: VARCHAR(512, NULLABLE)
├── webhook_secret: VARCHAR(255, NULLABLE)
├── export_format: ENUM('INTERNAL','OPTIMOROUTE_CSV','ROUTIFIC_JSON','ONFLEET_CSV') (default 'INTERNAL')
├── edi_sender_id: VARCHAR(50, NULLABLE)  -- EDI ISA sender ID
├── edi_receiver_id: VARCHAR(50, NULLABLE)
├── timezone: VARCHAR(50) (default 'Australia/Brisbane')
├── late_manifest_cutoff_time: TIME (default '07:00')
├── created_at: TIMESTAMPTZ
└── updated_at: TIMESTAMPTZ
```

### Indexes and Performance Considerations

Key indexes for query performance on high-volume tables:
- `manifest_raw_records`: index on `manifest_upload_id`
- `manifest_staging_records`: index on `manifest_upload_id`, `status`; partial index on `parcel_id` where not null
- `delivery_records`: index on `delivery_date`, `client_id`, `routing_status`; index on `validated_latitude`, `validated_longitude` (for geo queries — consider PostGIS extension)
- `anomaly_flags`: index on `manifest_upload_id`, `severity`, `resolution_status`
- `address_validation_results`: index on `gnaf_pid` (for FTD rate lookups); index on `validated_postcode`

Table partitioning: `delivery_records` and `manifest_staging_records` partitioned by `delivery_date` (monthly range partitions) once record count exceeds ~10M rows.
