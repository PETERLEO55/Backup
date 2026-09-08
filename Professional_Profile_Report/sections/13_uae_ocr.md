# Project 13: UAE OCR AI (FastAPI / Azure Document Intelligence)

## 1. Project Overview

- **Project name:** UAE OCR AI, exposed as the "UAE OCR API" microservice (document no. AT/uae-ocr-api/V2, app version `2.0.0`, dated 01-Sep-2026; FastAPI app description string `'UAE OCR powered by Azure Document Intelligence Service'`). Client: Qatar Insurance Group (QIC / Anoud).
- **Business domain:** Insurance — UAE motor claims (intimation, registration, liability determination, subrogation recovery, underwriting).
- **Target users (as listed in the document):** (1) internal claims and policy administration systems that submit scans and consume the JSON to automate motor claim registration, intimation, and underwriting; (2) motor claims operations teams within QIC / Anoud who need liable-party, affected-party, driver-license, and vehicle-registration data to determine liability and subrogation recovery; (3) executive and technical management evaluating extraction efficiency, service reliability, and auditability; (4) external customer-facing web or mobile intimation portals uploading documents on behalf of policyholders.
- **Problem statement:** UAE motor claims arrive with police accident reports from three jurisdictions (Sharjah/Rafid, Abu Dhabi/Saaed, Dubai Police) — each with its own layout, some with legacy/modern variants or editable/scanned forms — plus six card layouts (UAE Driving License, Emirates Residence ID, Vehicle Registration "Mulkiya", front and back). Content is bilingual Arabic/English, dates are inconsistent, insurer names are free text, and uploads may be blank, corrupted, upside down, or the wrong document type. Claims handlers transcribed accident numbers, dates, vehicle specs, damage descriptions, and driver identities by hand, and bad uploads risked propagating corrupt records into policy and claims administration platforms.
- **Project objective:** Build an internal AI backend that accepts a PDF or image, verifies it is the expected document class, extracts fields via a cost-optimized hybrid of local text-layer parsing and custom-trained Azure Document Intelligence models, normalizes dates to `DD/MM/YYYY`, translates Arabic to English while preserving reshaped Arabic, resolves insurer names to canonical internal codes, and returns one standardized JSON envelope — or a structured error envelope when the input is unusable.
- **Expected business outcome (the document's own "Business Benefits" section):** *Operational* — elimination of manual data transcription (accident numbers, dates, coordinates, vehicle specs, damage descriptions, driver identities); zero-cloud-cost local extraction bypass for digital Abu Dhabi and Dubai reports; intelligent layout auto-classification removing manual pre-sorting of Sharjah documents; deterministic input quality enforcement protecting downstream core insurance databases; automated subrogation entity resolution; seamless bilingual reconciliation for legal audits and regulatory compliance. *Strategic* — a standardized multi-emirate claims pipeline (one unified JSON schema across three police jurisdictions); a decoupled, maintainable microservice architecture in which changes to one emirate's reporting template or Azure model "do not risk introducing regressions in other document classes"; turnkey containerized cloud and on-premise portability; regulatory compliance through strict `DD/MM/YYYY` standardization and right-to-left word reconstruction that "prevent transcription errors in policy enforcement and statutory reporting". No quantified accuracy, throughput, or cost figures are given.

## 2. Role and Responsibilities

**Core responsibilities (inferred from scope and build):**

- Designed the star-shaped architecture: `main.py` entry point, `utils.py` shared core, emirate handlers (`sharjah.py`, `abudhabi.py`, `dubai.py`) and `id_card_parser.py`, each importing only from `utils.py`.
- Built the FastAPI service: `GET /health` plus nine `POST` extraction endpoints, multipart `UploadFile` handling and integrity validation, and two global exception handlers (`OCRError` -> 400/404/422/502 envelopes; generic `Exception` -> HTTP 500 with `logger.critical` stack traces).
- Integrated ten custom Azure Document Intelligence models (four police-report, six card, all `anoud_ocr_*`); training/labeling is implied by "custom-trained" but not described (inferred).
- Engineered the hybrid extraction engine: `is_scanned_pdf` detection, PyMuPDF local text/table parsing for digital PDFs, Azure delegation for raster scans.
- Implemented pre-cloud layout classification: Tesseract (`--psm 6`) on the top third of page 1 at 150 DPI, keyword voting between `anoud_ocr_uae_PR_sharjahnew` and `anoud_ocr_uae_PR_sharjahold`.
- Implemented deterministic quality gates: anchor-keyword verification (>= 50% hit rate) for cards and the `> 3 non-empty fields` gate (`ensure_report_parsed`) for police reports, both HTTP 422.
- Built dual-language insurer resolution with RapidFuzz token-sort / token-set (threshold 85) against 42 `insurers.json` records.
- Integrated Azure Translator; handled Arabic reshaping and bidi display with `arabic-reshaper` and `python-bidi`.
- Containerized the service (Python 3.11-slim, bundled Tesseract, OpenCV/PyMuPDF runtime deps, non-root uvicorn) for on-premise, Azure Container Apps, or Kubernetes.

**Supporting tasks:**

- Defined the response envelope (`message`, `messageType: "S"`, `response`) and per-document field schemas.
- Wrote numeric-identifier extractors: 13+ digit Saaed incident numbers via regex (stated); Dubai 5-6 digit security code and 8+ digit report number (extraction method not stated; regex inferred).
- Parallelized Dubai vector-PDF page and party extraction with `ThreadPoolExecutor(max_workers=6)`.
- Implemented right-to-left polygon sorting to rebuild Arabic `makeModelColorYear` strings.
- License-back post-processing: `SequenceMatcher` (threshold 0.6) normalization of `transmission_type` and permitted-vehicle mapping to English/Arabic lists.
- Date normalization via `python-dateutil`; `RotatingFileHandler` on `logs/app.log` (10,485,760 bytes / 10 MB, 5 backup files); three-path Tesseract discovery (`TESSERACT_PATH` env var, `shutil.which('tesseract')`, AppData local install path); `.env` configuration (its contents — presumably Azure endpoints and keys — are never specified in the document); `/health` probe returning status, service name, and version string.
- Exposed the framework-default documentation surface: `GET /docs` (Swagger UI), `GET /redoc` (ReDoc), `GET /openapi.json` (OpenAPI 3.1 schema); tagged `/health` under `monitoring` and the nine extraction endpoints under `UAE`.
- Authored the technical documentation (endpoint reference, verified constants/thresholds table, business benefits); the document states its content was derived "directly from project source files, configuration manifests, and architectural documentation" (inferred that the candidate authored it).

**Estimated ownership level: Senior AI Engineer (inferred).**
Rationale: the document describes end-to-end build of an "enterprise document intelligence microservice" (its production status and traffic are not stated) — ten endpoints, ten custom cloud OCR models, three police-report pipelines with editable and scanned branches, a local OCR classifier, fuzzy entity resolution, machine translation, structured error contracts, log rotation, and a hardened Docker image targeting Kubernetes/Azure Container Apps. Design decisions (star-shaped dependencies, sync endpoints on the AnyIO pool, cost-avoiding local bypass) are deliberate and justified. No evidence of managing engineers or of architecture beyond this service, so Solution Architect / Technical Lead is not supported; the breadth of AI, backend, and deployment work exceeds Developer or ML Engineer. Note that the document describes itself as derived "directly from project source files, configuration manifests, and architectural documentation" and refers to "verified" constants and launch commands, so it reads as a code-derived write-up; it never names the author or the author's title.

## 3. End-to-End Workflow

**Processing stages:**

1. **Upload and validation.** Client POSTs `multipart/form-data` with a required binary `file` field (FastAPI `UploadFile`). Police-report endpoints are documented as accepting PDF document bytes; card endpoints accept image files (JPEG, PNG). FastAPI performs stream reading and validates file integrity before handing the payload to the domain dispatcher. Empty files or invalid PDF streams raise HTTP 400.
2. **Dispatch.** Police reports route to the emirate handler; cards go to `id_card_parser.py`, which selects the Azure model and runs type validation.
3. **Layout / scan detection.** Sharjah: `page_to_image` rasterizes the top third of page 1 at 150 DPI, Tesseract `--psm 6` counts layout keywords, a vote picks new or old model (ties -> new), and the whole document is then submitted to Azure — the local text-layer bypass is documented only for Abu Dhabi and Dubai reports. Abu Dhabi and Dubai: `is_scanned_pdf(doc)` branches between local and cloud. Dubai scanned: Tesseract on 150 DPI page crops keeps only pages scoring `> 2` of 7 keywords before any page is submitted to Azure (the constants table states `> 2 keywords`, the endpoint text `>= 3 keywords` — equivalent for integer counts, so the two phrasings agree).
4. **Extraction.** Editable: PyMuPDF parses JSON text blocks and table cells locally (Saaed `_parse_editable`: `FAULTY PARTY`, `NON FAULTY PARTY`, `PROPERTY DAMAGES` sliced into indexed coordinate dictionaries, Arabic descriptions shaped, claims translated; Dubai: text spans extracted across all pages, incident-details table plus affected/liable party tables in parallel threads, trailing image-coordinate artifacts popped). Scanned: bytes go to the matching custom Azure model (Saaed `_parse_scanned` parses the returned parties array into driver, insurance, and vehicle profiles; Dubai submits only the pages that passed the keyword filter).
5. **Type verification (cards).** Raw OCR lines are checked against per-card anchor phrases; below 50% the request is rejected with HTTP 422 (e.g. `'Provided image is not a license front'`).
6. **Quality gate (police).** `ensure_report_parsed` rejects `<= 3` non-empty fields with HTTP 422 ("document quality is low"). The document states this gate generically for police reports in its Key Capabilities and constants table, and lists it explicitly for the Saaed `_parse_scanned` branch and the Dubai endpoint; the Rafid endpoint's documented error list names only 400 / 422 (Tesseract missing) / 502, so whether the field-count gate also runs for Sharjah is not stated.
7. **Normalization.** Party fields grouped by prefix (`liable/affected` for Sharjah new, `faulty_/nonfaulty_` for old); dates to `DD/MM/YYYY`; Arabic reshaped for display while unshaped forms are kept for translation and lookup; Azure word polygons sorted right-to-left.
8. **Translation and entity resolution.** Azure Translator produces `*_english` fields; insurer names are fuzzy-matched against `insurers.json` to emit `insurance_company_code` (police documents only, per the architecture diagram).
9. **Response.** `{ "message": "success", "messageType": "S", "response": {...} }`, or an `OCRError` envelope (400/404/422/502); unhandled faults return 500.

**Data flow:** PDF/JPEG/PNG bytes -> PyMuPDF document (handled as a byte stream; the document never mentions writing files to disk, so "in-memory" is inferred) / 150 DPI NumPy raster via `page_to_image` -> Tesseract text (classification and page filtering only) -> either PyMuPDF JSON text blocks and coordinate-indexed table cells (local) or Azure field/word-polygon results (cloud) -> Python dicts -> Azure Translator for Arabic strings -> RapidFuzz lookup against `insurers.json` -> JSON envelope.

**Documented architecture diagram (Figure 1, "End-to-End Architecture"), five stages:** FastAPI service (validation and routing) -> routing endpoints (validation and routing) -> Azure Document Intelligence (extracts fields) -> Normaliser (insurer code, **only police docs**) -> JSON conversion.

**Integrations and dependencies:** Azure Document Intelligence (ten custom models) and Azure Translator via the `azure-ai-*` SDKs; local Tesseract via pytesseract; PyMuPDF, NumPy, RapidFuzz, python-dateutil, arabic-reshaper, python-bidi; `insurers.json` (42 records); `.env` configuration. Pinned versions are listed in Section 4.

```
Client -> FastAPI (validate, dispatch)
       -> [text layer? yes -> PyMuPDF local parse | no -> Tesseract classify/filter -> Azure DI custom model]
       -> anchor / field-count gate -> date normalize + Arabic reshape/bidi
       -> Azure Translator -> RapidFuzz insurer resolution -> JSON envelope
```

**Architecture Summary:** A flat, star-shaped FastAPI microservice: `main.py` declares the ten endpoints and two global exception handlers; every domain module depends only on `utils.py`. Endpoints are plain `def` functions, so FastAPI runs them on the AnyIO worker thread pool and blocking Azure calls, PyMuPDF rendering, and Tesseract stay off the event loop. Extraction is hybrid: digital PDFs are parsed locally at zero cloud cost; raster scans go to one of ten custom Azure models chosen statically (cards) or by a local Tesseract classifier (Sharjah old/new, Dubai page filtering). Deterministic post-processing — anchor verification, field-count gating, date normalization, reshaping, translation, fuzzy insurer resolution — yields one JSON schema across three police jurisdictions and six card types. Delivery is a hardened Docker image (Python 3.11-slim, bundled Tesseract, non-root uvicorn on port 8000).

## 4. Technologies and Tools Used

| Category | Technologies (pinned versions where given) |
|---|---|
| Programming languages | Python (Docker base image Python 3.11-slim) |
| Frameworks | FastAPI `>=0.110.0`; Uvicorn `>=0.28.0` (ASGI); AnyIO worker thread pool (via FastAPI) |
| Libraries | PyMuPDF (fitz) `>=1.24.0`; NumPy `>=1.26.0`; pytesseract `>=0.3.10`; RapidFuzz `>=3.6.0`; azure-ai-documentintelligence `>=1.0.0b1`; azure-ai-translation-text `>=1.0.0b1`; azure-core `>=1.30.0`; python-dateutil `>=2.9.0`; arabic-reshaper `>=3.0.0`; python-bidi `>=0.4.2`; stdlib `difflib.SequenceMatcher`, `concurrent.futures.ThreadPoolExecutor`, `logging.handlers.RotatingFileHandler`, `re`, `shutil`; OpenCV listed as a Docker runtime dependency |
| AI/ML models | Ten custom Azure Document Intelligence models: `anoud_ocr_uae_PR_sharjahnew`, `anoud_ocr_uae_PR_sharjahold`, `anoud_ocr_uae_PR_abudhabi`, `anoud_ocr_uae_PR_dubai`, `anoud_ocr_custom_uae_df`, `anoud_ocr_custom_uaedb`, `anoud_ocr_custom_uaerf`, `anoud_ocr_custom_uaerb`, `anoud_ocr_custom_uae_vf`, `anoud_ocr_custom_uae_vb`; Azure Cognitive Services Translator |
| OCR tools | Azure Document Intelligence (cloud, custom models); Tesseract OCR via pytesseract (local classification, `--psm 6`, 150 DPI) |
| Databases | Not stated in documentation (master data is a flat file, `insurers.json`, 42 records) |
| Cloud platforms | Microsoft Azure (Document Intelligence, Translator); Azure Container Apps and Kubernetes named as deployment targets |
| APIs | Exposed: REST with OpenAPI 3.1 (`/docs` Swagger UI, `/redoc`, `/openapi.json`); `/health` under the `monitoring` tag, the nine POST endpoints under the `UAE` tag. Consumed: Azure Document Intelligence API, Azure Translator Text API |
| DevOps tools | Docker (hardened Dockerfile, non-root uvicorn, `--env-file .env`); RotatingFileHandler logging (`logs/app.log`, 10 MB, 5 backups); `/health` probe |
| Version control | Not stated in documentation |
| Deployment tools | Docker (`docker build -t uae-ocr .`, `docker run -p 8000:8000 --env-file .env uae-ocr`); Uvicorn (`uvicorn main:app --host 0.0.0.0 --port 8000` / `python main.py`); targets: on-premise, Azure Container Apps, Kubernetes |
| Document processing tools | PyMuPDF (text-layer/table extraction, `page_to_image` rasterization); arabic-reshaper and python-bidi; python-dateutil |
| Automation tools | Not stated in documentation (no CI/CD or workflow tooling described) |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Star-shaped modular design with single-responsibility modules importing exclusively from `utils.py`; custom `OCRError` mapped to dynamic HTTP status codes; global exception handlers with `logger.critical` stack traces; regex extractors for numeric identifiers; rotating file logging (10 MB, 5 backups); environment-based configuration; three-path native-binary discovery for cross-platform portability.
- **AI Engineering:** Integrating ten custom Azure Document Intelligence models; hybrid local/cloud routing that calls cloud AI only when a text layer is absent; using a cheap local OCR pass to select the right neural model before paying for the cloud call.
- **Machine Learning:** Custom document-model training on Azure Document Intelligence is implied by "custom-trained" (inferred); no training procedure, evaluation, or metrics are described. Approximate string matching with tuned thresholds (RapidFuzz 85, `SequenceMatcher` 0.6).
- **NLP:** Arabic/English fuzzy entity resolution (token-sort/token-set); Arabic reshaping and bidi; neural MT integration; right-to-left word reconstruction from OCR polygons; keyword-vote document classification.
- **Computer Vision:** PDF rasterization to NumPy arrays at 150 DPI; header-region cropping for OCR; geometric sorting of word polygons; coordinate-based table slicing. No CNN/YOLO training here.
- **Data Engineering:** Normalizing heterogeneous report formats into one JSON schema; date standardization; party-prefix grouping; flat-file master-data lookup; stripping image-coordinate artifacts.
- **Cloud:** Azure Document Intelligence and Azure Translator SDK integration (`azure-ai-documentintelligence`, `azure-ai-translation-text`, `azure-core`); container packaging for on-premise / Azure Container Apps / Kubernetes; `.env`-file configuration passed via `--env-file` (the document does not say what the file contains; Azure endpoint and key are inferred).
- **MLOps:** Model-per-document-type routing by model ID; deterministic guardrails around model output; health endpoint and rotating logs. No monitoring, retraining, or experiment tracking described.
- **API Development:** FastAPI multipart endpoints, OpenAPI 3.1 docs, consistent success/error envelopes, blocking-work isolation on the AnyIO pool, HTTP semantics 400/404/422/502/500.
- **Prompt Engineering:** Not demonstrated in this project (no LLM prompting; extraction uses trained document models and deterministic logic).
- **System Design:** Cost-aware routing, pre-cloud classification, fail-fast validation protecting downstream systems, concurrency choices, containerized portability.

## 6. Detailed Technical Contributions

**Features implemented**

- `GET /health` -> `{"status": "healthy", "service": "AI OCR API", "version": "2.0.0"}`.
- `POST /police_report/rafid`, `/police_report/saaed`, `/police_report/dubai` — police-report extraction. The one full sample envelope given (Rafid) returns `nature_of_loss, reason_of_accident, report_number, report_date, incident_time, road_condition, is_unknown_damage, incident_number, street_name, incident_date, weather_condition, is_legal_claim, accident_location, region, emirate, incident_type, vehicle` (vehicle count, e.g. `"2"`), `primary_reason, visiblity_condition, damage_description`, plus `vehicle_party[]` with `driver_detail` (`driver_name`, `driver_name_english`, `license_expiry_date`, `license_source`, `license_no`), `insurance_detail` (`insurance_type`, `insurance_policy_no`, `insurance_company`, `insurance_company_english`, `insurance_company_code`, `insurance_validity`), `vehicle_detail` (`owner_name`, `plate_no`, `vehicle_model`, `vehicle_make_and_model`, `is_blamed_party`, `chasiss_no`), and `injured_party[]`, `property_damage_party[]`, `claim_description{}`. Saaed is described in prose only, as "Abu Dhabi claim, property damage, and party structures". Dubai is described in prose as containing primary/secondary causes, visibility, `incident_number`, `security_code`, `vehicle_party` profiles, and `important_notes`. (`visiblity_condition` and `chasiss_no` spellings are as documented.)
- `POST /uae/license_front` -> `date_of_birth, expiry_date, issue_date, license_number, name, nationality, place_of_issue, traffic_code_number`.
- `POST /uae/license_back` -> `traffic_code_number, transmission_type` (`automatic gear` / `manual gear`), `permitted_vehicle.{english, arabic}` (documented sample: `english: ["light vehicle", "motorcycle"]` with the parallel Arabic list; category mapping example `'light vehicle'` -> `مركبة خفيفة`).
- `POST /uae/residence_front` -> `id_number, name, nationality, issue_date, expiry_date`.
- `POST /uae/residence_back` -> `card_number, occupation, employer, issuing_place, date_of_birth, sex`.
- `POST /uae/vehicle_front` -> `traffic_plate_no, place_of_issue, policy_no, insurance_company, insurance_expiry, mortgage_by, owner_name`.
- `POST /uae/vehicle_back` -> `chassis_number, engine_number, model_year, vehicle_make, vehicle_model, vehicle_type, empty_weight, gross_vehicle_weight (g_v_w), number_of_passengers`.

**Models used:** police — `anoud_ocr_uae_PR_sharjahnew`, `_sharjahold`, `_abudhabi`, `_dubai`; cards — `anoud_ocr_custom_uae_df` (license front), `_uaedb` (license back), `_uaerf` (residence front), `_uaerb` (residence back), `_uae_vf` (vehicle front), `_uae_vb` (vehicle back); Azure Translator for Arabic -> English; Tesseract for local classification only.

**Pipelines built:** the Sharjah classify-then-Azure pipeline; the Saaed `_parse_editable` / `_parse_scanned` split; the Dubai parallel-editable / filtered-scanned split (scanned branch reconstructs Arabic `makeModelColorYear` by sorting word polygons right-to-left); and the card select-model -> Azure -> anchor-verify -> normalize pipeline (see Section 3).

**APIs integrated:** Azure Document Intelligence custom-model analysis; Azure Translator Text; Tesseract via pytesseract. The service publishes OpenAPI 3.1 with Swagger UI and ReDoc.

**Data extraction methods:** native PDF text-layer and table extraction (PyMuPDF); cloud layout-model field extraction; regex for numeric identifiers; coordinate-indexed table slicing; polygon-order reconstruction; keyword scoring for classification.

**Validation logic**

- HTTP 400: empty upload or invalid PDF stream. HTTP 404: unknown card type.
- HTTP 422: anchor hit rate below `ANCHOR_MATCH_THRESHOLD = 0.5`; police report with `<= 3` non-empty fields ("document quality is low"); Tesseract binary missing (documented for the Rafid and Dubai endpoints).
- HTTP 502: Azure client uninitialized or analysis call failed. HTTP 500: unhandled exception, logged via `logger.critical` with stack trace.
- Errors are delivered as structured JSON envelopes by `@app.exception_handler(OCRError)` (dynamic status code) and `@app.exception_handler(Exception)` (500).
- Anchor sets, e.g. license front: `'driving license', 'License No.', 'Place of Issue'` plus an Arabic ID-number phrase; license back: `'Traffic Code No.', 'Permitted Vehicles', 'Police Vehicles', 'Give way to', 'This license'`; residence front: `'United Arab Emirates', 'Resident Identity Card', 'Port Security', 'Federal Authority', 'ID Number', 'Signature', 'nationality'`; residence back: `'Card Number', 'if you find this card', 'please return it to', 'Organization', 'nearest police station'`; vehicle front: `'vehicle License', 'Place of Issue', 'Traffic Plate No', 'Policy No', 'Mortgage  By', 'Ins. Exp.'`; vehicle back: `'vehicle Information', 'Model', 'changes to vehicle', 'owner Infomation must be notified', 'G. V. W', 'Veh. Type', 'Empty Weight'`.

**Configuration constants and thresholds (the document's "Implementation Constants & Thresholds" table, presented as verified from code):** `ANCHOR_MATCH_THRESHOLD` 0.5 (50%); `REPORT_QUALITY_MIN_FIELDS` `> 3` non-empty; `INSURER_FUZZY_THRESHOLD` 85 (RapidFuzz token score, 0-100); `CATEGORY_FUZZY_THRESHOLD` 0.6 (SequenceMatcher ratio for license-back vehicle categories); `TESSERACT_RASTER_DPI` 150; `TESSERACT_CANDIDATE_PATHS` 3 (env var, `shutil.which`, AppData); `LOG_ROTATION_MAX_BYTES` 10,485,760; `LOG_BACKUP_COUNT` 5; `DUBAI_WORKER_THREADS` 6; `DUBAI_SCANNED_PAGE_SCORE` `> 2` of 7 keywords; `INSURERS_MASTER_COUNT` 42 records.

**Automation workflows:** automatic Sharjah old/new selection (no manual pre-sorting); automatic scanned-vs-editable branching; automatic insurer-code resolution and translation; container startup via `python main.py` (which invokes `uvicorn.run('main:app', host='0.0.0.0', port=8000)`) or `uvicorn main:app --host 0.0.0.0 --port 8000`; Docker build/run (`docker build -t uae-ocr .`, `docker run -p 8000:8000 --env-file .env uae-ocr`).

**Optimization techniques:** PyMuPDF bypass avoiding per-page Azure fees and network latency; Tesseract only on a cropped header region; Dubai page pre-filtering so only qualifying pages are billed; `ThreadPoolExecutor(max_workers=6)` for Dubai parallelism; sync endpoints offloaded to the AnyIO pool.

**Performance improvements:** The document states local extraction runs "in milliseconds" and that parallelism "accelerates vector PDF table slicing"; no measurements are provided.

## 7. Challenges and Solutions

1. **Three jurisdictions, multiple layouts (technical).** Sharjah has old and new templates; Abu Dhabi and Dubai arrive as editable PDFs or raster scans. *Solution:* one Azure custom model per layout, a Tesseract keyword vote on the page-1 header for Sharjah (ties -> new), and `is_scanned_pdf` to branch local vs cloud. *Alternatives:* a composed Azure model with built-in classification, or a small image classifier (analyst suggestion); the local vote is free and fast.

2. **Cloud OCR cost and latency (business/technical).** Each page sent to Azure incurs a fee and a round-trip. *Solution:* parse native text layers locally with PyMuPDF; pre-filter Dubai scan pages (`> 2` of 7 keywords). *Alternatives:* send everything to Azure, or run a full local OCR stack end-to-end (analyst suggestion) at the cost of accuracy on Arabic scans.

3. **Wrong, blank, or upside-down uploads (business).** *Solution:* anchor-phrase verification at a 50% hit rate for cards and a `> 3` non-empty-field gate for police reports, both HTTP 422. *Alternatives:* Azure confidence-score thresholds or a dedicated document-type classifier (analyst suggestion); anchors are deterministic and explainable to auditors.

4. **Free-text insurer names in two languages (technical).** *Solution:* RapidFuzz token-sort and token-set scoring (threshold 85) against Arabic and English aliases for 42 `insurers.json` records, emitting codes such as ADNIC -> `2600040`. *Alternatives:* embedding-based or LLM matching (analyst suggestion); deterministic matching was preferred for auditability.

5. **Arabic handling (technical).** Arabic must render, translate, and match correctly, and Azure returns Dubai vehicle strings as unordered word polygons. *Solution:* keep unshaped Arabic for translation/lookup and reshaped Arabic (arabic-reshaper + python-bidi) for display; sort polygons right-to-left; Azure Translator for `*_english` fields.

6. **Blocking work in an async framework (technical).** *Solution:* plain `def` endpoints dispatched to the AnyIO worker pool, plus `ThreadPoolExecutor(max_workers=6)` inside the Dubai parser. *Alternative:* async Azure SDK clients (analyst suggestion), which would still require executor jobs for CPU-bound PDF/Tesseract work.

7. **Tesseract portability (technical).** Tesseract is a native binary that must exist on every developer machine and in every container. *Solution:* three-path discovery (`TESSERACT_PATH` env var -> `shutil.which('tesseract')` -> AppData local install path) and a Docker image that bundles Tesseract OCR alongside the OpenCV/PyMuPDF runtime dependencies. *Analyst observation (the document states the behavior but does not flag it as a problem):* a missing binary surfaces as HTTP 422, conflating a server configuration fault with a client error; 500/503 would be more accurate.

8. **Contract-hygiene and hardening gaps (analyst observations drawn from documented facts; the document does not itself flag them):** `/health` reports `"service": "AI OCR API"` while the app title is `'UAE OCR API'`; response fields carry typos (`visiblity_condition`, `chasiss_no`) that are now part of the published contract; Azure SDKs are pinned to beta (`>=1.0.0b1`); insurer-code resolution applies only to police documents (per the architecture diagram), so `vehicle_front` returns `insurance_company` without a code; the full response schema is shown only for Rafid and license-back, with Saaed and Dubai described in prose; the Rafid error list omits the field-count quality gate; no authentication, rate limiting, tests, or CI/CD are described anywhere in the document.

## 8. Impact Analysis

- **Business impact:** Unified multi-emirate claims pipeline — disparate, non-standard accident report formats from three police jurisdictions normalized into a single JSON schema "across the organization"; automated liable-carrier identification for inter-company subrogation recovery; protection of core policy/claims databases from corrupt or truncated records; bilingual output preserving formatted Arabic presentation strings for legal audits and regulatory compliance.
- **Maintainability impact:** The flat, star-shaped architecture isolates domain logic into single-responsibility modules importing exclusively from `utils.py`, so (per the document) changes to one emirate's reporting template or Azure model do not risk introducing regressions in other document classes.
- **Portability impact:** A hardened Dockerfile bundling Python 3.11-slim, Tesseract OCR, OpenCV/PyMuPDF runtime dependencies, and a non-root uvicorn configuration is stated to allow "instant deployment" across on-premise servers, Azure Container Apps, or Kubernetes clusters.
- **Compliance impact:** Strict `DD/MM/YYYY` date standardization and right-to-left word reconstruction are stated to prevent transcription errors in policy enforcement and statutory reporting.
- **Productivity improvements:** Removes manual transcription of accident numbers, dates, coordinates, vehicle specs, damage descriptions, and driver identities, and manual pre-sorting of Sharjah reports.
- **Accuracy improvements:** No figures provided. Qualitatively, date normalization, RTL reconstruction, anchor gating, and insurer-code resolution are stated to prevent transcription errors and invalid data propagation.
- **Cost savings:** No monetary figures provided. The PyMuPDF bypass is stated to eliminate per-page cloud transaction costs for digital Abu Dhabi and Dubai reports; Dubai page filtering limits Azure submissions.
- **Time savings:** No measured latencies. Local extraction is stated to complete "in milliseconds"; parallel Dubai parsing accelerates table slicing.
- **User benefits:** Claims processors get structured, bilingual, code-resolved party data; integrating systems get a stable JSON contract with actionable HTTP errors; management gets a health probe and rotating logs.

The document provides no quantified accuracy, throughput, or cost-savings metrics; its figures are configuration constants and scope counts (10 endpoints, 10 custom Azure models, 3 emirates, 6 card layouts, 42 insurer records, 4 documented target-user groups, and the thresholds 50%, `> 3` fields, 85, 0.6, 150 DPI, 6 worker threads, `> 2` of 7 keywords, 10 MB / 5 log backups, 3 Tesseract lookup paths).

## 9. Interview Discussion Points

- Why route on the presence of a PDF text layer rather than always calling Azure: cost per page, latency, native-text fidelity; the risk of hybrid PDFs with partial text layers.
- How the Sharjah old/new classifier works and why Tesseract at 150 DPI on only the top third of page 1; keyword voting and the tie-break to new.
- Anchor verification: why a 50% hit rate, how anchor sets were chosen per card, how to tune false rejects vs false accepts.
- The `> 3 non-empty fields` gate: what it catches (blank, upside-down, wrong-type reports) and what it misses (partially wrong extractions).
- RapidFuzz token-sort vs token-set, why threshold 85 across Arabic and English aliases, and handling a new insurer or a near-tie.
- Arabic handling: reshaped vs unshaped strings, bidi layout, right-to-left polygon sorting for `makeModelColorYear`, translating the unshaped form.
- Concurrency: why endpoints are `def` not `async def`, what AnyIO does with them, and why `ThreadPoolExecutor(max_workers=6)` sits inside the Dubai parser.
- Error contract: `OCRError` -> 400/404/422/502, the generic 500 handler, and the debatable 422 for a missing Tesseract binary.
- Ten custom Azure models: how they were labeled/trained (be candid that the document does not say), how model IDs are managed, how you would version or retrain them.
- Deployment: Python 3.11-slim, bundled Tesseract, non-root uvicorn, `.env` secrets, on-prem/Azure Container Apps/Kubernetes; what is missing (auth, rate limiting, CI/CD, tests).
- What to measure next: field-level accuracy per document type, Azure calls avoided, p95 latency local vs cloud, rejection rates by reason.
- Schema hygiene: `visiblity_condition` / `chasiss_no` and the `/health` name mismatch, and evolving the contract without breaking consumers.
- Why one custom Azure model per layout (ten of them) rather than one composed/classifier model: per-layout labeling effort and model sprawl versus higher per-template accuracy and the ability to change one emirate's template without regressing others — tie this to the documented "no regressions in other document classes" benefit of the star-shaped module layout.
- Why Sharjah has no local text-layer bypass: the document documents the PyMuPDF zero-cost path only for Abu Dhabi and Dubai, and always sends Rafid documents to Azure after classification — was that a data reality (Rafid reports arrive as scans) or unfinished work?
- Why insurer-code resolution runs only on police documents: `vehicle_front` extracts `insurance_company` but emits no `insurance_company_code`, so a downstream consumer gets an unresolved free-text carrier name from the Mulkiya path — deliberate scoping to the subrogation use case, or a gap?
- Where the anchor check gets its text: the document says anchors are matched against "extracted OCR lines" but does not say whether those come from the Azure response or a separate local Tesseract pass — a meaningful cost difference, since anchor verification after the Azure call still pays for the call it rejects.
- The four documented consumer classes — internal claims/policy systems, claims operations teams, executive/technical management, and **external customer-facing intimation portals** — and what it means that an externally reachable upload path has no documented authentication or rate limiting.
- Error-envelope design: one `OCRError` type carrying a dynamic status code versus per-condition exception classes, and why the API returns a structured actionable envelope rather than a bare error, given the stated goal of preventing invalid data propagation into downstream policy and claims platforms.
- Operability story and its limits: `/health` (status, service, version), `RotatingFileHandler` at 10 MB with 5 backups, and what is absent — request tracing, per-model call metrics, alerting, and any record of Azure spend actually avoided.
- Beta Azure SDK pins (`azure-ai-documentintelligence`, `azure-ai-translation-text` at `>=1.0.0b1`) plus open-ended `>=` constraints across the whole stack: reproducibility risk and how you would move to pinned or hash-locked builds.

## 10. Architecture Explanation Points

The document's own Figure 1 draws five boxes — FastAPI service (validation and routing) -> routing endpoints -> Azure Document Intelligence (extracts fields) -> Normaliser (insurer code, only police docs) -> JSON conversion — which is the skeleton to expand on when whiteboarding.

Start with the client: an internal claims system or an external intimation portal POSTs a PDF or image to one of the nine POST extraction endpoints (ten routes in total including `GET /health`). Draw `main.py` as the hub that validates the upload, with `sharjah.py`, `abudhabi.py`, `dubai.py`, and `id_card_parser.py` as spokes sharing only `utils.py`. Next draw the decision diamond: does the PDF have a native text layer? Yes -> PyMuPDF parses tables locally in milliseconds with no cloud cost. No -> a cheap Tesseract pass either picks the right Azure model (Sharjah old vs new) or filters which pages are worth sending (Dubai), then the scan goes to one of ten custom Azure Document Intelligence models. Below that, draw the normalization strip: anchor/field-count gates returning HTTP 422, `DD/MM/YYYY` dates, Arabic reshape/bidi plus RTL polygon sorting, Azure Translator for English fields, and RapidFuzz resolution of insurer names to internal codes from `insurers.json`. Finish with the JSON envelope and the handler mapping `OCRError` to 400/404/422/502 and everything else to 500.

Key decisions, each with the reason to state out loud: (1) **cost-first hybrid routing** — check for a native text layer before spending a cloud call, because per-page Azure Document Intelligence charges and a network round-trip are avoidable whenever the PDF already carries high-fidelity text; (2) **cheap local OCR as a router, not an extractor** — Tesseract runs only on a 150 DPI crop of the page-1 header (Sharjah) or on page crops (Dubai), so it decides *which* model or *which* pages get paid for, and its own accuracy on Arabic never reaches the output; (3) **deterministic, auditable validation instead of model confidence scores** — anchor keyword hit rates and non-empty-field counts are explainable to auditors and stable across model retrains; (4) **one custom model per layout** rather than one generalized model, so each of the ten templates can be retrained independently; (5) **sync `def` endpoints on the AnyIO pool** because every heavy step (Azure I/O, PyMuPDF rendering, Tesseract) is blocking, plus a nested `ThreadPoolExecutor(max_workers=6)` where Dubai page and party extraction can genuinely run in parallel; (6) **a self-contained Docker image** with Tesseract and the OpenCV/PyMuPDF runtime baked in and a non-root uvicorn user, so the same artifact runs on-premise, on Azure Container Apps, or on Kubernetes; (7) **fail fast at the edge** — a 422 at the boundary is cheaper than a corrupt record in the core claims database.

Trade-offs to concede: Tesseract is a native dependency with a startup failure mode that currently surfaces as a client-side 422; the modular one-model-per-layout design multiplies labeling and lifecycle work across ten Azure models with no documented retraining or versioning process; the insurer master is a 42-record flat file rather than a database, so adding a carrier is a redeploy; insurer resolution is wired only into the police path; the Azure SDKs are beta pins and every dependency is an open `>=` range; and there is no auth, rate limiting, test suite, or CI/CD at the service layer even though one documented consumer class is an external customer-facing portal. Next improvements: per-field accuracy telemetry and Azure-call-avoidance metrics, async Azure clients with bounded concurrency, confidence-score fallbacks alongside anchors, an API-key/JWT layer, contract tests pinning the response schema (typos included, then a versioned migration off them), and a CI pipeline that builds, scans, and pushes the image.
