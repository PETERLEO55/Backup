# Project 3: SLM AI OCR (FastAPI / YOLOv8 / Azure Document Intelligence / Qwen LLM)

Source: `5056_AI_OCR_Documentation.md` (Document no. AT/slm-ocr/V1, dated 01-Sep-2026). Client: Qatar Insurance Group.

## 1. Project Overview

- **Project name:** SLM AI OCR (`SLM_AI_OCR`) — an automated document-processing microservice for UAE identity and vehicle documents.
- **Business domain:** Insurance (Qatar Insurance Group). Consuming systems named in the document: automated insurance underwriting platforms, retail motor claims systems, digital customer onboarding workflows, and fleet management portals ingesting UAE identity and vehicle cards. The architecture table names the calling client as an "underwriting portal, claims management system, or mobile app backend". The `SLM_AI_OCR_Project_Specification.docx` classifies the project as a "Management Overview for UAE Document Ingestion"; individual end-user roles, operator personas and admin hierarchies are "Not specified in the project."
- **Problem statement (inferred from the document's Business Benefits section):** Underwriting, claims and onboarding processes depend on data printed on UAE Driving Licenses, Emirates ID / Residence Cards and Vehicle Mulkiya (registration) cards issued across different Emirates and regulatory authorities. Manual rekeying of these credentials is error-prone, documents arrive as loose images or multi-page PDFs containing several cards, wrong document types get submitted, and UAE regulatory and localized reporting requires bilingual (English/Arabic) values. There was no centralized, standardized service abstraction across these document types.
- **Project objective:** Build a stateless, asynchronous FastAPI microservice that ingests a single image or multi-page PDF, localizes and crops each card with a trained YOLOv8 model, runs Azure AI Document Intelligence (`prebuilt-read`) OCR, validates document type against configured anchor tokens, reconstructs reading order, extracts fields with a locally hosted Qwen model under strict JSON-schema constraints (via Ollama's OpenAI-compatible endpoint), and returns normalized snake_case JSON enriched with automated Arabic translations.
- **Expected business outcome:** Elimination of clerical transcription errors, automatic multi-page separation without operator cropping, bilingual enrichment without human translators, fail-fast rejection of misclassified documents before they reach enterprise databases, lower cloud token cost and better PII sovereignty from the hybrid cloud-OCR / self-hosted-LLM design, and centralized credential management via HashiCorp Vault. The document also frames the design as an "Open-source approach": a small language model that "adapts to CPU bound" hardware so that "the extraction may happen with good accuracy within 10 secs timeframe" (stated as intent, not a measurement). ROI, cost-saving percentages, accuracy rates and throughput are explicitly "Not specified in the project."

## 2. Role and Responsibilities

**Core responsibilities (inferred from the delivered system):**

- Designed the seven-phase stateless pipeline (Section 3) and its per-request memory-purging execution model.
- Built the FastAPI service (`main.py`): CORS, HTTP Basic Authentication against Vault secrets, `log_requests` latency middleware, and a `create_route` factory registering six POST routes under the `/uae` APIRouter plus `GET /api/test`.
- Implemented the CV splitter (`utils/yolo_splitter.py`): pypdfium2 rasterization at `scale=2`, OpenCV decoding, YOLOv8 inference with `yolo-multipage-OCR-cls.pt`, bounding-box cropping, and diversion of non-documents to `skipped_images/`.
- Integrated Azure AI Document Intelligence `prebuilt-read` (`logic/ocrllm.py`) for line text plus polygon coordinates, with `HttpResponseError` handling.
- Designed the `DOCUMENT_ANCHORS` multi-token lists and anchor-scoring validation for all six document types, and wrote the `combine_text_lines` clustering algorithm (10-pixel vertical tolerance, horizontal word sort).
- Engineered LLM extraction: pipe-delimited field schemas (`OUTPUT_DICT`, `logic/ocrllm.py:18-25`), `/no_think` prompt control, `temperature=0`, strict `json_schema` response format, via `AsyncOpenAI` against Ollama.
- Built the output mapper (`utils/mapping_output.py`): snake_case normalization and English-to-Arabic derivation with `GoogleTranslator` under a 3-second thread timeout and `max_retries=0`.

**Supporting tasks (inferred):**

- Secrets integration with HashiCorp Vault via `hvac` KV v2 (`utils/vault_credentials.py`, `retail-dev` path).
- Observability: daily-rotating `OCR_API.log` with 24-day retention, plus `gc.collect()` after each request.
- Standardized error payload contract (`{"message": ..., "messageType": "E"}`); dependency pinning in `requirements.txt` and a `.venv`.
- Integration test runs recorded in `log_databases/ocr_api_logs/OCR_API.log` (March 31, 2026) capturing per-endpoint latency.
- Possible contribution to this technical documentation and the `SLM_AI_OCR_Project_Specification.docx` management overview (inferred; authorship is not stated anywhere in the document).
- YOLOv8 model training is implied by the "trained PyTorch YOLOv8 model" wording; data and procedure are not documented.

**Estimated ownership level: Senior AI Engineer (inferred).**
Rationale: the document describes an architecturally broad system: a custom-trained CV model, a cloud OCR integration, a self-hosted LLM with constrained decoding, a bespoke spatial-clustering algorithm, a timeout-isolated translation step, Vault-backed authentication, and production concerns (rotating audit logs, latency profiling, memory hygiene, standardized error contracts, six endpoints). The hybrid cloud/edge decision (Azure OCR plus on-prem Qwen for PII sovereignty and token cost) is an architectural trade-off, not just implementation. No team leadership, CI/CD, containerization or automated tests are evidenced, so "Solution Architect" or "Technical Lead" would overstate what the document supports.

## 3. End-to-End Workflow

**Processing stages:**

1. **Request ingestion & authentication** — Client POSTs `multipart/form-data` with HTTP Basic Authentication to one of the six `/uae/*` routes. Credentials are checked against Vault secrets; exactly one file is required (`len(file) == 1`, `main.py:53`).
2. **Rendering & decoding** — Images (JPEG, PNG, etc.) are decoded from the byte buffer with OpenCV; PDFs are rendered page-by-page with pypdfium2 at `scale=2`.
3. **YOLOv8 localization & cropping** — `yolo-multipage-OCR-cls.pt` (loaded from `./utils/yolo-multipage-OCR-cls.pt`, `utils/yolo_splitter.py:12`) detects document boundaries, crops distinct cards, isolates document classes from extraneous content, and diverts non-document images to `skipped_images/` (directory constant defined at `logic/ocrllm.py:161`). A decode/parse failure in `yolo_split` returns the "Unsupported file type" error; an empty flattened crop list returns "No cropped images found."
4. **Azure OCR** — Each crop is sent to Azure AI Document Intelligence `prebuilt-read` for line text and polygon coordinates.
5. **Anchor validation & reading-order reconstruction** — OCR tokens are scored against `DOCUMENT_ANCHORS` for the targeted type; mismatches are rejected before any LLM call. Valid documents pass through `combine_text_lines`.
6. **Schema-constrained LLM extraction** — Ordered text plus the document type's pipe-delimited field list go to the local Qwen model (`model="qwen"`, `temperature=0`, `/no_think`, strict `json_schema`) via `AsyncOpenAI` at `http://172.20.132.97:11434/v1`.
7. **Normalization, translation & delivery** — `utils/mapping_output.py` maps raw keys to snake_case and derives Arabic (`target='ar'`) values under a 3-second timeout; JSON is returned, `log_requests` records latency (described as "millisecond-accurate timing") and request metadata, then runs `gc.collect()`. If the results list contains no valid documents, the route raises HTTP 400 `{"detail": "No valid documents found"}`.

**Data flow:** multipart binary file -> NumPy image arrays (OpenCV decode / pypdfium2 render) -> cropped card arrays -> Azure line tokens + polygons -> ordered text string -> schema-constrained LLM JSON -> normalized bilingual JSON dict -> HTTP JSON response; side channels: `skipped_images/` on disk and `OCR_API.log`.

**Integrations and dependencies:** Azure AI Document Intelligence (`prebuilt-read`); Ollama serving Qwen behind an OpenAI-compatible API consumed with `AsyncOpenAI`; Ultralytics YOLOv8 with local PyTorch weights; OpenCV and pypdfium2 for decoding and rendering; deep-translator (GoogleTranslator); hvac against HashiCorp Vault; Requests, python-multipart, Pydantic and Uvicorn. Pinned versions are in Section 4.

**Flow line:**
`Client -> FastAPI /uae/<doc_type> (Basic Auth via Vault) -> yolo_split (pypdfium2 / OpenCV -> YOLOv8 crop) -> Azure DI prebuilt-read -> DOCUMENT_ANCHORS check -> combine_text_lines -> Ollama Qwen (AsyncOpenAI, json_schema, temp=0) -> mapping_output (snake_case + GoogleTranslator ar) -> JSON`

**Architecture Summary:** A stateless asynchronous linear pipeline behind a single FastAPI process (Uvicorn on port 5055). Perception is split between an on-box CV model that decides *where* documents are and a cloud OCR model that decides *what text* is present; understanding is delegated to a self-hosted small language model forced into deterministic, schema-valid JSON; enrichment is a bounded-latency translation step. Cross-cutting concerns (Vault auth, CORS, rotating latency logs, garbage collection) live in middleware. Document-type specificity is data (anchor lists and pipe-delimited schemas keyed by endpoint), so six endpoints come from one `create_route` factory.

## 4. Technologies and Tools Used

| Category | Technologies (with pinned versions where documented) |
|---|---|
| Programming languages | Python |
| Frameworks | FastAPI ==0.114.0; Uvicorn ==0.30.6 (ASGI server); Ultralytics YOLOv8 ==8.4.14 |
| Libraries | OpenCV (opencv-python) ==4.11.0.86; NumPy ==2.2.6 (declared; 2.5.2 installed in .venv); PyPDFium2 ==4.30.0; OpenAI Python SDK (AsyncOpenAI) ==2.26.0; deep-translator ==1.11.4 (GoogleTranslator); hvac ==2.4.0; Requests ==2.32.3; python-multipart ==0.0.22; Pydantic ==2.10.6; PyTorch (YOLO weights); asyncio; logging (`TimedRotatingFileHandler`); gc |
| AI/ML models | YOLOv8 custom weights `yolo-multipage-OCR-cls.pt`; Azure Document Intelligence `prebuilt-read`; Qwen (Ollama model id `qwen`, version not specified) |
| OCR tools | Azure AI Document Intelligence ==1.0.2 (`prebuilt-read`) |
| Databases | Not stated in documentation (service is stateless; only file-based logs under `log_databases/`) |
| Cloud platforms | Microsoft Azure (AI Document Intelligence); self-hosted Ollama endpoint `http://172.20.132.97:11434/v1` |
| APIs | REST: `GET /api/test`, six `POST /uae/*` routes; FastAPI auto-docs `/docs` (Swagger UI), `/redoc` (ReDoc), `/openapi.json` (OpenAPI 3.0 specification); Azure Document Intelligence API; Ollama OpenAI-compatible API; Google Translate via deep-translator; HashiCorp Vault KV v2 API |
| DevOps tools | HashiCorp Vault (secrets); Python `.venv`; `.env` file (`port = 5055` at `.env:3`); `requirements.txt`. No CI/CD or containerization stated in documentation |
| Version control | Not stated in documentation (document mentions avoiding credentials "in source control" but names no VCS) |
| Deployment tools | Uvicorn (`uvicorn main:app --host 0.0.0.0 --port 5055`, the verified command); `python main.py` only after setting the `PORT` env var (`set PORT=5055` or `$env:PORT='5055'`) |
| Document processing tools | pypdfium2 (PDF rasterization, scale=2); OpenCV (image decoding); YOLOv8 (card cropping); Azure Document Intelligence (OCR) |
| Automation tools | Custom FastAPI middleware (`log_requests`), `TimedRotatingFileHandler` daily rotation; no external workflow/automation tool stated in documentation |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Modular layout (`main.py`, `utils/`, `logic/`, `log_databases/`); `create_route` factory generating six endpoints from data; standardized error contract; pinned dependencies; middleware for cross-cutting concerns; `gc.collect()` memory hygiene.
- **AI Engineering:** CV + cloud OCR + local LLM + MT composed into one deterministic pipeline; constrained decoding via `json_schema` at `temperature=0`; `/no_think` control; fail-fast anchor gate ahead of the LLM.
- **Machine Learning:** Deployment of a trained YOLOv8 detector (`yolo-multipage-OCR-cls.pt`) with Ultralytics 8.4.14. Training methodology not documented.
- **NLP:** Structured entity extraction from noisy OCR text with an SLM; pipe-delimited schema-to-field mapping; snake_case normalization; English-to-Arabic translation of name, place, nationality, occupation and employer fields.
- **Computer Vision:** PDF rasterization at 2x; OpenCV buffer decoding; YOLOv8 detection and cropping; non-document filtering to `skipped_images/`; polygon-driven line clustering (10-px vertical tolerance, horizontal sort).
- **Data Engineering:** Multi-format ingestion (JPEG/PNG/PDF); flattening multi-page, multi-card crops into a results list; JSON normalization; rotating logs with 24-day retention.
- **Cloud:** Azure AI Document Intelligence SDK integration with `HttpResponseError` handling; hybrid cloud/on-prem rationale (token cost, PII sovereignty).
- **MLOps:** Local weight management (`./utils/yolo-multipage-OCR-cls.pt`); self-hosted LLM serving via Ollama; per-endpoint latency profiling in `OCR_API.log`; version pinning. No model registry, CI/CD or monitoring stack documented.
- **API Development:** FastAPI + Uvicorn; `APIRouter` with `/uae` prefix; multipart uploads; HTTP Basic Auth; CORS; auto-generated OpenAPI/Swagger docs; health endpoint.
- **Prompt Engineering:** `/no_think` system prompt; extraction prompts built from per-type pipe-delimited field lists; strict JSON schema response_format; zero-temperature determinism.
- **System Design:** Stateless linear pipeline separating localization (edge CV), recognition (cloud OCR) and understanding (local SLM); data-driven type configuration (`DOCUMENT_ANCHORS`, `OUTPUT_DICT`); bounded external calls (3-second translation timeout, `max_retries=0`); secrets delegated to Vault.

## 6. Detailed Technical Contributions

**Features implemented**
- `GET /api/test` health check returning the string `'cloud vision data'`.
- Six POST extraction endpoints under `/uae`: `license_front` (identity and license attributes), `license_back` (traffic code, transmission type, permitted vehicle categories), `residence_front` (Emirates ID Number, demographics, validity dates), `residence_back` (Card Number, occupation, employer, issuing place, demographics), `vehicle_front` (ownership, traffic code, plate number, registration dates, insurance policy details), `vehicle_back` (technical specs, model year, origin, chassis number, engine number, gross weight).
- Cross-cutting: Vault-backed Basic Auth; `CORSMiddleware` configured with `allow_origins=['*']`, `allow_credentials=True`, `allow_methods=['*']`, `allow_headers=['*']`; request logging and per-request garbage collection.
- Framework-provided routes: `GET /docs` (Swagger UI), `GET /redoc` (ReDoc), `GET /openapi.json` (OpenAPI 3.0 specification).

**Models used**
- YOLOv8 (Ultralytics 8.4.14) with custom PyTorch weights `./utils/yolo-multipage-OCR-cls.pt` (`utils/yolo_splitter.py:12`).
- Azure Document Intelligence `prebuilt-read` (`logic/ocrllm.py:107`).
- Qwen served by Ollama, model id `qwen`, `temperature=0` (`logic/ocrllm.py:137`), endpoint `http://172.20.132.97:11434/v1`, consumed via `AsyncOpenAI`.
- GoogleTranslator (deep-translator 1.11.4), target `'ar'` (`utils/mapping_output.py:103`).

**Pipelines built**
- Seven-phase asynchronous pipeline described in Section 3; document-type-specific behaviour is driven by `DOCUMENT_ANCHORS` and `OUTPUT_DICT` (`logic/ocrllm.py:18-25`) rather than per-type code.

**APIs integrated**
- Azure AI Document Intelligence (SDK 1.0.2) with `HttpResponseError` capture.
- Ollama OpenAI-compatible chat completions with `response_format` `json_schema` (strict = True, `logic/ocrllm.py:139`).
- Google Translate via deep-translator over a Requests session with `HTTPAdapter(max_retries=0)` (`utils/mapping_output.py:109`).
- HashiCorp Vault KV v2 via hvac (`VAULT_BASE=retail`, `APP_OCR_ENV=retail-dev`, fallback `APP_VAULT_URL=https://hctest.anoudapps.com`; `utils/vault_credentials.py:5-9`).

**Data extraction methods**
- OCR line tokens + polygons -> `combine_text_lines` agglomerative vertical clustering at a 10-pixel threshold (`logic/ocrllm.py:41`), then horizontal word sorting -> ordered text.
- LLM extraction against pipe-delimited field definitions, zero temperature, `/no_think` system prompt (`logic/ocrllm.py:89, 99`), strict JSON schema enforcement.
- Post-extraction mapping of raw keys into snake_case schema fields (`utils/mapping_output.py:6-100`).

**Validation logic**
- File count: `len(file) == 1` (`main.py:53`) else `{"message": "Please attach at least one file/document to proceed...!", "messageType": "E"}`.
- Decode/parse failure: `{"message": "Unsupported file type:  <filename>", "messageType": "E"}`.
- Empty crop list: `{"message": "No cropped images found.", "messageType": "E"}`.
- Azure dimension error: `{"message": "The input image dimensions are out of range. ...: <error_msg>", "messageType": "E"}`.
- Anchor mismatch: `{"message": "Provide image is not <detected_doc>, Please provide valid image of type <doc_type>", "messageType": "E"}` — evaluated before LLM inference.
- No valid results: HTTP 400 `{"detail": "No valid documents found"}`.

**Automation workflows**
- `log_requests` middleware: latency in seconds ("millisecond-accurate timing"), client host, timestamp, method, path, status -> `OCR_API.log`; then `gc.collect()` to clear transient image matrices (`main.py:66-76`, `log_databases/logging_config.py:24-70`).
- `TimedRotatingFileHandler` with `when='midnight'`, `interval=1`, `backupCount=24` (`log_databases/logging_config.py:39-41`).
- Translation executed in a worker thread with `asyncio.wait_for` 3-second timeout (`utils/mapping_output.py:115`).

**Optimization techniques**
- Documented mechanisms: 2x PDF render scale producing "high-resolution image arrays"; YOLO cropping of card regions before OCR; anchor gate rejecting wrong documents "before invoking LLM extraction"; deterministic decoding (`temperature=0`, strict `json_schema`); `/no_think` which "disables internal reasoning preamble"; bounded translation latency (3 s, `max_retries=0`); explicit `gc.collect()` "to clear transient image matrices"; a small language model chosen to suit CPU-bound hardware ("Open-source approach").
- Rationale attributed to these mechanisms beyond the document's own wording (e.g., cropping limits Azure payload size, determinism avoids parse retries, `/no_think` reduces token count) is analyst inference.

**Performance improvements**
- No before/after comparison is documented. The integration test log (March 31, 2026) records end-to-end latencies between 5.07 s and 19.09 s across the six endpoints (per-endpoint ranges quoted verbatim in Section 8), against a stated design intent of extraction "with good accuracy within 10 secs timeframe" on CPU-bound hardware.

## 7. Challenges and Solutions

1. **Multi-page PDFs and mixed content (technical).** PDFs contain several cards plus unrelated pages. *Solution:* pypdfium2 rasterization at 2x, YOLOv8 cropping of each card, non-documents diverted to `skipped_images/`. *Alternatives:* fixed-grid splitting or classical contour detection (analyst suggestion), brittle against varied layouts.
2. **Fragmented OCR reading order (technical).** `prebuilt-read` returns line fragments with polygons rather than logically ordered text. *Solution:* `combine_text_lines` vertical clustering at 10 px plus horizontal sort. *Alternatives:* Azure `prebuilt-layout` reading order, or feeding polygons to the LLM (analyst suggestion).
3. **Wrong document uploaded to an endpoint (business).** Misfiled documents would yield plausible but wrong records. *Solution:* `DOCUMENT_ANCHORS` multi-token scoring rejects mismatches with a descriptive message before LLM inference. *Alternatives:* a YOLO classifier head or an LLM classification call (analyst suggestion); anchors are cheaper and deterministic.
4. **Unreliable LLM output format (technical; challenge inferred from the controls applied).** *Solution:* `temperature=0`, strict `json_schema` response_format, `/no_think`, schema from pipe-delimited field lists. *Alternatives:* regex/rule extraction, Azure custom extraction models, or function-calling with Pydantic validation (analyst suggestion).
5. **PII sovereignty and cloud token cost (business).** *Solution:* hybrid architecture: Azure performs OCR only; the Qwen SLM runs on a self-hosted Ollama endpoint. *Alternatives:* fully cloud (GPT/Gemini) or fully on-prem OCR such as Tesseract/PaddleOCR (analyst suggestion).
6. **Bilingual regulatory output (business).** *Solution:* GoogleTranslator derivation of five field families with a 3-second timeout and `max_retries=0` so translation cannot stall the request. *Alternatives:* LLM emits Arabic in the same pass, or Azure Translator (analyst suggestion).
7. **Credential hygiene (business).** *Solution:* hvac KV v2 reads from Vault (`retail-dev`); no plaintext secrets in the repository. *Alternatives:* deploy-time environment variables or a cloud KMS such as Azure Key Vault (analyst suggestion).
8. **Memory growth from image matrices (technical).** *Solution:* `gc.collect()` per request. *Alternative:* scoped context managers / explicit array deletion (analyst suggestion).
9. **Latency against the CPU-bound design intent (technical).** The document positions the SLM as one that "adapts to CPU bound" hardware with a "within 10 secs" intent, yet observed latencies in the March 31, 2026 test log reach 19.09 s on `residence_back` (the hardware used for that test run is not stated). Unresolved in the document; GPU serving, quantized Qwen variants or batched Azure calls are candidate fixes (analyst suggestion).

**Documented limitations and defects (honest talking points):**
- `python main.py` fails with `TypeError` because `main.py:81` evaluates `int(os.getenv('PORT'))` while `.env` defines lowercase `port = 5055`; the Uvicorn CLI command is the verified path.
- `utils/output_keys.py` is an orphaned schema module (superseded by `OUTPUT_DICT`) and contains a duplicated `'issue_place|issue_place'` field on `license_front`.
- NumPy version drift: 2.2.6 declared vs 2.5.2 installed in `.venv`.
- CORS is `allow_origins=['*']`, `allow_credentials=True`, `allow_methods=['*']`, `allow_headers=['*']` — permissive for a Basic-Auth service (analyst observation).
- One file per request (`len(file) == 1`); Azure rejects out-of-range image dimensions; the Ollama endpoint is an internal IP address (`http://172.20.132.97:11434/v1`; whether hard-coded or configured is not stated); Swagger UI and Ollama versions are "Not specified in the project" and the Qwen variant/size is not given.
- No Jupyter notebooks (`.ipynb`), scratch experiment scripts (`poc_*`) or draft prototypes were found in the repository; the only experimentation artifact is the orphaned `utils/output_keys.py`.

## 8. Impact Analysis

- **Business impact:** Provides a single standardized API for six UAE document faces consumed by underwriting, claims, onboarding and fleet systems; prevents misclassified documents from entering core systems; delegates credential lifecycle to Vault for compliance.
- **Productivity improvements:** Removes manual rekeying and manual PDF splitting/cropping; bilingual values generated without translators. No quantified productivity figure is given.
- **Accuracy improvements:** The document states extraction "may happen with good accuracy" but explicitly says overall extraction accuracy rates are "Not specified in the project." Qualitatively, determinism (`temperature=0`, JSON schema) and the anchor gate reduce format and misclassification errors.
- **Cost savings:** Percentage clerical cost savings and ROI are "Not specified in the project." Qualitatively, self-hosting the Qwen SLM reduces cloud token consumption relative to a cloud LLM.
- **Time savings:** Verbatim observed latencies per endpoint (March 31, 2026 test log): license_front 7.00–9.98 s; license_back 5.07–10.54 s; residence_front 8.27–13.66 s; residence_back 6.62–19.09 s; vehicle_front 7.71–8.86 s; vehicle_back 5.41–8.05 s. Throughput (documents per minute) is not specified.
- **User benefits:** Consuming systems receive normalized snake_case, bilingual JSON with descriptive, machine-readable error payloads (`messageType: "E"`), plus auto-generated Swagger/ReDoc documentation.

## 9. Interview Discussion Points

- Why split perception across a local YOLOv8 detector and Azure `prebuilt-read` rather than one OCR pass: cropping raises OCR quality, filters non-documents, and scopes Azure calls to card regions.
- How `combine_text_lines` works (10-px vertical agglomeration, then horizontal sort) and why polygon-based reading order matters before LLM extraction.
- The anchor-validation gate: how `DOCUMENT_ANCHORS` scoring works and why it runs before the LLM.
- Constrained decoding: `json_schema` strict mode, `temperature=0`, `/no_think`, pipe-delimited schemas per endpoint, and what happens when the SLM still hallucinates a value.
- Hybrid cloud/edge rationale: PII sovereignty and token cost drove a self-hosted Qwen via Ollama; the trade-off versus a stronger cloud LLM.
- Latency profile: 5.07–19.09 s observed in the March 31, 2026 test log against a design framed as CPU-bound with a 10-second intent (test hardware not stated); what dominates (Azure round-trip vs LLM decode vs translation) and how to get under 10 s consistently.
- The "Open-source approach": why a small open-weight model (Qwen via Ollama) and an open-source detector (YOLOv8) were chosen over a larger cloud LLM, and what accuracy you would expect to give up on CPU-bound hardware.
- The single-file-per-request contract (`len(file) == 1`) and multi-card PDF handling: how you would evolve the API to batch uploads or async job processing without breaking existing clients (analyst discussion point).
- Failure isolation for translation: 3-second thread timeout, `max_retries=0`, and why a non-critical enrichment step must never fail the request.
- Security posture: Vault KV v2 for Basic-Auth credentials versus wildcard CORS with credentials.
- The `PORT`/`.env` case mismatch bug and the orphaned `output_keys.py` with the duplicated `issue_place` field, and how you would fix them.
- Production hardening gaps (tests, CI/CD, containerization, model versioning, accuracy benchmarking) and the undocumented provenance of the YOLOv8 weights.

## 10. Architecture Explanation Points

Start with the contract: one image or PDF posted with Basic Auth to `/uae/<document_face>` returns normalized bilingual JSON or a structured error. Draw five boxes left to right. **FastAPI service** — middleware handles CORS, Vault-backed auth, per-request timing to a daily-rotating log, and `gc.collect()` on exit; one `create_route` factory produces all six endpoints. **YOLO splitter** — pypdfium2 renders PDF pages at 2x, OpenCV decodes images, a custom YOLOv8 model crops each card and dumps non-documents to `skipped_images/`. **Azure Document Intelligence** — `prebuilt-read` returns text lines with polygons; only cropped cards are sent. **Validation + ordering** — anchor tokens confirm the document type before any LLM cost, then 10-pixel vertical clustering rebuilds reading order. **Qwen via Ollama + mapper** — an OpenAI-compatible call at temperature 0 with a strict JSON schema from pipe-delimited field lists; keys normalized to snake_case, five field families translated to Arabic under a 3-second timeout.

Key decisions (stated in the document): perception on cloud, understanding on-prem (the document cites reduced cloud token consumption and PII data sovereignty); an "Open-source approach" with a small language model suited to CPU-bound hardware; determinism wherever the LLM touches output; document types as data, not code; every external call bounded or gated. Trade-offs (analyst assessment): observed 5.07–19.09 s latency versus a 10-second intent, dependence on Azure availability, a general-purpose translator instead of domain glossaries, no persistent store or queue, and permissive wildcard CORS. Improve next (analyst suggestions): GPU or quantized Qwen serving, async batching of Azure calls, a field-level accuracy benchmark, containerization plus CI/CD, tightened CORS, the `PORT`/`.env` config-key fix, retiring the orphaned schema file, and a YOLO-class-based document classifier where anchor heuristics prove brittle.
