# Challenge 2: Inbound Data Normalisation & Manifest Automation — Product Strategy

---

## The Pain

### The Daily Ritual That Shouldn't Exist

Every morning, before the first truck leaves the depot, the operations manager at a 35-vehicle Brisbane delivery business sits down to a task that should have been automated years ago: manually reconciling 4 incoming data feeds from 4 different clients into one coherent dispatch manifest.

The process looks like this:

- **Client A (e-commerce retailer)** sends a CSV export from Shopify. Columns are labelled `ship_to_name`, `ship_to_address1`, `ship_to_city`, `ship_to_zip`. Every column label is different from the operator's internal schema. The ops manager opens Excel, manually maps the columns, deletes irrelevant columns, reformats postcodes (Shopify exports them as numeric, losing leading zeros for QLD postcodes starting with 4), and pastes into the master dispatch sheet.
- **Client B (FMCG brand)** sends an EDI 204 feed from their enterprise WMS. The ops manager does not have EDI parsing software. They use a legacy conversion tool that produces garbled output 2–3 times per week, requiring manual correction. On bad days this alone takes 25 minutes.
- **Client C (healthcare consumables)** uploads manually to a web portal. The portal exports a non-standard Excel file with merged header rows, subtotals mixed into data rows, and date fields formatted as `DD/MM/YYYY` strings rather than ISO dates. The ops manager strips the subtotals, unmerges headers, and reformats dates by hand.
- **Client D (occasional volume e-commerce)** sends ad hoc CSV files with no fixed schema. Column names and ordering vary between uploads. The ops manager reads the file, guesses the column intent from content, and maps manually each time.

**Total daily time cost: 60–90 minutes**, every single working day, before any operational work has begun. At 250 working days per year, this represents **250–375 hours per year** of senior operational staff time consumed by data entry and format translation — tasks with zero strategic value.

### The Cascade of Errors

Manual data transformation is error-prone by nature. The failure modes are not theoretical; they are daily occurrences:

**Address errors** are the most costly. A single transposition — `St` vs `Street`, `Rd` vs `Road`, an incorrect postcode — typically goes undetected at ingestion time. The error propagates silently into route planning. The driver is dispatched to a geocoded approximation of a bad address. One of three outcomes follows: (1) the driver finds the correct address by local knowledge and delivers, wasting 5–15 minutes; (2) the driver cannot locate the address and marks a failed first attempt (FTD), triggering a re-delivery cycle at ~$8–12 direct cost plus the SLA penalty for the client; (3) the address is completely undeliverable and the parcel returns to depot, generating an invoice dispute when the client argues the address was correct in their system but wrong in the delivery manifest.

**Missing records** create a different class of problem. A parcel skipped in the copy-paste operation means a driver completes their run without it, it sits in the depot, the client's customer doesn't receive their order, and the client raises an enquiry 24–48 hours later. The ops manager must trace the failure back through the manifest. If the record was missing from ingestion, it may not appear in any system log, making the investigation manual. Invoice disputes arising from missing records typically extend DSO (Days Sales Outstanding) by 7–14 days while the dispute is resolved.

**Duplicate records** send a driver to the same address twice, wasting fuel, time, and driver hours. On a 3,000-parcel day across 35 vehicles, even a 0.1% duplicate rate means 3 phantom stops consuming roughly 45 minutes of combined driver time.

**Quantified annual impact (estimated for this operator):**
- Ops manager time on manual normalisation: 312 hours/year (assuming 75-minute average, 250 days)
- Address-error-driven FTDs attributable to bad ingestion data: estimated 2–4 per day = 500–1,000 redeliveries/year at $8–12 each = **$4,000–$12,000 direct cost**
- SLA penalties from late/failed deliveries with address root cause: estimated $15,000–$25,000/year
- Invoice disputes from missing records: 3–5 per month, $200–$500 average resolution cost = **$7,200–$30,000/year in dispute overhead and DSO drag**
- **Total addressable pain: $26,200–$67,000/year plus 312 hours of senior staff time**

### The Single Point of Failure

The entire morning ingestion operation is held in the ops manager's head. There is no written SOP that another staff member can execute. The column mapping knowledge, the quirks of each client's format, the workarounds for the EDI converter — it is all tacit knowledge.

When the ops manager is sick, on leave, or delayed, the options are: (a) wait for them, delaying the entire dispatch schedule; (b) have another staff member attempt the process with a high error rate; or (c) call the ops manager at home. This is not a hypothetical: in a business of this size, the ops manager takes approximately 10–15 sick/personal days per year. Each one creates a dispatch risk.

### The Late Manifest Problem

Clients do not stop placing orders at midnight. Last-minute e-commerce orders, healthcare emergency stock movements, and same-day additions arrive until 7:00am or later. Routes are pre-planned the night before or early morning using the manifest available at that time. Late additions are either: (a) added manually to driver phones/tablets by calling or texting drivers individually; (b) shoehorned into the nearest route without optimisation; or (c) held over to the next day, violating the client's SLA.

The manual re-routing required to integrate even 10 late parcels into 35 pre-planned routes takes 20–30 minutes and produces sub-optimal results. On days with 50+ late additions (common before public holidays or during promotional events), the process breaks down entirely.

---

## The Status Quo: How Competitors Handle This (and Fail)

Every competing product in this category treats manifest ingestion as a solved problem. It is not. Their approach: define a fixed template, tell the client to format their data to match it, and call the import feature "flexible" because they added a drag-and-drop column mapping UI. This approach places the normalisation burden squarely on the client and the operator, and provides no intelligence at any layer of the pipeline.

### Locate2u
- **What they offer**: Native Shopify integration (only useful for Shopify-sourced manifests); basic CSV import using a fixed template with prescribed column names.
- **The gap**: Column names must match Locate2u's template exactly or import fails. No fuzzy matching, no LLM inference, no learning from client formats. The Shopify integration handles one client type only. EDI: unsupported. PDF: unsupported. Address validation: basic geocoding only, no GNAF cross-reference. Anomaly detection: none.
- **Operator impact**: Every non-Shopify client requires manual format conversion before upload. The ops manager's workload is unreduced.

### Onfleet
- **What they offer**: Batch CSV import with a column mapping UI (drag column names to field slots); REST API for programmatic upload accepting JSON; basic geocoding on upload.
- **The gap**: The column mapping UI requires a human to map columns every time a new file format is encountered, or every time a client changes their export. The API requires the sending system to conform to Onfleet's JSON schema — not useful for ad hoc client uploads. No LLM-based schema inference means no ability to handle arbitrary or evolving formats. No EDI 204 support. No PDF parsing. Anomaly flagging is limited to geocoding failures; no ML-based duplicate or pattern detection. No GNAF integration (critical for Australian address accuracy).
- **Operator impact**: The column mapping UI reduces one type of manual work (copy-paste reformatting) but still requires human attention for every non-standard or first-time format. The core problem — 4 different client formats, all requiring human interpretation — is not solved.

### OptimoRoute
- **What they offer**: CSV import with geocoding and confidence scores; manual column mapping interface; ability to reject imports below a geocoding confidence threshold.
- **The gap**: The geocoding confidence threshold is a useful concept, but it is applied only to geocoding, not to schema mapping or field-level data quality. Column mapping is manual. No EDI, no PDF, no ML anomaly detection. No GNAF integration. No per-client schema learning. The geocoding confidence threshold creates a binary reject/accept decision with no suggested corrections.
- **Operator impact**: Marginally better than raw CSV import for address quality, but the normalisation workflow remains entirely manual.

### Shippit
- **What they offer**: 100+ carrier integrations; retailer-facing platform for booking and managing shipments across multiple carriers.
- **The gap**: Shippit is a retailer/shipper tool, not an operator tool. It helps a retailer route parcels to the right carrier, not help a carrier/operator process inbound manifests from multiple client formats. This product category is not relevant to the operator's use case. Included here for completeness; it does not compete on this dimension.

### Routific
- **What they offer**: Drag-and-drop CSV import; visual column mapping interface; route optimisation focus.
- **The gap**: The drag-and-drop interface is the same category of manual column mapping as Onfleet and OptimoRoute, just with a more polished UI. No intelligence behind the mapping. No schema learning. No EDI, no PDF, no anomaly detection, no GNAF. Routific's investment is in route optimisation, not manifest intelligence. The ingestion layer is a minimum viable feature.
- **Operator impact**: Same manual column mapping burden. The visual UI slightly reduces friction but does not change the underlying workflow.

### The Universal Gap

After surveying all viable competitors, the gap is consistent and complete:

| Capability | Locate2u | Onfleet | OptimoRoute | Routific | This Product |
|---|---|---|---|---|---|
| Arbitrary CSV/Excel (no fixed template) | No | No (manual mapping) | No (manual mapping) | No (manual mapping) | Yes (LLM inference) |
| EDI X12 204 support | No | No | No | No | Yes |
| PDF manifest extraction | No | No | No | No | Yes |
| GNAF address validation | No | No | No | No | Yes |
| Per-client schema learning | No | No | No | No | Yes |
| ML anomaly detection | No | No | No | No | Yes |
| Late manifest auto-merge | No | No | No | No | Yes |

No competitor uses LLMs to infer schema from arbitrary files. No competitor supports EDI 204. No competitor parses PDFs. No competitor uses GNAF (Australia's most authoritative address database) for postal validation. No competitor has ML-based anomaly detection. No competitor has a late manifest merge engine. Every capability in this product's differentiation stack is a white space in the competitive landscape.

---

## The Disruption: 10x Better

### Innovation 1: LLM-Powered Schema Inference

**What it does:**

The operator uploads any file — a CSV from Shopify, an Excel export from a custom WMS, a PDF manifest printed from a legacy system. The system uses the Claude API (claude-sonnet-4-6) to read the file's headers, sample rows, and content patterns, then infers the semantic meaning of each column and maps it to the canonical delivery schema: `recipient_name`, `address_line_1`, `address_line_2`, `suburb`, `state`, `postcode`, `phone`, `parcel_id`, `weight_kg`, `special_instructions`, etc.

The mapping is returned as a JSON structure with a confidence score per field (0.0–1.0). High-confidence mappings (>0.95) are shown to the ops manager as pre-confirmed. Low-confidence mappings are flagged for one-click review. After the ops manager confirms or corrects a mapping, that mapping is stored as a per-client schema profile. After 3 successful uploads from the same client, the system auto-processes without human review (provided all fields score above the confidence threshold).

The LLM prompt includes few-shot examples of common delivery manifest formats (Shopify, WooCommerce, MYOB, SAP delivery notes) and is instructed to return structured JSON conforming to the field mapping schema. The prompt also instructs the model to reason about ambiguous columns — e.g., a column labelled `Phone` that contains a mix of mobile and landline numbers is identified as `phone` with a note that landline numbers may not receive SMS notifications.

**Why it's 10x better:**

Every competitor requires the client to conform to a fixed template or the operator to perform manual column mapping. The LLM approach inverts this: the system adapts to the client, not the other way around. The first upload from a new client format takes approximately 2 minutes of human review time (reviewing and confirming the LLM's mapping). All subsequent uploads from that client are automatic. By week 4 of operation, all established clients are processing without human review.

**Efficiency metrics:**
- First-time format: manual time reduced from 15–20 minutes to 2–3 minutes (review and confirm LLM mapping)
- Known client format (learned): manual time reduced from 15–20 minutes to 0 minutes (fully automated)
- Error rate from column misidentification: target <0.5% (vs. ~3–5% manual error rate)
- After 8-week learning period: 90%+ of daily manifests processed with zero human intervention

### Innovation 2: Australian Address Intelligence

**What it does:**

Every address in every manifest is validated against GNAF (Geocoded National Address File) — the Australian federal government's authoritative national address database, maintained by PSMA Australia and updated quarterly. The GNAF contains approximately 15 million address records across all of Australia, including all Queensland addresses.

Validation proceeds in layers:
1. **Exact GNAF match**: Address string matched exactly to a GNAF record. Confidence: 1.0. Geocoordinates, state, postcode, suburb, and LGA enriched from GNAF.
2. **Fuzzy GNAF match**: Address string matches a GNAF record after normalisation (abbreviation expansion, punctuation removal, case normalisation). Confidence: 0.85–0.99 depending on edit distance. Corrections suggested to the ops manager.
3. **Near match with suggested correction**: Address has a recognisable street name and suburb but an incorrect street number or postcode. GNAF returns the nearest valid address and flags it for human confirmation.
4. **GNAF miss, Google Maps fallback**: Address not found in GNAF (may be new development, rural delivery point, or PO Box). Geocoded via Google Maps API. Result flagged as unvalidated against GNAF with a warning.
5. **Undeliverable**: Address cannot be geocoded by any method, or geocodes to a location with historical delivery failure rate >40% (from the FTD history database). Flagged as high-risk before routing.

**Why it's 10x better:**

Competitors use Google Maps geocoding as their only address validation. Google Maps geocodes most addresses successfully but has no knowledge of which Australian addresses are legitimate registered delivery points vs. approximate geocodes for a street. GNAF validation catches addresses that Google Maps would geocode but are actually invalid (e.g., `150 Smith St, Brisbane` when the valid range is 1–148). It also catches postcode/suburb mismatches that Google Maps silently corrects — masking data quality issues from the client.

**Efficiency metrics:**
- Address errors caught before routing: target 95%+ of bad addresses identified at ingestion
- Reduction in address-error-driven FTDs: target 60–70% reduction vs. baseline
- Geocoding enrichment coverage: 99%+ of metro Brisbane addresses resolved with coordinates
- Correction suggestion accuracy (fuzzy match): target 85%+ of suggested corrections are right

### Innovation 3: Intelligent Anomaly Detection

**What it does:**

A two-layer anomaly detection pipeline runs on every processed manifest before it reaches the routing system:

**Layer 1 — Rule-based checks (deterministic, fast):**
- Exact duplicate parcel IDs within the manifest or against today's existing records
- Near-duplicate detection: same delivery address, different parcel ID (suggests a re-submission or error)
- Missing required fields: `phone` absent (prevents SMS notification), `postcode` absent, `address_line_1` blank
- Format anomalies: invalid Australian phone number format, postcode not in the 4000–4999 range (for QLD validation), date fields containing non-date values
- Suspicious patterns: more than 5 parcels to a single residential address (flags potential consolidation error or address reuse); zero-weight parcel (likely data error)

**Layer 2 — ML-based detection (Isolation Forest):**
The Isolation Forest model is trained on historical manifest data. It learns the statistical distribution of "normal" delivery records for this operator: typical address patterns, parcel weight distributions, volume per client per day, geographic spread of deliveries. Records that are statistically anomalous — outside the learned distribution — are flagged with an anomaly score (0.0–1.0, where 1.0 is most anomalous).

Each anomaly from either layer receives a severity classification: `CRITICAL` (must be resolved before routing — e.g., missing address), `WARNING` (should be reviewed — e.g., near-duplicate, suspicious volume), or `INFO` (logged for awareness — e.g., unusual weight).

The ops manager sees an anomaly report before final dispatch approval: total records, clean records, flagged records by severity, and a one-click resolution interface for each anomaly.

**Why it's 10x better:**

No competitor has any ML-based anomaly detection. Rule-based checks exist in some products (e.g., geocoding failure flagging in OptimoRoute) but are narrow in scope. The Isolation Forest approach surfaces anomalies that no rule would catch because they represent statistical outliers in the operator's own historical data. The first week of operation catches systematic issues in client data that have been silently causing problems for months or years.

**Efficiency metrics:**
- Duplicate detection rate: 100% of exact duplicates, >95% of near-duplicates
- Critical anomaly detection rate: target >90% of records with missing required fields
- False positive rate: target <2% of clean records flagged as anomalies
- Time to review anomaly report: target <5 minutes for a 3,000-parcel manifest (anomalies are rare; the report shows only flagged records)

### Innovation 4: Late Manifest Merge Engine

**What it does:**

When a late manifest upload arrives after routes have already been planned (e.g., a 6:45am upload when drivers are departing at 7:30am), the merge engine:

1. **Parses and validates** the late manifest using the same pipeline as normal manifests (LLM schema inference if new format, address validation, anomaly detection), targeting <90 second processing time for up to 200 late parcels.
2. **Identifies affected routes**: For each new stop, determines which existing route's geographic cluster it belongs to, using the existing stops' geocoordinates and a nearest-centroid algorithm.
3. **Triggers partial re-optimisation**: Rather than re-optimising all 35 routes (expensive and disruptive — drivers may already be en route), the engine re-optimises only the specific route segments affected by the new stops. Uses the same routing engine (Google OR-Tools or equivalent) but with a constraint that already-completed stops are locked.
4. **Computes impact metrics**: Estimates time added to affected routes, flags routes that may exceed the driver's planned end time, and highlights any new stops that cannot be accommodated without violating route constraints.
5. **Pushes updates to driver apps**: Sends updated route order and new stop details to the affected drivers' mobile apps via push notification. No ops manager phone call required.
6. **Notifies the dispatcher**: Generates a summary — "7 new stops merged into routes 3, 7, and 12. Route 7 extended by 23 minutes. Driver on Route 7 may run late by 18 minutes vs. schedule."

**Why it's 10x better:**

Currently, every competitor requires manual re-planning for late additions. The ops manager must identify which route to add the stop to, manually edit the route, notify the driver by phone or text, and estimate the schedule impact. For 10 late parcels across 35 routes, this takes 20–30 minutes. For 50 late parcels, it is unmanageable. The merge engine reduces this to zero manual intervention; the ops manager receives a summary notification and approves the change with one click (or auto-approves if within configured thresholds).

**Efficiency metrics:**
- Processing time for late manifest (<200 parcels): target <90 seconds end-to-end
- Manual intervention required for typical late merge: 0 minutes (one-click approve)
- Driver notification latency after merge approval: <30 seconds
- Route quality degradation from late merge vs. full re-optimisation: target <3% increase in total route distance

---

## Why Now

Three years ago, building LLM-powered schema inference would have required a large team of ML engineers, months of training data collection, and a narrow model that only handled formats it had been explicitly trained on. The fundamental problem was that generalising across arbitrary, unseen file formats required language understanding that narrow ML models could not provide.

Large language models, and Claude specifically, change the calculus entirely:

**Zero-shot and few-shot generalisation**: Claude can read a CSV header row it has never seen before and correctly infer that `ship_to_address1` means `address_line_1` because it understands the semantic meaning of both strings. No training data required for new formats. No retraining when a client changes their export format.

**Structured output**: The Claude API supports structured JSON output with schema enforcement. The schema inference engine can define the exact JSON structure it needs (field name, mapped canonical field, confidence, reasoning) and receive it reliably. This was not possible with earlier generation models that produced free-form text requiring fragile regex parsing.

**Cost economics**: The cost of a Claude API call to process a 1,000-row manifest header and sample rows is approximately $0.002–$0.005 per manifest at current pricing. At 5–8 manifests per day, this is less than $0.04/day in LLM inference cost. The economics make LLM inference at ingestion time entirely viable.

**API reliability and latency**: The Claude API returns schema inference results in 1–3 seconds for a typical manifest header. This is fast enough to include in a synchronous ingestion pipeline without degrading UX.

**Australian address data**: GNAF/PSMA data has been available for years, but integration with it required significant engineering effort to build a validation service on top. Mature Python libraries (`address`, `pygnaf`) and the PSMA Data API now make GNAF integration a 2–3 week engineering project rather than a 3-month infrastructure project.

**Isolation Forest maturity**: Scikit-learn's Isolation Forest implementation has been production-stable for several years. The key enabler is now operational: having enough historical delivery data (this operator has years of records) to train a meaningful anomaly model. A business at the $7M revenue scale has sufficient data volume to produce a useful model.

The combination of these factors — LLM maturity, structured output APIs, GNAF accessibility, and ML library maturity — makes 2025–2026 the right window to build this product. A year earlier, the LLM economics and reliability were not there. A year later, a well-funded competitor may have closed the gap.

---

## Target Metrics

The following metrics define product success at the end of a 12-week implementation cycle and at the 6-month operational milestone.

### Processing Time

| Task | Baseline (Manual) | End of Phase 2 (Week 6) | End of Phase 4 (Week 12) | 6-Month Target |
|---|---|---|---|---|
| Daily manifest normalisation (all clients) | 60–90 minutes | 20–30 minutes | <10 minutes | <5 minutes |
| Single new-format manifest (first upload) | 15–20 minutes | 3–5 minutes (review LLM mapping) | 2–3 minutes | 2 minutes |
| Known client manifest (learned format) | 15–20 minutes | 5–10 minutes | 0 minutes (auto) | 0 minutes (auto) |
| Late manifest merge (<50 parcels) | 20–30 minutes | N/A (not built yet) | <2 minutes (auto) | <90 seconds (auto) |
| Address validation per 1,000 records | Done post-dispatch (zero pre-validation) | <60 seconds | <30 seconds | <20 seconds |

### Error Rates

| Error Type | Baseline | Week 12 Target | 6-Month Target |
|---|---|---|---|
| Address errors reaching routing (per 1,000 records) | ~8–15 estimated | <3 | <1 |
| FTDs attributable to bad ingestion data (per 1,000 deliveries) | ~3–5 estimated | <1.5 | <0.8 |
| Duplicate records reaching routing (per day) | ~3–5 estimated | 0 (100% detection) | 0 |
| Missing required fields reaching routing | Unknown (untracked) | 0 (100% detection) | 0 |

### Automation Rates

| Metric | Week 6 | Week 12 | 6-Month Target |
|---|---|---|---|
| Manifests processed without human schema review | 30% (newly learned clients) | 80% | 95% |
| Manifests processed without any human intervention | 10% | 65% | 90% |
| Client formats with confirmed schema profiles | 2–3 | All 4 current clients | All clients + new onboards auto-learning |

### Operational Resilience

| Metric | Baseline | 6-Month Target |
|---|---|---|
| Dependency on ops manager for daily ingestion | Critical (single point of failure) | Eliminated (automated pipeline, any staff member can manage exceptions) |
| Maximum tolerable ops manager absence | 0 days (immediate degradation) | 5+ days (automated processing continues; only anomaly review requires human attention) |
| New client onboarding time (manifest format setup) | 1–2 hours manual configuration | <15 minutes (upload sample manifest, review LLM mapping, confirm) |

### Financial Impact Targets

| Impact Category | Baseline Annual Cost | 6-Month Projected Saving |
|---|---|---|
| Ops manager time on normalisation | ~$18,000 (312 hours @ $58/hr fully loaded) | ~$16,000 (90% reduction) |
| Address-error-driven redeliveries | $4,000–$12,000 | $2,500–$8,500 (65% reduction) |
| SLA penalties from ingestion errors | $15,000–$25,000 | $10,000–$18,000 (65% reduction) |
| Invoice dispute overhead | $7,200–$30,000 | $5,000–$22,000 (70% reduction) |
| **Total projected annual saving** | **$44,200–$85,000** | **$33,500–$64,500** |

At a $7M revenue base with 8–12% net margins ($560K–$840K net), saving $33K–$64K annually represents a 4–8% improvement in net margin — significant for a business at this scale. Combined with the ops manager time freed for higher-value work (client relationship management, driver performance, growth planning), the ROI on this product capability is achievable within the first quarter of full deployment.
