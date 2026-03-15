# Challenge 2: Inbound Data Normalisation & Manifest Automation — Modular Breakdown

---

## Module 1: File Ingestion Gateway

### Purpose

The File Ingestion Gateway is the single entry point for all inbound manifest data, regardless of how it arrives. It abstracts over the four intake channels (SFTP, REST API, email, portal UI upload), ensures every file is archived before any processing begins, deduplicates re-submitted files, and hands off to the processing pipeline with a consistent internal event model. It is intentionally thin — it does not parse or interpret files. Its job is: receive, validate receipt, archive, and enqueue.

### Key Functions

- Accept file uploads via REST API endpoint (multipart/form-data) and return immediately with a `manifest_upload_id`
- Poll AWS Transfer Family SFTP directories every 5 minutes for new files; trigger processing on new object detection via S3 event notification (primary) with polling as fallback
- Parse inbound emails via AWS SES + Lambda; extract attachments; identify client by sender email address; reject unrecognised senders
- Accept direct JSON payload via REST API (for clients with programmatic integrations who do not upload files)
- Run lightweight pre-flight checks on every inbound file: size limit (<50MB), MIME type plausibility check, ClamAV virus scan
- Compute SHA-256 hash of file content; reject if hash matches a `manifest_uploads` record created within the past 24 hours (configurable window) — prevents ops manager from accidentally processing a file twice
- Write `ManifestUpload` record to PostgreSQL with status `RECEIVED` before any other work begins
- Archive raw file to S3 at `s3://manifest-raw-uploads/{client_id}/{YYYY-MM-DD}/{manifest_upload_id}/{original_filename}` — this happens synchronously before the HTTP response is returned; the record is never processed without an archived copy
- Enqueue `process_manifest` Celery task with `manifest_upload_id`
- Emit WebSocket progress stream for portal UI clients

### API Endpoints

```
POST   /api/v1/manifests/upload
       Content-Type: multipart/form-data
       Fields: file (required), manifest_date (optional ISO date), client_reference (optional)
       Headers: Authorization: Bearer {jwt} OR X-API-Key: {key}
       Response 202: { manifest_upload_id, status: "RECEIVED", progress_ws_url }
       Response 400: file missing, file too large, unsupported content type
       Response 409: { error: "DUPLICATE_FILE", original_upload_id, original_uploaded_at }
       Response 413: file exceeds 50MB limit

POST   /api/v1/manifests/upload/json
       Content-Type: application/json
       Body: { records: [...], schema_version, manifest_date, client_reference }
       Response 202: { manifest_upload_id, status: "RECEIVED" }

GET    /api/v1/manifests/{manifest_upload_id}/status
       Response 200: { manifest_upload_id, status, stage, progress_pct, record_count,
                       anomaly_summary, created_at, updated_at }

GET    /api/v1/manifests?client_id=&date=&status=&limit=&offset=
       Response 200: { manifests: [...], total, limit, offset }

DELETE /api/v1/manifests/{manifest_upload_id}
       Response 204: marks upload as CANCELLED; does not delete S3 archive
       Response 409: cannot cancel if status is PROCESSED

WS     /ws/manifests/{manifest_upload_id}
       Emits: { stage, progress_pct, message, timestamp } on each pipeline stage transition
```

### State Management

The gateway owns the `manifest_uploads` table and is the only writer to the `status` column during the ingestion phase. Status transitions are enforced by a state machine (Python `transitions` library or custom): `RECEIVED → PARSING → ... → PROCESSED | FAILED`. No other module may set `status` backward or skip states. Invalid transitions raise `InvalidStatusTransitionError` which is logged and alerted.

### Dependencies

- **AWS S3**: raw file archival (via `boto3`)
- **AWS Transfer Family**: SFTP server (managed; no application code dependency)
- **AWS SES + Lambda**: email ingestion (Lambda code lives in `infrastructure/lambdas/ses-attachment-handler/`)
- **Redis**: WebSocket state for progress streaming (via `redis-py` async)
- **PostgreSQL**: `manifest_uploads` table writes
- **Celery**: task enqueue only
- **ClamAV**: virus scan (runs as a sidecar container; called via REST API at `CLAMAV_API_URL` env var)
- **`python-magic`**: MIME type detection
- **`chardet`**: encoding detection (used at this stage to detect encoding of text files for downstream parsers)

---

## Module 2: Format Parser Library

### Purpose

The Format Parser Library provides a unified interface over five distinct file formats. Each parser takes raw file bytes and returns a common `ParseResult` object: a list of raw string dicts (one per data row), the original column headers, row count, and any parse warnings. The parsers are deliberately stateless and have no knowledge of the downstream processing pipeline. They do one job: turn bytes into rows.

### Key Functions

**Common Interface** (`parsers/base.py`):
```python
class BaseParser:
    def parse(self, file_bytes: bytes, **options) -> ParseResult: ...

@dataclass
class ParseResult:
    headers: list[str]
    records: list[dict[str, str]]
    row_count: int
    warnings: list[str]
    encoding_detected: str | None
    sheet_name: str | None  # Excel only
```

**CSV Parser** (`parsers/csv_parser.py`):
- Use `chardet.detect()` on first 10KB to determine encoding; default UTF-8 if confidence <0.7
- Auto-detect delimiter: try comma, tab, pipe, semicolon in that order; select the one producing the most consistent column count across rows
- Handle quoted fields with embedded commas correctly (stdlib `csv.reader`)
- Detect and skip blank rows (all fields empty after strip)
- Detect and report multi-row header scenarios (flag if row 0 and row 1 both contain no numeric-looking values)
- Strip BOM (byte order mark) from UTF-8-BOM encoded files (common from Windows Excel exports)

**Excel Parser** (`parsers/excel_parser.py`):
- `.xlsx`: use `openpyxl`; load workbook with `data_only=True` (resolve formula values)
- `.xls` (legacy Excel 97–2003): use `xlrd 2.0+`
- Sheet selection: default to first non-empty sheet; if multiple sheets found, check `ClientConfig.excel_sheet_name`; if not configured, use first sheet and emit warning
- Handle merged header cells: detect merged regions in first 3 rows; unmerge and propagate values
- Detect and remove subtotal rows: rows where the first column contains keywords `Total`, `Subtotal`, `Grand Total`, `Sum` (case-insensitive)
- Convert all cell values to strings (preserve original representation; do not auto-type)
- Handle date cells: `openpyxl` returns `datetime` objects; convert to `YYYY-MM-DD` string at parse time
- Handle numeric cells: convert to string without scientific notation (use `f"{value:.10g}"`)

**PDF Table Extractor** (`parsers/pdf_parser.py`):
- Primary: `pdfplumber` — extract tables from each page using `page.extract_table()`
- Fallback: `camelot-py` with `flavor='lattice'` (for PDFs with visible grid lines) and `flavor='stream'` (for whitespace-delimited tables)
- Multi-page PDFs: extract tables from all pages; concatenate row lists; handle headers that repeat on each page (detect by comparing first row of each page's table to the first page's header row; discard repeated headers)
- Convert all cell values to strings; strip newlines within cells (common in PDFs)
- Log warning for any page that yields no table
- Return `ParseResult` with `warnings` listing any pages with extraction failures

**EDI X12 204 Parser** (`parsers/edi_parser.py`):
- Use `pyx12` library to parse ISA/GS/ST envelope and 204 transaction set
- Extract key loops: L11 (Reference Numbers), N1 (Name), N3/N4 (Address), S5 (Stop Off Details), L5 (Description/Routing), AT8 (Weight)
- Map EDI elements to intermediate dict with keys matching the canonical delivery schema field names
- Validate ISA control number; log warning if ISA13 not unique
- Handle multiple stops in a single EDI 204 (iterate L5 loop): produce one `ParseResult` record per stop
- Handle both ISA acknowledgement (997/999) generation requirement: set `edi_requires_ack = True` in `ParseResult.metadata`

**JSON/JSONL Parser** (`parsers/json_parser.py`):
- Detect JSONL (newline-delimited JSON): check if content contains multiple top-level objects separated by newlines
- For JSON: support `{ "records": [...] }` wrapper OR top-level array `[...]`
- For JSONL: parse each line independently; collect errors per line without stopping
- Flatten nested objects one level deep: `{ "address": { "line1": "..." } }` → `{ "address.line1": "..." }`
- Convert all values to strings for consistency with other parsers
- Report schema variance: if any record has keys absent in >80% of other records, flag as `INFO` warning

**Format Dispatcher** (`parsers/dispatcher.py`):
- Takes `detected_format` from Stage 2 and instantiates the correct parser
- Passes parser options from `ClientConfig` (e.g., `excel_sheet_name`, `csv_delimiter_override`)
- Returns `ParseResult` or raises `ParseError` with structured detail

### API Endpoints

The parser library has no HTTP API. It is called directly by the `parse_file` Celery task. It exposes a Python API only:
```python
from parsers.dispatcher import dispatch_parser
result: ParseResult = dispatch_parser(file_bytes, detected_format, client_config)
```

### State Management

Parsers are stateless. `ParseResult` is an immutable dataclass. No database writes occur in any parser. All side effects (DB writes, S3 reads) happen in the Celery task layer that calls the parsers.

### Dependencies

- `chardet` >= 5.0 (encoding detection)
- `openpyxl` >= 3.1 (xlsx parsing)
- `xlrd` >= 2.0 (xls legacy parsing)
- `pdfplumber` >= 0.10 (PDF primary)
- `camelot-py[cv]` >= 0.11 (PDF fallback — includes OpenCV dependency)
- `pyx12` >= 2.0 (EDI X12 parsing)
- `jsonlines` >= 3.1 (JSONL parsing)
- Python stdlib: `csv`, `json`, `io`

---

## Module 3: LLM Schema Inference Engine

### Purpose

The LLM Schema Inference Engine is the core intelligence of the normalisation pipeline. It takes raw column headers and sample data rows from any manifest format and produces a confidence-annotated mapping from source columns to canonical delivery fields. It manages per-client schema learning — storing confirmed mappings and auto-applying them on repeat uploads. It owns the human review workflow for low-confidence or first-time mappings.

### Key Functions

**Schema Inference** (`llm/schema_inference.py`):
- Build inference prompt from `prompts/schema_inference_system.txt` (system) and a dynamically constructed user message containing: column headers as a JSON array, 10 sample rows formatted as a markdown table, and any `ClientSchemaProfile` context (if a partial profile exists)
- Call Claude API: `model=claude-sonnet-4-6`, `max_tokens=2000`, temperature=0 (deterministic output preferred), response forced to JSON conforming to `FieldMappingResult` schema
- Parse response with `FieldMappingResult` Pydantic model; validate all canonical fields listed; reject response if JSON is malformed (retry once)
- Compute overall manifest confidence score: `min(field_confidences)` — the manifest is only as confident as its least-confident field
- Return `SchemaInferenceResult`: list of `FieldMapping` objects, overall confidence, reasoning summary

**Prompt Engineering** (`prompts/schema_inference_system.txt`):
- System prompt includes:
  - Definition of all canonical fields with examples of common source column names for each
  - 5 few-shot examples: Shopify CSV headers → mapping; WooCommerce order export → mapping; MYOB customer delivery report → mapping; SAP delivery note → mapping; generic unstructured CSV → mapping
  - Instructions for handling ambiguous cases: column that could be either `address_line_1` or `suburb` → choose most likely and set confidence <0.8; column with mixed content → flag as `AMBIGUOUS` with confidence 0.5
  - Instructions for columns with no canonical match: set `canonical_field: null`, `confidence: 1.0`, `action: IGNORE`
  - Instructions for multiple source columns mapping to the same canonical field: choose the one that looks most complete; flag the others as `REDUNDANT`

**Client Schema Profile Management** (`llm/schema_profile.py`):
- `get_profile(client_id, header_fingerprint)`: query `client_schema_profiles` table; return profile or None
- `save_profile(client_id, header_fingerprint, field_mappings, upload_id)`: create or update profile
- `increment_success_count(profile_id)`: atomically increment `successful_upload_count`; if count reaches 3 and all field confidences ≥ threshold, set `auto_process_enabled = True`
- `header_fingerprint(headers)`: sort headers, normalise to lowercase, compute SHA-256 — used for cache key
- Profile is updated with new LLM reasoning on each upload even in auto-processing mode, so drift in client formats is detected when header fingerprint changes (cache miss → force new LLM inference)

**Confidence Threshold Logic**:
- Per-field threshold: default 0.95 (configurable per client in `ClientConfig.auto_process_threshold`)
- If any field's confidence < threshold → flag for human review (do not block processing; add to review queue)
- If any field's confidence < 0.5 → set `severity: CRITICAL`; block auto-processing regardless of client config
- Fields with `canonical_field: null` (ignored columns) always confidence 1.0; do not count toward threshold evaluation

**Human Review Queue Integration** (`llm/review_queue.py`):
- On low-confidence inference, set `manifest_uploads.status = AWAITING_SCHEMA_REVIEW`
- Write `FieldMapping` records to DB with `human_confirmed = False`
- Emit WebSocket event to connected ops manager clients: `{ type: 'SCHEMA_REVIEW_REQUIRED', manifest_upload_id, low_confidence_fields: [...] }`
- `approve_mapping(manifest_upload_id, mappings: list[FieldMappingApproval], user_id)`: write confirmations; update `human_confirmed = True`; update `ClientSchemaProfile` with human corrections; transition status to `SCHEMA_CONFIRMED`; resume Celery chain

### API Endpoints

```
GET    /api/v1/manifests/{manifest_upload_id}/schema-mapping
       Response 200: { manifest_upload_id, field_mappings: [...], overall_confidence,
                       requires_review, low_confidence_fields: [...] }

POST   /api/v1/manifests/{manifest_upload_id}/schema-mapping/approve
       Body: { field_mappings: [{ source_column, canonical_field, human_override? }] }
       Response 200: { manifest_upload_id, status: "SCHEMA_CONFIRMED" }

POST   /api/v1/manifests/{manifest_upload_id}/schema-mapping/reject
       Body: { reason }
       Response 200: { manifest_upload_id, status: "FAILED" }

GET    /api/v1/schema-profiles?client_id=
       Response 200: { profiles: [{ id, profile_name, header_fingerprint,
                                    auto_process_enabled, successful_upload_count, last_used_at }] }

DELETE /api/v1/schema-profiles/{profile_id}
       Response 204: deletes profile; next upload from this client will require LLM inference
```

### State Management

The engine reads and writes `FieldMapping` and `ClientSchemaProfile` records. It transitions `ManifestUpload.status` between `SCHEMA_INFERENCE → AWAITING_SCHEMA_REVIEW | SCHEMA_CONFIRMED`. The Celery task chain is paused (using Celery chord or a polling mechanism) while status is `AWAITING_SCHEMA_REVIEW` — the chain resumes only after the `approve_mapping` endpoint is called.

### Dependencies

- `anthropic` Python SDK >= 0.25 (`ANTHROPIC_API_KEY` env var)
- `pydantic` v2 (response model validation)
- `hashlib` stdlib (header fingerprint)
- PostgreSQL (profile storage)
- Redis (WebSocket state for review notifications)
- Celery (chain management)

---

## Module 4: Address Intelligence Service

### Purpose

The Address Intelligence Service validates and enriches every delivery address in a manifest against authoritative Australian address data. It is the pre-routing address quality gate — its job is to ensure that no bad address enters the routing system. It provides correction suggestions, geocoordinates, delivery zone assignment, and FTD history cross-reference.

### Key Functions

**Validation Orchestrator** (`address/validator.py`):
- `validate_batch(records: list[StagingRecord]) -> list[AddressValidationResult]`: main entry point; processes a list of staging records in parallel using `asyncio.gather` with a semaphore limiting to 50 concurrent GNAF API calls
- Per-record flow: normalise input → GNAF exact match → GNAF fuzzy match → Google Maps fallback → HERE fallback → unresolvable
- Emit validation summary: counts by method, coverage percentage, unresolvable addresses list

**Address Normalisation** (`address/normalise.py`):
- Expand common Australian abbreviations before API calls: `St` → `Street`, `Rd` → `Road`, `Ave` → `Avenue`, `Dr` → `Drive`, `Blvd` → `Boulevard`, `Hwy` → `Highway`, etc. (full list in `address/abbreviations.py`)
- Normalise state name variations: `Queensland` → `QLD`, `New South Wales` → `NSW`, etc.
- Strip unit/apartment prefix variations to a canonical form: `Unit 4`, `U4`, `4/` → `Unit 4`
- Strip excess whitespace, punctuation artefacts
- Concatenate address components into a single search string for API calls

**GNAF Integration** (`address/gnaf_client.py`):
- Client wraps PSMA Data API (or self-hosted GNAF PostgreSQL mirror — see note below)
- `exact_match(address_string, postcode, state)`: returns GNAF record or None
- `fuzzy_search(address_string, suburb, state, limit=5)`: returns list of candidates with match scores
- GNAF record enrichment: extract `gnaf_pid`, `latitude`, `longitude`, `mesh_block`, `lga_name`, `locality_name`, `state_abbreviation`, `postcode`
- Note on GNAF deployment: for production, self-hosted GNAF PostgreSQL instance is strongly recommended over the PSMA API to avoid per-call costs and latency. GNAF data is downloadable quarterly from data.gov.au. A self-hosted instance on RDS PostgreSQL with PostGIS resolves address queries in <10ms. This is the recommended architecture for >1,000 validations/day.

**Fuzzy Correction Suggestions** (`address/fuzzy.py`):
- When GNAF exact match fails, use `RapidFuzz` to compute similarity between input address tokens and GNAF candidate tokens
- `suggest_corrections(input_address, gnaf_candidates)`: for each candidate, compute token sort ratio; return candidates sorted by score descending
- Threshold: suggest corrections only for candidates with score ≥ 0.85
- Return top 3 suggestions with diff highlighting (which tokens changed)

**Google Maps Fallback** (`address/google_maps_client.py`):
- Called when GNAF match confidence < 0.7 or GNAF returns no results
- Use `Geocoding API` (not Places): `https://maps.googleapis.com/maps/api/geocode/json?address={address}&components=country:AU&key={key}`
- Extract `geometry.location` (lat/lng), `address_components` (parse street_number, route, locality, administrative_area_level_1, postal_code)
- Mark result as `validation_method = GOOGLE_MAPS`; confidence computed from `partial_match` flag: 0.75 if no partial match, 0.6 if partial_match=true
- Apply rate limiting: 50 QPS max (Google Maps free tier = 40,000 calls/month; monitor via Datadog custom metric `google_maps.geocoding_calls`)

**HERE Fallback** (`address/here_client.py`):
- Called when Google Maps returns an error or when `GOOGLE_MAPS_QUOTA_EXCEEDED` flag is set in Redis
- Uses HERE Geocoding & Search API v7: `https://geocode.search.hereapi.com/v1/geocode?q={address}&in=countryCode:AUS&apiKey={key}`
- Same response mapping as Google Maps client; mark as `validation_method = HERE`

**FTD History Lookup** (`address/ftd_history.py`):
- `get_ftd_rate(gnaf_pid: str | None, address_line_1: str, postcode: str) -> float`: looks up historical FTD rate for this address
- Primary lookup by `gnaf_pid` (exact match against `delivery_records` history)
- Fallback lookup by `(address_line_1_normalised, postcode)` if no `gnaf_pid`
- FTD rate = `COUNT(deliveries where outcome='FAILED') / COUNT(all deliveries)` for this address over last 12 months
- Threshold: `ftd_rate >= 0.3` (30% historical failure rate) → flag as `HIGH_FTD_RATE` anomaly with severity `WARNING`

**Delivery Zone Assignment** (`address/zones.py`):
- Assign `delivery_zone` from a lookup table mapping postcode ranges to operator-defined zones (e.g., "Brisbane CBD", "Logan", "Redlands", "Moreton Bay")
- Zone table stored in `delivery_zones` table; managed via admin UI
- Used downstream by routing for zone-based vehicle assignment

### API Endpoints

The Address Intelligence Service is primarily internal (called by the Celery task pipeline). It also exposes:

```
POST   /api/v1/address/validate
       Body: { address_line_1, address_line_2?, suburb, state, postcode, country? }
       Response 200: { validation_method, validated_address, confidence, lat, lng,
                       correction_applied, suggestions: [...], is_deliverable }
       Use case: on-demand validation from the Human Review Interface for manual address editing

GET    /api/v1/address/zones
       Response 200: { zones: [{ zone_id, zone_name, postcodes: [...] }] }

POST   /api/v1/address/zones
       Body: { zone_name, postcodes: [...] }
       Response 201: { zone_id }
```

### State Management

Writes `AddressValidationResult` records to PostgreSQL. Reads `delivery_records` for FTD history lookup. Does not modify `ManifestStagingRecord` directly — the validation result is linked via FK. The Celery task layer reads validation results and updates staging record fields after validation completes.

### Dependencies

- `httpx` (async HTTP for GNAF/Google Maps/HERE API calls)
- `rapidfuzz` >= 3.0 (fuzzy matching)
- `phonenumbers` (phone number normalisation — used here to validate Australian mobile numbers)
- PostgreSQL (GNAF mirror, FTD history, zone tables, validation result writes)
- `GNAF_API_URL`, `GOOGLE_MAPS_API_KEY`, `HERE_API_KEY` env vars
- Redis (quota exceeded flag for Google Maps)

---

## Module 5: Anomaly Detection Pipeline

### Purpose

The Anomaly Detection Pipeline is the data quality gate that runs after transformation and address validation. It applies two complementary detection approaches: a deterministic rule engine that checks for known failure patterns (missing fields, duplicates, format violations) and a statistical ML model (Isolation Forest) that identifies records that are unusual given the operator's historical data. Every detected anomaly receives a severity classification and a human-readable description enabling the ops manager to act quickly.

### Key Functions

**Pipeline Orchestrator** (`anomaly/pipeline.py`):
- `run_pipeline(manifest_upload_id: UUID) -> AnomalyReport`: main entry point
- Loads all staging records for the manifest
- Runs rule-based checks first (fast, synchronous)
- Runs Isolation Forest scoring second
- Runs FTD history cross-reference third
- Deduplicates anomaly flags (a record may be flagged by multiple rules; consolidate to one `AnomalyFlag` per `(record_id, anomaly_type)` pair)
- Writes `AnomalyFlag` records to DB
- Updates `manifest_uploads` with anomaly counts by severity
- Returns `AnomalyReport` dataclass

**Rule Engine** (`anomaly/rules/`):
Each rule is a Python class inheriting from `BaseRule`:
```python
class BaseRule:
    severity: AnomalySeverity
    anomaly_type: AnomalyType
    def check(self, record: StagingRecord, context: RuleContext) -> list[AnomalyFlag]: ...
```

Rules implemented in `anomaly/rules/`:
- `MissingAddressRule`: flags records where `address_line_1` is blank or None — `CRITICAL`
- `MissingPostcodeRule`: flags records where `postcode` is blank — `CRITICAL`
- `MissingPhoneRule`: flags records where both `phone_mobile` and `phone_landline` are None — `WARNING`
- `InvalidPostcodeQldRule`: flags records where postcode is not in range 4000–4999 (for QLD deliveries) — `WARNING`
- `InvalidPhoneFormatRule`: flags records where phone is present but fails E.164/Australian format validation — `INFO`
- `ExactDuplicateParcelIdRule`: flags records where `parcel_id` matches another record in the same manifest OR in `delivery_records` for the same date — `CRITICAL`
- `NearDuplicateAddressRule`: flags pairs of records with identical `(address_line_1, postcode)` but different `parcel_id` — `WARNING`; includes link to related record
- `SuspiciousVolumeRule`: flags any single address with >5 deliveries in the manifest — `WARNING`; threshold configurable in `ClientConfig`
- `ZeroWeightRule`: flags records where `weight_kg` is present but equals 0 — `INFO`
- `UndeliverableAddressRule`: flags records where `AddressValidationResult.is_deliverable = False` — `CRITICAL`
- `HighFtdRateRule`: flags records where `AddressValidationResult.ftd_rate_historical >= 0.3` — `WARNING`

`RuleContext` includes: the full manifest's staging records (for cross-record rules like duplicates), the `client_id`, the manifest date.

**Isolation Forest Model** (`anomaly/ml/isolation_forest.py`):
- Model trained on `delivery_records` table: 6 months of historical data minimum
- Feature vector per record:
  - `weight_kg` (normalised)
  - `delivery_zone` (one-hot encoded)
  - `service_type` (one-hot encoded)
  - `hour_of_day_of_upload` (integer 0–23)
  - `client_id` (one-hot encoded)
  - `postcode_prefix` (first 2 digits, integer — proxy for delivery area)
  - `has_special_instructions` (binary)
  - `requires_signature` (binary)
- `train_model(client_id: UUID, min_records: int = 500)`: trains a per-client model if sufficient history; falls back to operator-wide model if insufficient; saves with `joblib.dump()` to `models/isolation_forest_{client_id}.joblib`
- `score_records(records: list[StagingRecord], client_id: UUID) -> list[float]`: loads model, computes anomaly scores (0.0–1.0; score ≥ 0.7 → flag as `STATISTICAL_OUTLIER` with severity `INFO`; score ≥ 0.9 → severity `WARNING`)
- Model retraining: Celery beat task `retrain_anomaly_models` runs weekly; retrains all client models with updated delivery history
- Cold start: if no model exists for a client, skip ML scoring and log `INFO`; run rule engine only

**Anomaly Report Generation** (`anomaly/report.py`):
- `generate_report(manifest_upload_id) -> AnomalyReport`: aggregates all `AnomalyFlag` records for the manifest
- Groups anomalies by severity and type
- Generates a plain-English summary: "3 critical anomalies require resolution before dispatch. 7 warnings should be reviewed. 2 informational items noted."
- Produces `suggested_actions`: ordered list of recommended ops manager actions
- Sends dispatcher notification: Slack webhook (configurable) + in-app notification

**Dispatcher Notification** (`anomaly/notifications.py`):
- On any `CRITICAL` anomaly: immediate alert via configured channels (Slack `#dispatch-alerts`, in-app notification, optional SMS via Twilio)
- On `WARNING` count > 10: summary notification (do not spam per-anomaly)
- On `INFO` only: no immediate notification; included in daily summary report

### API Endpoints

```
GET    /api/v1/manifests/{manifest_upload_id}/anomalies
       Query params: severity (filter), status (filter: UNRESOLVED|ACCEPTED|CORRECTED|REJECTED)
       Response 200: { anomaly_report: { summary, counts_by_severity, counts_by_type },
                       anomalies: [{ id, anomaly_type, severity, description,
                                     suggested_action, record_id, related_record_ids, score }] }

POST   /api/v1/anomalies/{anomaly_id}/resolve
       Body: { resolution: "ACCEPTED"|"CORRECTED"|"REJECTED", note?, corrected_record? }
       Response 200: { anomaly_id, resolution_status, resolved_at }

POST   /api/v1/manifests/{manifest_upload_id}/anomalies/bulk-resolve
       Body: { resolutions: [{ anomaly_id, resolution, note? }] }
       Response 200: { resolved_count, remaining_critical_count }

POST   /api/v1/anomaly-models/retrain
       Body: { client_id? }  -- null = retrain all
       Response 202: { task_id }
       Auth: admin only
```

### State Management

Writes `AnomalyFlag` records. Reads `ManifestStagingRecord` and `AddressValidationResult`. Updates `manifest_uploads.anomaly_count_*`. Manages model files on local filesystem (ECS ephemeral storage) with S3 backup (`s3://manifest-models/isolation_forest_{client_id}.joblib`). On container start, model files are downloaded from S3 if not present locally.

### Dependencies

- `scikit-learn` >= 1.4 (`IsolationForest`)
- `pandas` >= 2.0 (feature matrix construction)
- `joblib` (model serialisation)
- `numpy` >= 1.26
- `boto3` (model S3 sync)
- PostgreSQL (anomaly flag writes, history reads)
- Redis (Celery task coordination)
- Twilio SDK (optional SMS notifications — `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` env vars)
- Slack webhook URL (`SLACK_ANOMALY_WEBHOOK_URL` env var)

---

## Module 6: Late Manifest Merge Engine

### Purpose

The Late Manifest Merge Engine handles the case where new delivery stops arrive after route planning has already completed. Rather than requiring the ops manager to manually re-plan, it automatically integrates new stops into existing routes by identifying the nearest suitable route, triggering partial re-optimisation only for affected segments, updating driver apps, and providing the dispatcher with an impact summary for one-click approval.

### Key Functions

**Late Detection** (`merge/detector.py`):
- A manifest is classified as "late" if `manifest_uploads.created_at` is after the `late_manifest_cutoff_time` configured in `ClientConfig` (default 07:00 local time on the delivery date) AND at least one route for that delivery date has status `OPTIMISED` or `IN_PROGRESS`
- Late classification is determined during the `finalise_records` stage; sets `manifest_uploads.is_late_manifest = True`
- Triggers `merge_late_manifest` Celery task instead of normal `dispatch_to_routing`

**Affected Route Identification** (`merge/route_matcher.py`):
- For each new `DeliveryRecord`, determine which existing route it should be assigned to
- Algorithm: compute distance from new stop's geocoordinates to the centroid of each existing route's undelivered stops; assign to the route with the minimum centroid distance, subject to constraints:
  - Route must not have a `status = COMPLETED`
  - Route's estimated end time (after adding the new stop) must not exceed `driver.max_shift_end_time` by more than 30 minutes (configurable)
  - Route's vehicle capacity (weight and parcel count) must accommodate the new parcel
- If no suitable route found for a new stop → flag as `UNASSIGNABLE_LATE_STOP`; alert dispatcher for manual assignment
- Returns: `{ new_stop_id → route_id }` mapping, list of affected route IDs

**Partial Re-optimisation** (`merge/reoptimiser.py`):
- For each affected route, extract the sub-sequence of stops that have not yet been attempted (driver's current position and all future stops)
- Call routing engine (internal `RoutingService` or Google OR-Tools directly) with the existing future stops + new stops; optimise stop order only for this sub-sequence
- Lock already-completed stops (cannot be re-ordered)
- Apply time window constraints from delivery records
- Return updated stop sequence for each affected route
- Target: re-optimisation per affected route in <5 seconds (using OR-Tools local search heuristics, not full re-solve)

**Driver App Push** (`merge/driver_notifier.py`):
- For each affected route, call `DriverAppService.update_route(route_id, updated_stops)`
- Driver app update includes: new stop details (address, parcel info, special instructions), updated stop sequence, revised ETA for remaining stops
- Push notification to driver's device: "Your route has been updated. 2 new stops added. Revised ETA: 4:45 PM."
- Uses Firebase Cloud Messaging (FCM) for push notifications (`FCM_SERVER_KEY` env var)
- Logs notification delivery status; alerts if driver app has not acknowledged update within 2 minutes

**Impact Summary** (`merge/impact_calculator.py`):
- Computes per-route impact: stops added, estimated time added (minutes), revised completion ETA, capacity utilisation after addition
- Flags routes where addition causes >30-minute time extension
- Flags stops that could not be assigned to any route
- Generates `LateMergeReport` for dispatcher review

**Dispatcher Approval Workflow**:
- If `ClientConfig.auto_approve_late_merge = True` and no `UNASSIGNABLE_LATE_STOP` anomalies → auto-approve; push to drivers immediately
- Otherwise → status `AWAITING_LATE_MERGE_APPROVAL`; send Slack notification with summary; dispatcher approves via React UI one-click action

### API Endpoints

```
GET    /api/v1/manifests/{manifest_upload_id}/merge-report
       Response 200: { is_late_manifest, affected_routes: [{ route_id, driver_name,
                         stops_added, time_added_minutes, revised_eta, capacity_pct }],
                       unassignable_stops: [...], total_new_stops, auto_approve_eligible }

POST   /api/v1/manifests/{manifest_upload_id}/merge-report/approve
       Response 200: { status: "MERGE_APPROVED", routes_updated: [...], drivers_notified: [...] }

POST   /api/v1/manifests/{manifest_upload_id}/merge-report/reject
       Body: { reason }
       Response 200: { status: "MERGE_REJECTED" }

GET    /api/v1/routes/{route_id}/merge-history?date=
       Response 200: { merges: [{ manifest_upload_id, stops_added, merged_at, approved_by }] }
```

### State Management

Reads `delivery_records`, `routes`, `route_stops`, `drivers`, `vehicles`. Writes updated `route_stops` records. Updates `delivery_records.route_id`, `stop_sequence`. Triggers `DriverAppService`. Writes `LateMergeEvent` records to `late_merge_events` table for audit trail.

### Dependencies

- Google OR-Tools (`ortools` Python package >= 9.8) for partial re-optimisation
- Firebase Admin SDK (`firebase-admin` Python SDK) for FCM push notifications
- PostgreSQL (route, delivery record reads/writes)
- Redis (Celery)
- Internal `RoutingService` API (if routing is a separate service within the platform)
- `FCM_SERVER_KEY`, `ROUTING_SERVICE_URL` env vars

---

## Module 7: Human Review Interface

### Purpose

The Human Review Interface is the React front-end for the three situations that require ops manager attention: (1) low-confidence schema mappings produced by the LLM inference engine, (2) anomaly flags from the detection pipeline, and (3) late manifest merge approval. It presents information at the right level of detail for a non-technical ops manager: confidence scores are shown as colour-coded indicators, not raw floats; anomaly descriptions are plain English; one-click actions are the primary interaction pattern. Every human decision is recorded in a full audit trail.

### Key Functions

**Schema Mapping Review View** (`src/features/schema-review/`):
- Displays a table of source columns (from the uploaded file) alongside their LLM-inferred canonical field mappings
- Per-field confidence indicator: green (≥0.95), yellow (0.80–0.94), red (<0.80)
- LLM reasoning shown in a tooltip on hover (from `FieldMapping.llm_reasoning`)
- Ops manager can change the canonical field assignment via a dropdown of all canonical field options
- Ops manager can mark a column as "ignore" (not mapped to any canonical field)
- Sample data preview: clicking any row in the mapping table shows the first 5 values from that column to help the ops manager verify the mapping
- "Approve All" button applies when all confidence indicators are green; otherwise "Review & Approve" button is shown
- On approval, calls `POST /api/v1/manifests/{id}/schema-mapping/approve`
- State: managed by React Query; optimistic updates on approval actions

**Anomaly Review View** (`src/features/anomaly-review/`):
- Summary panel: total records, clean records, CRITICAL count (red badge), WARNING count (amber badge), INFO count (grey badge)
- Anomaly list sorted by severity (CRITICAL first), then by anomaly score descending
- Per-anomaly card: shows affected record(s), anomaly type (human-readable label), description, suggested action
- Inline editing: for anomalies where the resolution is "correct the data" (e.g., wrong postcode), ops manager can edit the field value directly in the card; edit is validated on blur (re-runs address validation for address edits)
- Bulk actions: "Accept all WARNINGs" button for ops managers who want to proceed without correcting warnings; "Accept all INFOs" one-click
- CRITICAL anomalies cannot be bulk-accepted; each must be individually resolved
- Duplicate anomalies show both records side-by-side for comparison
- Near-duplicate anomalies show address diff highlighting (which tokens differ between the two records)
- On all anomalies resolved → "Approve & Proceed to Routing" button becomes active

**Late Manifest Merge Approval View** (`src/features/late-merge/`):
- Summary: "X new parcels arrived at [time]. Below is the proposed merge into existing routes."
- Route impact table: per-affected route — driver name, current stop count, stops added, time added, revised ETA, vehicle capacity bar
- Visual map (Google Maps embed or Leaflet) showing existing route stops and new stops highlighted in a distinct colour
- Unassignable stops highlighted in red with explanation
- "Approve Merge" button → calls approve endpoint; shows confirmation: "Drivers notified. Updates sent to [N] drivers."
- "Reject and Handle Manually" button → routes to manual assignment UI

**Real-time Progress View** (`src/features/manifest-progress/`):
- Shows pipeline stage progress for in-flight manifests
- Stage timeline: each stage shown as a step with status (pending, in-progress, complete, error)
- WebSocket connection to `wss://api.odocs.app/ws/manifests/{id}`; updates stage state in real time
- Error state: if a stage fails, shows the error message and available recovery actions (retry, manual override, contact support)

**Manifest History Dashboard** (`src/features/dashboard/`):
- Table of recent manifests with status, record count, anomaly summary, processing time
- Filter by client, date range, status
- Quick actions: re-process failed manifests, download clean export, view anomaly report
- Daily summary stats: manifests processed today, auto-processed percentage, total records, anomaly rate

### Field-Level Confidence Indicators

Implemented as a reusable component `<ConfidenceBadge confidence={0.87} />`:
- 0.95–1.0: green filled dot + "Verified" label
- 0.80–0.94: yellow dot + "Review" label (draws attention without blocking)
- 0.50–0.79: amber dot + "Low confidence" label
- <0.50: red dot + "Manual required" label

The design intentionally avoids showing raw float values to ops managers. The colour + label system is sufficient for action decisions.

### Audit Trail

Every human action in the review interface writes an audit record:
```
audit_events
├── id: UUID (PK)
├── event_type: VARCHAR(100)  -- e.g. 'SCHEMA_MAPPING_APPROVED', 'ANOMALY_ACCEPTED', 'FIELD_EDITED'
├── manifest_upload_id: UUID
├── entity_type: VARCHAR(50)  -- 'field_mapping' | 'anomaly_flag' | 'staging_record' | 'late_merge'
├── entity_id: UUID
├── user_id: UUID (FK → users.id)
├── before_value: JSONB (NULLABLE)
├── after_value: JSONB (NULLABLE)
├── ip_address: INET
└── created_at: TIMESTAMPTZ
```

Audit records are append-only (no UPDATE or DELETE). They are the authoritative record of what a human changed and when.

### API Endpoints

The Human Review Interface consumes all API endpoints defined in Modules 3, 5, and 6. Additional endpoints specific to the review UI:

```
GET    /api/v1/manifests/{manifest_upload_id}/review-summary
       Response 200: { needs_schema_review: bool, needs_anomaly_review: bool,
                       needs_merge_approval: bool, schema_mapping_summary: {...},
                       anomaly_summary: {...}, merge_summary: {...} }
       Use case: single API call to determine what review actions are pending for a manifest

GET    /api/v1/dashboard/today
       Response 200: { manifests_today: int, auto_processed: int, awaiting_review: int,
                       total_records: int, anomaly_rate: float, avg_processing_seconds: float }

GET    /api/v1/audit?manifest_upload_id=&user_id=&event_type=&from=&to=&limit=&offset=
       Response 200: { events: [...], total }
       Auth: admin role required
```

### State Management

React Query is the primary state manager for server state. Zustand stores: `useReviewStore` (current manifest under review, active review step), `useNotificationStore` (pending review alerts from WebSocket). WebSocket connection managed by a custom React hook `useManifestProgress(manifestUploadId)` which subscribes on mount and unsubscribes on unmount.

### Dependencies

**Frontend**:
- React 18 + TypeScript 5
- Zustand (client state)
- TanStack Query v5 (server state)
- TanStack Table v8 (virtual scrolling for large manifests)
- shadcn/ui + Tailwind CSS
- react-dropzone (file upload)
- Leaflet + react-leaflet (map view for late merge)
- `diff` npm package (for address diff highlighting in near-duplicate view)

**Backend consumed by this module**:
- All endpoints from Modules 3, 5, 6
- WebSocket from Module 1 (progress streaming)

**Environment variables (frontend)**:
- `VITE_API_BASE_URL`
- `VITE_WS_BASE_URL`
- `VITE_GOOGLE_MAPS_KEY` (for Leaflet/Maps embed in merge view)
