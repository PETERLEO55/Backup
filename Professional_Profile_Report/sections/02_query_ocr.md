# Project 2: UAE OCR API — Query-Based Extraction (FastAPI / Azure Document Intelligence)

Source document: `4-8-Query_OCR_Documentation.md` (Document no. AT/query-ocr/V1, dated 01-Sep-2026, Client: Qatar Insurance Group). Service self-identifies as "Query-Based OCR & Bilingual Extraction API", version 1.0.0.

## 1. Project Overview

- **Project name:** UAE OCR API (project name `Query_OCR`), technology line "FastAPI / Azure Document Intelligence (Query-Based)".
- **Business domain:** Insurance — customer onboarding / KYC, policy issuance, automated underwriting, and claims registration for client Qatar Insurance Group, covering UAE/GCC identity and commercial documents. Documents handled are UAE Driving Licenses (front/back), Emirates ID / Resident Cards (front/back), Vehicle Registration Cards / Mulkiya (front/back), and Department of Economy & Tourism Commercial Trade Licenses. The document states that the repository contains no user persona documentation, access control rosters, or role declarations; it infers the intended consumers (upstream insurance administration systems, digital customer onboarding portals, automated underwriting pipelines, third-party policy verification microservices) from code artifacts and schema design.
- **Problem statement:** Insurance onboarding and policy administration require transcribing identity and vehicle credentials by hand. Traditional template-based OCR and bounding-box coordinate scraping break under rotation, formatting shifts, resolution degradation, and document redesigns; the document also ties bilingual (Arabic/English) enrichment to "core operational compliance standards", implying conventional OCR does not produce the bilingual values insurers need for compliance and core-system ingestion (inferred).
- **Project objective:** Deliver a stateless, synchronous RESTful microservice that runs zero-shot semantic queries against document layouts using Azure AI Document Intelligence's `QUERY_FIELDS` feature on the `prebuilt-layout` model, enriches key fields with Arabic/English translations via Azure AI Translation, and returns strongly typed Pydantic JSON in a standard enterprise envelope — with no custom model training, annotation datasets, or weight deployment.
- **Expected business outcome:** Elimination of manual data entry ("up to 11 discrete structured fields per document side in a single API call"), zero-shot adaptability to new formats through `/api/v1/extract/query`, deterministic bilingual normalization, contract-safe ingestion into insurance core systems, and support for UAE Central Bank / KYC mandates. No ROI, cost, accuracy, or throughput figures are recorded.

## 2. Role and Responsibilities

**Core responsibilities (inferred from what was built):**

- Designed the end-to-end architecture: FastAPI gateway -> Azure Document Intelligence query extraction -> Azure translation enrichment -> typed bilingual JSON (Figure 1 in the document).
- Ran feasibility research in `workbook.ipynb` (Cells 1-5), proving that `QUERY_FIELDS` on `prebuilt-layout` extracts key fields from real UAE scans without custom models.
- Authored per-document query presets — `df`, `db` (driving license), `rf`, `rb` (resident ID), `vf`, `vb` (vehicle Mulkiya), `tr` (trade license) — carried into `services/ocr_service.py`.
- Built the Azure client layer (`services/azure_client.py`) wrapping `azure-ai-documentintelligence` 1.0.2 and `azure-ai-translation-text` 2.0.0.
- Implemented the FastAPI application (`main.py`, `routers/`): 10 declared routes (2 health/root, 8 extraction), CORS middleware, async lifespan events, `/api/v1` namespace.
- Designed the Pydantic v2 domain models in `models/schemas.py` (`DrivingLicenseFrontData`, `PermittedVehicle`, `TradeLicenseData`, and the `BaseResponse` envelope).
- Implemented `services/translation_service.py` with `@lru_cache(maxsize=1024)`, default `en` -> `ar`, and graceful exception fallbacks.
- Hardened the prototype into production: Pydantic `BaseSettings` configuration, `UploadFile` stream ingestion, extension whitelisting, empty-file detection, centralized exception handling.

**Supporting tasks (inferred):**

- Settings management via `config.py` (translator region `centralus`, host `0.0.0.0`, port `8000`, `.env` loading with environment fallbacks).
- Test tooling setup with `pytest` 9.1.1 and `httpx` 0.28.1 (listed in the stack as automated test runner and HTTP test client; test coverage details are not described).
- Producing the technical documentation itself, including the "Verified Implementation Constants" table with file/line references and the notebook-to-production parameter evolution record.
- Curating sample data for experimentation (e.g. `Dubai - Sample 1.pdf` trade license).

**Estimated ownership level:** AI Engineer (end-to-end owner; inferred).

Rationale: the document shows full-lifecycle ownership — POC notebook, architecture, two Azure cognitive service integrations, API contract design, typed schemas, caching, validation, error handling, and documentation. Design drifts were "intentionally resolved" during the notebook-to-production transition, evidence of engineering judgment beyond a task-executing developer. Scope, however, is a single microservice with no described deployment infrastructure, authentication, observability, or team coordination, so Solution Architect / Technical Lead is not supported. Production-readiness elements (health probes, enterprise envelope, extension whitelist, pytest/httpx test tooling — actual tests are not described) place this at the upper end of AI Engineer, bordering Senior AI Engineer.

## 3. End-to-End Workflow

**Processing stages:**

1. **Client upload.** An upstream system (underwriting platform, onboarding portal, mobile app, verification microservice) sends `multipart/form-data` with field `file` (binary scan: `.jpg`, `.jpeg`, `.png`, `.pdf`, `.tiff`, `.bmp`) to one of the eight `POST /api/v1/extract/...` endpoints. The dynamic endpoint additionally accepts `query_fields` as a comma-separated list or JSON-array string (e.g. `Tax_Invoice_No, Total_Amount, IBAN`).
2. **Gateway validation.** FastAPI (`main.py`, `routers/`) applies CORS headers and, in the document's words, "validates MIME extensions and payload boundaries": file existence, non-zero length, and extension whitelist (`routers/driving_license.py:20`). Extraction routers are bound under the `/api/v1` prefix at `main.py:47-52`. Failures map to `400 Bad Request` (missing/empty file or empty query fields), `415 Unsupported Media Type` (invalid extension), or `422 Unprocessable Entity` (malformed form data).
3. **Query dispatch.** `services/ocr_service.py` selects the preset query list for the endpoint (or the caller-supplied list) and forwards the raw bytes to Azure AI Document Intelligence with `model_id = prebuilt-layout` (`ocr_service.py:116`) and `features = QUERY_FIELDS` (`ocr_service.py:118`).
4. **Cloud layout analysis.** Azure analyzes layout geometry and resolves each natural-language query field against the document, returning string values.
5. **Bilingual enrichment.** For designated fields (holder name, place of issue, nationality, occupation, employer, owner, company name, permitted vehicle classes — grouped by the document as "key demographic, geographic, and organizational values") `services/translation_service.py` calls Azure AI Translation (default source `en`, default target `ar`, both at `translation_service.py:16`; region `centralus`, `config.py:19`), fronted by an in-memory `@lru_cache(maxsize=1024)` with graceful exception fallback. The document's introduction describes the enrichment as "bidirectional Arabic and English translations", but the only documented defaults are `en` -> `ar`; no `ar` -> `en` path is shown.
6. **Typed serialization.** Values are mapped into Pydantic v2 models (`DrivingLicenseFrontData`, `PermittedVehicle`, `TradeLicenseData`, etc.) and wrapped in `BaseResponse` (`message`, `messageType` `'S'`/`'E'`, `response`).
7. **Response / error.** `200 OK` with the envelope; cloud API failures surface as `500 Internal Server Error` through centralized exception handling.

**Data flow:** binary document bytes -> multipart stream (`UploadFile`) -> Azure Document Intelligence request with query strings -> resolved field strings -> translation cache/Azure Translator -> Pydantic model -> JSON envelope. The document describes the pipeline as stateless and synchronous; no persistence layer is mentioned (absence of persistence is inferred from "stateless").

**Integrations and dependencies:** Azure AI Document Intelligence (`prebuilt-layout`, `DocumentAnalysisFeature.QUERY_FIELDS`), Azure AI Translation (multi-service endpoint, default region `centralus`), plus the pinned Python stack listed in Section 4. FastAPI also exposes `GET /docs`, `GET /redoc`, and `GET /openapi.json` (OpenAPI 3.1.0).

**Flow line:**

`Client -> FastAPI (/api/v1, CORS, file validation) -> ocr_service (query preset) -> Azure Document Intelligence (prebuilt-layout + QUERY_FIELDS) -> translation_service (LRU cache -> Azure Translator en->ar) -> Pydantic schema -> BaseResponse JSON`

**Architecture Summary:** A stateless, synchronous ASGI microservice. The document's Figure 1 shows four components (Client Consumer, FastAPI Gateway, Azure Doc Intel, Bilingual Output); regrouped by responsibility (analyst framing) these are: gateway (routing, validation, CORS, exception handling), cognitive-service (Azure client wrappers and per-document query presets), enrichment (cached bilingual translation), and contract (Pydantic v2 schemas inside a fixed envelope). The key design bet is to put field semantics into natural-language queries executed by a foundation layout model rather than trained templates, so document variance is absorbed by the cloud model and new formats are served by the generic `/api/v1/extract/query` endpoint. The gateway is decoupled from the cognitive provider, which the document cites as enabling future substitution with local LLMs or on-premise OCR engines.

## 4. Technologies and Tools Used

| Category | Technologies (with pinned versions where stated) |
|---|---|
| Programming languages | Python (inferred from FastAPI/Pydantic/Jupyter stack; version not stated) |
| Frameworks | FastAPI 0.141.1; Pydantic 2.13.4; Pydantic-Settings 2.14.2 |
| Libraries | azure-ai-documentintelligence 1.0.2; azure-ai-translation-text 2.0.0; azure-core 1.36.1; python-multipart 0.0.32; pytest 9.1.1; httpx 0.28.1; `functools.lru_cache` (standard library) |
| AI/ML models | Azure Document Intelligence `prebuilt-layout` foundation model with `QUERY_FIELDS` feature; Azure AI Translation text service (underlying translation model not named in the document) |
| OCR tools | Azure AI Document Intelligence (query-based layout extraction) |
| Databases | Not stated in documentation (stateless service; no persistence) |
| Cloud platforms | Microsoft Azure (Document Intelligence; Translator multi-service endpoint, default region `centralus`) |
| APIs | REST API built on FastAPI: `GET /`, `GET /health`, 8 `POST /api/v1/extract/*` endpoints; Swagger UI `/docs`, ReDoc `/redoc`, OpenAPI 3.1.0 `/openapi.json`; Azure Document Intelligence API; Azure Translator API |
| DevOps tools | Not stated in documentation (health probe described as intended for container orchestrators and load balancers, but no container/CI tooling is named) |
| Version control | Not stated in documentation |
| Deployment tools | Uvicorn 0.52.0 ASGI server (default bind `0.0.0.0:8000`); no further deployment tooling stated |
| Document processing tools | Azure Document Intelligence; FastAPI `UploadFile` multipart ingestion parsed by python-multipart 0.0.32; supported inputs `.jpg`, `.jpeg`, `.png`, `.pdf`, `.tiff`, `.bmp` |
| Automation tools | Jupyter notebook (`workbook.ipynb`) for POC; pytest for automated tests |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Layered package structure (`main.py`, `routers/`, `services/`, `models/`, `config.py`); Pydantic v2 domain models; `BaseSettings` configuration with `.env` fallbacks; `lru_cache` memoization; centralized exception handling; explicit HTTP status semantics (400/415/422/500); pytest + httpx tooling.
- **AI Engineering:** Zero-shot use of a foundation layout model via natural-language query fields instead of custom extractors; per-document query preset design; notebook POC to production-hardened service; composing two Azure cognitive services into one pipeline.
- **Machine Learning:** Not demonstrated in this project beyond model selection (`prebuilt-layout`); no training, fine-tuning, or evaluation is described.
- **NLP:** Bilingual entity normalization (English -> Arabic) via Azure AI Translation for names, places, nationalities, occupations, employers, company names, and vehicle classes (the document's "key demographic, geographic, and organizational values"); semantic natural-language field querying over document text.
- **Computer Vision:** Document image/PDF layout analysis delegated to Azure Document Intelligence; handling of rotation, formatting shifts, and resolution degradation is attributed to the cloud model, not custom CV code.
- **Data Engineering:** Multipart binary stream ingestion with `UploadFile`; extension whitelisting and empty-file detection; mapping unstructured extraction output into fixed typed schemas per document class.
- **Cloud:** Azure SDK integration (`azure-ai-documentintelligence`, `azure-ai-translation-text`, `azure-core`); region configuration (`centralus`); multi-service Translator endpoint.
- **MLOps:** Not demonstrated in this project (no model versioning, monitoring, or evaluation pipeline is described; the recorded POC-to-production evolution table is the closest artifact).
- **API Development:** RESTful design under a versioned `/api/v1` namespace; consistent `multipart/form-data` contract; standard response envelope (`message`, `messageType`, `response`); health/liveness endpoints; auto-generated OpenAPI 3.1.0 docs; CORS middleware; async lifespan management.
- **Prompt Engineering:** Query-field design — choosing field names such as `Main_Licence_No`, `Company_Name`, `Register_No`, `Phone_No`, `Mobile_No` that the layout model resolved in the recorded notebook run (all 8 trade-license fields extracted in Cell 3; no broader reliability data is recorded); supporting caller-defined query lists for arbitrary documents.
- **System Design:** Stateless synchronous pipeline; provider decoupling to allow future local LLM / on-prem OCR substitution; in-memory caching to cut translation round-trips and cloud cost; type-safe contracts to protect downstream insurance core systems.

## 6. Detailed Technical Contributions

**Features implemented**

- Ten declared routes: `GET /` (service identity: `service`, `version` "1.0.0", `status` "healthy", `docs` "/docs"), `GET /health` (`status` "UP"), and eight extraction endpoints:
  - `POST /api/v1/extract/driving-license/front` -> `license_authority_id`, `license_number`, `license_issue_date`, `license_expire_date`, `date_of_birth`, `issue_place_english/arabic`, `name_english/arabic`, `nationality_english/arabic` (11 fields).
  - `POST /api/v1/extract/driving-license/back` -> `traffic_code`, `transmission_type`, `permitted_vehicle` (`{english: [...], arabic: [...]}` via typed `PermittedVehicle` model).
  - `POST /api/v1/extract/resident-id/front` -> `id_number`, `name_english/arabic`, `date_of_birth`, `nationality_english/arabic`, `issue_date`, `expiry_date`, `sex`.
  - `POST /api/v1/extract/resident-id/back` -> `card_number`, `occupation_english/arabic`, `employer_english/arabic`, `issuing_place_english/arabic`, `family_sponsor`.
  - `POST /api/v1/extract/vehicle/front` -> `traffic_plate_no`, `place_of_issue_english/arabic`, `tc_no`, `owner_english/arabic`, `nationality_english/arabic`, `expiry_date`, `registration_date`, `insurance_expiry_date`, `policy_no`.
  - `POST /api/v1/extract/vehicle/back` -> `model`, `number_of_passengers`, `origin`, `vehicle_type`, `gvw`, `engine_number`, `chassis_number`, `empty_weight`.
  - `POST /api/v1/extract/trade-license` -> `main_licence_no`, `company_name_english/arabic`, `issue_date`, `expiry_date`, `register_no`, `email`, `phone_no`, `mobile_no`.
  - `POST /api/v1/extract/query` -> `extracted_fields` map, `query_count`, `matched_count`.
- Enterprise envelope on all business endpoints: `message`, `messageType` (`'S'` success / `'E'` error), typed `response`.
- Strict upload validation: file presence, non-zero length, whitelist `.jpg .jpeg .png .pdf .tiff .bmp`.
- CORS middleware and async lifespan events mounted in `main.py`; centralized exception handling in the gateway layer (`main.py`, `routers/`); extraction routers bound under `/api/v1` at `main.py:47-52`.
- The trade-license sample response in the API reference reproduces the values recorded in notebook Cell 3 (`main_licence_no` "120278", `register_no` "1014241", the same company name), so the documented contract is traceable to a real extraction run.

**Models used**

- Azure Document Intelligence `prebuilt-layout` with `DocumentAnalysisFeature.QUERY_FIELDS` (no custom-trained model).
- Azure AI Translation text service (default `en` -> `ar`).

**Pipelines built**

- Stateless synchronous request pipeline: validate -> query Azure DI -> translate -> serialize -> envelope.
- POC pipeline in `workbook.ipynb`: direct initialization of the Azure Document Intelligence client (`prebuilt-layout`, `DocumentAnalysisFeature.QUERY_FIELDS`) and Azure Text Translation client plus definition of the seven query lists (Cells 1-2); trade license test on `Dubai - Sample 1.pdf` with 8 query fields `Main_Licence_No`, `Company_Name`, `Issue_Date`, `Expiry_Date`, `Register_No`, `Email`, `Phone_No`, `Mobile_No` (Cell 3, execution count 20, all 8 extracted; recorded output `Company_Name` 'STRONG PLANT FOR DEWATERING & DRAINAGE SERVICES (L.L.C)', `Main_Licence_No` '120278', `Register_No` '1014241'); driving-license front mapping (`DL_front`) with `to_arabic` translation on place of issue and nationality (Cell 4, execution count 15; recorded output `issue_place_english` 'Dubai' and `nationality_english` 'Philippines' with their Arabic translations); driving-license back skeleton (`DL_back`, Cell 5, null execution count — no stored output).

**APIs integrated**

- Azure Document Intelligence SDK (`azure-ai-documentintelligence` 1.0.2) via `services/azure_client.py`.
- Azure Translator SDK (`azure-ai-translation-text` 2.0.0), multi-service endpoint, region `centralus` (`config.py:19`).

**Data extraction methods**

- Zero-shot semantic layout queries: natural-language field names passed as query fields; the model resolves them against layout geometry, avoiding template coordinates.
- Preset query lists per document side (`df`, `db`, `rf`, `rb`, `vf`, `vb`, `tr`) in `services/ocr_service.py`; ad-hoc lists for `/query`.

**Validation logic**

- HTTP-layer: 400 for missing/empty file or empty query list; 415 for non-whitelisted extension; 422 for malformed form data; 500 for cloud failure.
- Schema-layer: "Strict Pydantic response models guarantee uniform JSON contracts across all document classes"; `BaseResponse` wraps every business payload. (The document does not state that fields are required — notebook Cell 4 recorded null license fields — so field-presence enforcement is not confirmed.)

**Automation workflows**

- Single-call extraction of a full document side (up to 11 fields) replacing manual transcription; pytest/httpx automated test harness (details not described).

**Optimization techniques**

- `@lru_cache(maxsize=1024)` on translation calls (`translation_service.py:16`) so high-frequency tokens (emirate names, issuing authorities, common nationalities) are translated once per process.
- "Graceful exception fallbacks" around translation (the document's wording); that this prevents a translator outage from failing the whole extraction is inferred — the fallback behaviour is not specified.
- Stream-based ingestion (`UploadFile`) rather than disk paths.

**Performance improvements**

- Qualitative only: fewer translator round-trips and lower cloud consumption from caching; the document records no latency, throughput, or accuracy measurements.

## 7. Challenges and Solutions

1. **Template brittleness of conventional OCR (technical).** Template/bounding-box extraction fails on rotation, formatting shifts, resolution loss, and redesigns. *Solution:* zero-shot `QUERY_FIELDS` on `prebuilt-layout`, so semantics live in query strings rather than coordinates. *Alternatives:* traditional template-based OCR and naive bounding-box coordinate scraping (explicitly contrasted in the document); custom-trained extraction models per document type (the document states the design avoids custom model training, annotation datasets, and weight redeployment); multimodal LLM vision extraction (analyst suggestion — the document only mentions future local LLM integration).

2. **Bilingual normalization for compliance (business).** Insurance core systems and KYC checks need consistent Arabic and English values. *Solution:* Azure AI Translation on selected fields, producing paired `*_english` / `*_arabic` attributes. *Alternatives:* curated lookup tables for bounded vocabularies such as emirates and nationalities, with MT only for open-ended names (analyst suggestion); reading the Arabic text already printed on the card via a second query set (analyst suggestion).

3. **Translation latency, cost, and connection drops (technical).** The notebook's raw translation calls "failed under high latency or connection drops". *Solution:* `@lru_cache(maxsize=1024)` plus graceful exception fallbacks. *Alternatives:* shared cache (Redis) across workers, retry with exponential backoff, batching all fields of a document into one translation call (analyst suggestions).

4. **Notebook-to-production drift (technical).** The POC loaded `.env` from an absolute path (`C:\...\5-3-2026-azure+llm\.env`), read files via `open(path, 'rb')`, and returned unstructured dicts. *Solution:* Pydantic `BaseSettings` with environment fallbacks, `UploadFile` streams with whitelist/empty-file checks, and rigid Pydantic v2 schemas inside `BaseResponse`. *Alternatives:* wrapping the notebook code in a thin CLI, or a plain `.env` loader with untyped dicts (analyst suggestions; the document records only the chosen refactor).

5. **New document types without retraining (business).** Emirate variations and new formats arrive faster than model retraining cycles. *Solution:* `/api/v1/extract/query` accepts arbitrary query lists and reports `query_count` / `matched_count`. *Alternatives:* per-type custom models (rejected implicitly by the zero-shot design).

6. **Untrusted uploads (technical).** Callers submit arbitrary binary payloads over multipart HTTP; missing, empty, or non-document files must be rejected before a cloud call is spent. *Solution:* existence, non-zero length, and extension whitelisting with explicit 400/415/422 semantics. *Alternative:* content-sniffing MIME validation and size caps (analyst suggestion).

**Documented limitations and bugs (honest talking points):**

- Notebook Cell 4 returned null license fields "because trade license data was piped" — a POC test-wiring bug. The document says the cell was "carried into production" as the driving-license/front endpoint but does not explicitly state the wiring was corrected (inferred from the per-endpoint routing design).
- Notebook Cell 5 (driving-license back) was never executed and used static permitted-vehicle classes ("Light Vehicle"); production adds a typed `PermittedVehicle` model, but the document does not confirm dynamic extraction of classes.
- No quantified accuracy, throughput, or ROI evidence exists.
- `lru_cache` is process-local, so cache benefit does not span multiple Uvicorn workers (analyst observation).
- No authentication, rate limiting, or persistence is described; dates are returned as raw strings without normalization.
- The endpoint summary table lists 8 rows while the text describes 10 routes — a minor documentation inconsistency (the trade-license and query endpoints are detailed in the text but absent from the table). The "Swagger UI Version" subsection is empty.
- The document describes the gateway as performing "MIME type filtering" / validating "MIME extensions", but the only concrete check documented is the extension whitelist (415 is raised for "invalid extension"); content-based MIME sniffing is not evidenced (inferred).
- The introduction claims "bidirectional" Arabic/English translation, yet the only documented defaults are `en` -> `ar` (`translation_service.py:16`); no Arabic-source path is shown.
- The document calls the service "high-accuracy" without recording any accuracy figure, and the repository has no user persona documentation, access control rosters, or role declarations (both stated in the document).

## 8. Impact Analysis

- **Business impact:** Typed, bilingual identity/vehicle/commercial data feeds onboarding, underwriting, claims, and third-party verification; supports UAE Central Bank and KYC mandates. The document claims cycle times move "from hours or days to sub-second automated verification" — a stated strategic benefit, not a measured result. A third stated strategic benefit is "Architectural Agility & Modularity": the REST gateway is decoupled from the cognitive cloud providers, which the document says enables straightforward future integration with local LLMs or on-premise OCR engines.
- **Productivity improvements:** Manual transcription replaced by a single API call returning "up to 11 discrete structured fields per document side"; new document formats served immediately via `/query` without retraining, annotation, or redeployment.
- **Accuracy improvements:** Not quantified. The document states no OCR character accuracy rates are recorded; qualitatively, deterministic translation removes spelling variation in identity values, and typed schemas prevent downstream ingestion errors.
- **Cost savings:** Not quantified. Qualitatively, the 1024-entry LRU cache reduces redundant translator calls and cloud consumption, and zero-shot extraction avoids model training/annotation costs.
- **Time savings:** Not quantified beyond the unmeasured "hours or days to sub-second" claim.
- **User benefits:** Uniform JSON contract across seven fixed document-side endpoints (four document types) plus a generic endpoint; explicit error codes; self-documenting Swagger/ReDoc; `GET /health` liveness probe for container orchestrators and load balancers; `GET /` service identity with a `/docs` pointer.

The document's "Quantified Evidence Disclaimer" is explicit: no ROI percentages, human-hour cost reduction figures, OCR character accuracy rates, or throughput statistics are recorded.

## 9. Interview Discussion Points

- Why zero-shot query fields over a custom-trained Document Intelligence model: no annotation datasets, resilience to redesigns, and instant coverage of new formats via `/query`; trade-off is dependence on cloud model quality with no in-house accuracy baseline.
- How the query presets (`df`, `db`, `rf`, `rb`, `vf`, `vb`, `tr`) were derived and validated in `workbook.ipynb`, including the Cell 4 wiring bug and what changed when moving to routers.
- Justify `@lru_cache(maxsize=1024)`: which tokens repeat (emirates, authorities, nationalities), why 1024, and the limitation that the cache is per-process and lost on restart.
- Machine-translating proper nouns (person and company names) — expected fidelity issues, and how a lookup table or reading printed Arabic could improve it.
- Error taxonomy: 400 vs 415 vs 422 vs 500, and why `messageType 'S'/'E'` exists alongside HTTP status codes for downstream insurance systems.
- The notebook-to-production refactor: config decoupling to `BaseSettings`, `UploadFile` streaming, typed `BaseResponse`, and graceful translator fallback.
- Synchronous vs asynchronous design: the pipeline is synchronous and stateless; how you would handle long Azure analysis times or bursty upload volumes (queueing, async job endpoints).
- Security gaps: extension-only validation, no auth, no size caps — what you would add before exposing this to third-party verification services.
- How `query_count` / `matched_count` on the dynamic endpoint lets callers detect partial extraction, and why the fixed endpoints do not expose per-field confidence.
- Testing approach with pytest and httpx, and how you would build an accuracy benchmark given the absence of recorded metrics.
- PII handling (Emirates ID numbers, sponsor details) in a stateless service that still transits data through two cloud services.
- Why the generic `prebuilt-layout` model plus `QUERY_FIELDS` rather than a document-specific prebuilt identity model: the document records no such comparison, so present it as your own reasoning (analyst suggestion) — one code path covers all seven document sides and arbitrary documents, at the cost of no built-in field confidence or document-type validation.
- Traceability from experiment to contract: the trade-license sample response in the API reference reproduces the values from notebook Cell 3 (execution count 20), so you can walk an interviewer from the recorded POC run to the `TradeLicenseData` schema.
- The "bidirectional" translation claim versus the documented `en` -> `ar` defaults: how you would support Arabic-source documents or verify that the translated Arabic matches the Arabic printed on the card.

## 10. Architecture Explanation Points

Whiteboard narrative (about two minutes):

1. **Components.** Draw four boxes left to right: Client Consumer (underwriting platform / onboarding portal / mobile app), FastAPI Gateway (`main.py`, `routers/`, `/api/v1`, CORS, validation, exception handling), Azure Document Intelligence (`prebuilt-layout` + `QUERY_FIELDS`), and Bilingual Output layer (`ocr_service.py`, `translation_service.py`, `schemas.py`). Note Azure Translator hanging off the output layer with an LRU cache (1024) in front.
2. **Data flow.** A multipart `file` arrives; the gateway checks presence, non-zero size, and extension whitelist; bytes plus a preset query list go to Azure; resolved strings come back; selected fields are translated `en` -> `ar` (cache hit or translator call); everything is bound into a Pydantic model and wrapped in `{message, messageType, response}`. Stateless, synchronous, no database.
3. **Key design decisions.** Semantics in query strings, not templates, so document variance is absorbed by the foundation model; a generic `/query` endpoint for new formats; typed contracts to protect insurance core systems; caching to cut translator cost and latency; provider boundary in `azure_client.py` for future substitution.
4. **Trade-offs.** Cloud dependency and PII transit through two external services; no measured accuracy; process-local cache; synchronous request holding while Azure analyzes; extension-based rather than content-based validation; machine-translated proper nouns.
5. **What to improve next.** Per-document-type accuracy benchmark; shared cache (Redis) with retry/backoff; authentication and size limits; content-based MIME validation to back the document's "MIME type filtering" wording; date normalization and per-field confidence; an Arabic-source (`ar` -> `en`) path to make the "bidirectional" claim true; async job mode for batch ingestion; observability to produce the quantified evidence the document says is missing. (All analyst suggestions — the document names only the local-LLM / on-premise OCR substitution as future work.)
