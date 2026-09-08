# Project 15: VL_version_OCR — Vision-Language OCR (FastAPI / YOLOv8 / PyPDFium2 / LangChain / VLMs)

## 1. Project Overview

- **Project name:** VL_version_OCR (document no. AT/VLV/V1, dated 01-Sep-2026). Client: Qatar Insurance Group.
- **Business domain:** Insurance — motor insurance underwriting, claims intake, and policy administration. The system ingests GCC governmental identity documents (driving licenses, residency permits / civil IDs, vehicle registrations), international passports, and traffic police accident reports from four police authorities (Saaed Abu Dhabi, Dubai Police, Rafid Sharjah, Doha Traffic Police).
- **Problem statement:** Policyholder credentials arrive as composite scans and PDF bundles containing several ID cards per page, in mixed Arabic/English, across multiple jurisdictions with different layouts. Police accident reports arrive as vector PDFs or raster scans with jurisdiction-specific table structures and bidirectional Arabic text. Downstream underwriting and claims systems need structured, schema-valid JSON — not free text — and cannot tolerate generative hallucinations. Earlier experiments with Azure Document Intelligence (documented in debug.ipynb) "struggled with multi-document scans" and exhibited "variable latency and high cloud API coupling."
- **Project objective:** Build an end-to-end multimodal document intelligence pipeline that (a) detects and crops individual sub-documents from composite uploads with YOLOv8, (b) classifies each crop and extracts entities via a two-stage Vision-Language Model pipeline with strict JSON schema enforcement, (c) parses four governmental police accident report formats with dedicated PDF/OCR engines, and (d) exposes all of this through a single FastAPI service with pluggable model backends (local Qwen3-VL, Google Gemini, Azure Llama).
- **Expected business outcome:** Straight-through processing (STP) for policy onboarding and motor claims registration; elimination of manual pre-processing of composite scans; "zero-hallucination" structured ingestion for downstream systems; bilingual Arabic-English handling; high-throughput concurrency for multi-document pages and multi-page police reports; cross-border regional scalability into a single unified API architecture; and model-provider agility with an on-premise inference option for data sovereignty.
- **Target users (as stated in the source):** (1) downstream Qatar Insurance Group underwriting, claims-intake and policy-administration systems (client inferred by the documentation from the repository path and the `.env` service-account name `ocrqicuser`); (2) claim-operations and underwriting personnel processing motor claims, driver identity verification and liability assessment; (3) API integration clients — microservices and backend integrators calling the endpoints for batch OCR and automated ingestion. The source states that specific end-user personas, employee IDs and external client roles are "Not specified in the project."

## 2. Role and Responsibilities

**Core responsibilities (inferred from the system built):**
- Designed the decoupled pipeline architecture: FastAPI router → YOLOv8 splitter → unified VLM engine → JSON, plus a parallel police-report controller path.
- Applied a custom-trained YOLOv8 multi-document detection model (weights `yolo-multipage-OCR-cls.pt`, 54.5 MB, loaded at `services/yolo_splitter.py:8`; alternate checkpoint `best.pt`, 45.2 MB, referenced only in debug.ipynb) and tuned detection thresholds (conf=0.5, IoU=0.4) in debug.ipynb cells 116-138 before promoting them to `services/yolo_splitter.py:39-40`. (The source describes the weights as "trained" but never states who trained them — candidate authorship of the training run is *inferred* and unconfirmed.)
- Built the pluggable asynchronous VLM execution engine (`services/llm_engine.py`) supporting both native AsyncOpenAI and LangChain Core paradigms across Qwen3-VL-4B-Instruct (local vLLM), Gemini-3-Flash-Preview (Google GenAI), and Dev-Llama-4-Scout-17B-16E-Instruct (Azure AI Services).
- Designed the two-stage classification/extraction prompt pipeline: country-specific keyword anchor rules (`DOCUMENT_ANCHORS`) for document typing, and dynamic strict JSON schema synthesis from `OUTPUT_DICT` (strict=True, temperature=0, additionalProperties=False).
- Authored jurisdictional schema configurations for 18+ document types across UAE, Qatar/Doha, Oman, Kuwait, and six passport nationalities (`config/document_anchors.py`, `config/output_keys.py`).
- Built four police accident report parsers in `services/police_docs/` combining PyMuPDF vector table extraction, Google Cloud Vision OCR, Wand/ImageMagick rendering, and a custom 2D coordinate sorting algorithm.
- Implemented the REST API surface (`app.py`, `routers/extract.py`): `app = FastAPI()` with two mounted routers — `router = APIRouter(prefix='/extract', tags=['OCR'])` and `pd_router = APIRouter(tags=['Police_docs'])` — giving a health-check `GET /`, five `/extract/{country}` endpoints and four police endpoints, with model selection via a required enum query parameter, `asyncio.gather` concurrency, HTTP 400 semantics, and gzip-compressed JSON error responses.
- Authored the per-jurisdiction output field contracts, e.g. UAE (`license_number`, `date_of_birth`, `owner_name_english`, `traffic_plate_number`), Oman `driving_license_front` (`license_number`, `issued_place_english`, `name_english`, `first_issued`, `expiry_date`), Kuwait `residency_permit_front` (`civil_id`, `name`, `passport_number`, `nationality`, `date_of_birth`, `expiry_date`), Doha `residency_permit_front` (`idNo`, `dob`, `expiry`, `nationalityEng`, `nationalityAra`, `nameEng`, `name_ar`), passports (`type`, `code`, `passport_number`, `given_name`, `fullname`, `date_of_birth`, `date_of_expiry`) — including the deliberate Kuwait exception where vehicle registration uses a single `vehicle_registration` key rather than separate front/back keys (inferred).

**Supporting tasks (inferred):**
- Ran and recorded an experimentation program (debug.ipynb, 214 cells) covering Azure Document Intelligence custom and prebuilt-layout models, Qwen 2.5 / Qwen 0.6B local prompt engineering via Ollama/vLLM, YOLO parameter sweeps, and Gemini-3-Flash/Pro structured-output trials, and made the documented keep/abandon decisions.
- Integrated Arabic NLP (arabic_reshaper + python-bidi) and English translation (deep_translator GoogleTranslator) into the police parsers.
- Managed configuration and secrets (`.env` port 5057, police sub-service port 5055, Google service account `OCR_key.json`, service account `ocrqicuser`).
- Refactored duplicated Qwen prototypes (`qwen_langchain.py`, `qwen_openai.py`) into one `config_dict`-driven engine; archived superseded engines under `services/backup/`.
- Image pre-processing for VLM submission (1536x1536 thumbnail at `llm_engine.py:55`, JPEG quality 80 at `llm_engine.py:57`); system documentation with verified code line references.
- Disabled the archived Azure path rather than deleting it — `services/backup/azure_engine.py` (async `azure_kv_extract` against prebuilt-layout) remains an inactive archive and the AZURE engine is explicitly commented out at `routers/extract.py:20` (inferred).
- Assembled and retained the test-asset and artefact inventory: raster test PDFs and scanned police-report JPEGs under `skipped_images/` (`police_report_0..5.jpg`, 1183x1663 to 1191x126 px, produced as output of `multiple_extract`), the in-repo YOLO weights, `OCR_key.json` (referenced at `.env:14` and `ocr_engine.py:24`), an unreferenced `labels.json` (94-byte driving-license-front label dictionary), and `architecture_flow_bw.png` used by an earlier docx report (inferred).

**Estimated ownership level: Senior AI Engineer (inferred).**
Rationale: The candidate owned the full stack — a custom computer-vision detector (training authorship itself unconfirmed), a multi-provider VLM orchestration layer with two execution paradigms, four bespoke document parsers, Arabic NLP integration, and the API layer — and made architecture-level trade-offs (on-premise vLLM vs. cloud foundation models; single-pass VLM vs. OCR+regex; abandonment of Azure DI). The scope (nine functional endpoints, three model backends, 18+ schemas, two OCR engines) and the recorded experimental lineage indicate end-to-end design authority. It stops short of "Solution Architect / Technical Lead" because the document gives no evidence of team leadership, and production-readiness gaps remain: no automated tests, unpinned dependencies with version drift, and a documented critical key-mismatch defect on the UAE path.

## 3. End-to-End Workflow

**Path A — Identity document extraction (`/extract/{country}`):**
1. Client POSTs `multipart/form-data` field `images` (PDF/JPEG/PNG, one or more) with required query parameter `model_name` (enum: `LLAMA-LANGCHAIN`, `LLAMA-OPENAI`, `GEMINI-LANGCHAIN`, `GEMINI-OPENAI`, `QWEN3-LANGCHAIN`, `QWEN3-OPENAI`).
2. FastAPI router (`routers/extract.py`, prefix `/extract`, tag `OCR`) validates, parses model selection, and dispatches via `asyncio.gather`.
3. YOLOv8 Splitter (`services/yolo_splitter.py`): pypdfium2 renders pages at `scale=2` (`page.render(scale=2)`, line 45); YOLOv8 (`yolo-multipage-OCR-cls.pt`) runs with `CONFIDENCE_THRESHOLD = 0.5` (line 39), `IOU_THRESHOLD = 0.4` (line 40); detections are cropped. Non-`doc` classes — in practice police-report pages — are filtered out and written to `skipped_images/<class>_<idx>.jpg`; if nothing is detected, the full uncropped image is safely retained as class `doc`.
4. Image preparation (`llm_engine.py`): `img.thumbnail((1536, 1536))`, JPEG `quality=80`.
5. Stage 1: crop plus country-specific `DOCUMENT_ANCHORS` keyword rules go to the selected VLM to determine `doc_type` (e.g., Doha residency permits keyed on 'ID. Card', 'residency permit', 'employer').
6. Guard: `if not doc_type or doc_type not in OUTPUT_DICT.get(country, {})` → None (`llm_engine.py:162`).
7. Stage 2: a strict JSON schema is synthesized from `OUTPUT_DICT[country][doc_type]`; the VLM is queried with `strict=True`, `temperature=0`, `additionalProperties=False`.
8. Results aggregate into `[{"messageType": "success", "message": "S", "document_type": "<doc_type>", "response": {...}}]`; failures return HTTP 400 `'No cropped images found'` or `'No valid documents found'`.

**Path B — Police accident reports (`/sharjah`, `/abu_dhabi`, `/uae`, `/doha` on `pd_router`, tag `Police_docs`):**
1. Client POSTs `multipart/form-data` field `file` (PDF); `police_report_controller.py` checks file count and enforces `mime_type == 'application/pdf'` (else `Unsupported file type: <mime>`).
2. Jurisdiction parser: Saaed Abu Dhabi — PyMuPDF `fitz.open(file_path)` vector tables, no OCR; Dubai Police — PyMuPDF text blocks, `affected_party`/`liable_party` parsed on `ThreadPoolExecutor(max_workers=6)`; Rafid Sharjah — Wand/ImageMagick 300 DPI render, Google Cloud Vision OCR, custom 2D coordinate sorting (10 px line threshold), 5 workers; Doha — PyMuPDF block spans, table extraction, GCV OCR.
3. Arabic reshaping (`arabic_reshaper`, `bidi.algorithm.get_display`) and GoogleTranslator English translation of accident causes, driver remarks, property damages.
4. Validity gate: >3 populated values (Sharjah, Abu Dhabi, Dubai) or >0 (Doha); otherwise `{"message": "Irrelevant document to proceed/Document quality is low, Please try another...!", "messageType": "E"}` (plain or gzip-compressed).
5. Success: `{"message": "success", "messageType": "S", "response": {...}}`, with an authority-specific response shape:
   - Abu Dhabi (`/abu_dhabi`): `report_date`, `incident_number`, `vehicle_party[{driver_detail, insurance_detail, vehicle_detail}]`, `faulty_damaged_parts`, `claim_description`.
   - Sharjah (`/sharjah`): `accident_info`, `vehicle_party[]`, `injured_party[]`, `property_damage_party[]`, `claim_description`.
   - Dubai (`/uae` on `pd_router`, **not** `/extract/uae`): `reason_of_accident`, `incident_number`, `vehicle_party[]`, `important_notes[]`.
   - Doha (`/doha` on `pd_router`): `report_status`, `incident_date`, `incident_number`, `vehicle_party[]`, `injured_party[]`, `property_damage_party[]`, `guilty_charge`.

**Data flow:** PDF/image bytes → pypdfium2 page renders → YOLO boxes → cropped, downscaled JPEGs → VLM (image + anchor prompt) → `doc_type` → VLM (image + strict schema) → JSON array. Police: PDF → PyMuPDF blocks/tables or 300 DPI raster → GCV OCR with coordinates → sorted lines → parsed dict → reshaped/translated → JSON object.

**Integrations and dependencies:** YOLOv8, pypdfium2, PyMuPDF, OpenCV, Pillow, Wand; Google Cloud Vision (service account `OCR_key.json`); AsyncOpenAI against a local vLLM host (172.20.132.226) serving Qwen3-VL-4B-Instruct; Google GenAI (Gemini-3-Flash-Preview); Azure AI Services (Dev-Llama-4-Scout-17B-16E-Instruct); LangChain Core/OpenAI/Google GenAI; arabic-reshaper, python-bidi, deep_translator.

```
Client -> FastAPI (/extract/{country}?model_name=...) -> pypdfium2 render 2.0x -> YOLOv8 (conf 0.5 / IoU 0.4) crop -> thumbnail 1536 / JPEG q80 -> VLM Stage 1 (DOCUMENT_ANCHORS) -> VLM Stage 2 (OUTPUT_DICT strict schema, T=0) -> JSON array
Client -> FastAPI (/sharjah | /abu_dhabi | /uae | /doha) -> police_report_controller -> PyMuPDF tables/blocks | Wand 300 DPI + GCV OCR + coordinate sort -> arabic_reshaper/bidi + GoogleTranslator -> validity gate (>3 / >0) -> JSON
```

**Architecture Summary:** A decoupled FastAPI service (port 5057) with a police-docs sub-service (port 5055). Identity documents pass through YOLOv8 pre-segmentation before a provider-agnostic VLM engine performs classification and schema-constrained extraction; police reports bypass the VLM and go to hand-built PyMuPDF/GCV parsers. Configuration-driven schemas (`DOCUMENT_ANCHORS`, `OUTPUT_DICT`) allow new document types without engine changes, and `model_name` switches between on-premise and cloud backends per request.

## 4. Technologies and Tools Used

| Category | Technologies (with versions where the document states them) |
|---|---|
| Programming languages | Python 3.11.15 (Conda environment `plp`) |
| Frameworks | FastAPI (unpinned; 0.141.1 installed); Uvicorn (unpinned; 0.52.0 installed); LangChain Core (unpinned; 1.5.4 installed); LangChain OpenAI (unpinned; 1.5.0 installed); LangChain Google GenAI (unpinned; installed) |
| Libraries | Ultralytics YOLOv8 (`==8.4.14` declared; 8.4.113 installed); OpenCV opencv-python 4.11.0.86; Pillow 12.3.0; PyMuPDF (fitz) 1.28.0; pypdfium2; OpenAI Python SDK 2.52.0 (AsyncOpenAI); arabic-reshaper; python-bidi 0.6.11; deep_translator (GoogleTranslator); Wand (ImageMagick binding); asyncio; concurrent.futures ThreadPoolExecutor; Pydantic (referenced for strict schema) |
| AI/ML models | YOLOv8 custom weights `yolo-multipage-OCR-cls.pt` (54.5 MB) and `best.pt` (45.2 MB); Qwen3-VL-4B-Instruct (local vLLM); Gemini-3-Flash-Preview (Google GenAI); Dev-Llama-4-Scout-17B-16E-Instruct (Azure AI Services). Experimental only: Gemini-3-Pro-Preview, Qwen 2.5, Qwen 0.6B, Azure Document Intelligence custom models (`anoud_ocr_custom_uae_vb`, `anoud_ocr_custom_uae_vf`, `prebuild-anoud-ocr-ai`) and prebuilt-layout |
| OCR tools | Google Cloud Vision API (google-cloud-vision); VLM-based OCR via Qwen3-VL / Gemini / Llama; PyMuPDF vector text extraction (no OCR) for Saaed Abu Dhabi and Dubai Police |
| Databases | Not stated in documentation |
| Cloud platforms | Google Cloud (Vision API, GenAI/Gemini); Azure AI Services (Llama-4-Scout); on-premise GPU vLLM host at 172.20.132.226 |
| APIs | FastAPI REST endpoints (GET /, POST /extract/uae, /extract/oman, /extract/kuwait, /extract/doha, /extract/passport, POST /sharjah, /abu_dhabi, /uae, /doha); auto-generated Swagger UI (/docs), ReDoc (/redoc), OpenAPI JSON (/openapi.json); Google Cloud Vision API; Google GenAI API; Azure AI Services API; OpenAI-compatible vLLM API; Google Translate via deep_translator |
| DevOps tools | Not stated in documentation (no CI/CD, containerization, or test framework present; automated testing "Not implemented in the project") |
| Version control | Not stated in documentation (repository path referenced, but no VCS tooling named) |
| Deployment tools | Uvicorn ASGI server; `.env` configuration (port 5057; police sub-service port 5055); Conda environment `plp`. No container or orchestration tooling stated |
| Document processing tools | pypdfium2 (page rendering at 2.0x); PyMuPDF (block spans, table extraction); Wand/ImageMagick (300 DPI rendering, `sharjah_police_report_img.py:731`; no version stated); Pillow/OpenCV (cropping, thumbnailing, JPEG compression); custom `coordinates_sort.py` (2D coordinate sequence sorting, 10 px line threshold at lines 33 and 210) |
| Automation tools | Python asyncio (`asyncio.gather`); ThreadPoolExecutor (5 workers Sharjah `sharjah_police_report_img.py:745`, 6 workers Dubai `dubai_police_report.py:297,312`); Jupyter notebook `debug.ipynb` (214 cells) for experimentation |

Documentation notes on tooling gaps, stated verbatim in the source: no automated test code, test scripts, or test frameworks (pytest/unittest) exist — "Automated testing status: Not implemented in the project"; no spreadsheet assets (.xlsx/.xls/.csv) exist in the project folder; and no specific Swagger UI library version string is declared anywhere in configuration or application code (FastAPI's auto-generated UI is used as-is).

## 5. Technical Skills Demonstrated

- **Software Engineering:** Modular package layout (`app.py`, `routers/`, `services/`, `services/police_docs/`, `config/`); refactoring duplicated Qwen prototypes into a single `config_dict`-driven engine; archiving superseded code under `services/backup/`; environment-based configuration; structured error envelopes and gzip-compressed error responses.
- **AI Engineering:** Designed a multi-backend VLM dispatcher supporting AsyncOpenAI and LangChain Core paradigms; per-request model selection across local and cloud foundation models; two-stage classification→extraction pipeline with dynamic strict JSON schema synthesis; constrained decoding via `strict=True`, `temperature=0`, `additionalProperties=False`.
- **Machine Learning:** Trained/used a custom YOLOv8 multi-document detection model; empirical threshold tuning (iou=0.9 vs. conf=0.5/iou=0.4) with recorded outcomes; model checkpoint management (`yolo-multipage-OCR-cls.pt`, `best.pt`).
- **NLP:** Bidirectional Arabic text correction with arabic_reshaper and `bidi.algorithm.get_display`; automated Arabic→English translation of accident narratives via deep_translator; keyword-anchor document classification (and its heuristic predecessor `detect_document_sides` in `services/detector.py`).
- **Computer Vision:** Object detection and cropping on rendered PDF pages; class-based routing of detections (`doc` vs. non-doc); image downscaling/compression for VLM input; 300 DPI rasterization for OCR; spatial line reconstruction from OCR coordinates (10 px threshold).
- **Data Engineering:** Configuration-driven schema catalogs (`DOCUMENT_ANCHORS`, `OUTPUT_DICT`) covering 18+ document types; structured JSON output contracts per jurisdiction; multi-format ingestion (PDF, JPEG, PNG) with rendering normalization.
- **Cloud:** Google Cloud Vision API with service-account authentication (`OCR_key.json`); Google GenAI (Gemini); Azure AI Services (Llama-4-Scout); hybrid on-premise/cloud inference topology for data sovereignty.
- **MLOps:** Limited — experiment tracking is notebook-based (debug.ipynb); model weights stored in-repo; declared vs. installed version drift documented. No CI/CD, tests, or monitoring demonstrated.
- **API Development:** FastAPI with APIRouter prefixes/tags, multipart uploads, required enum query parameters, auto-generated OpenAPI/Swagger/ReDoc, HTTP 400 error semantics, health-check route.
- **Prompt Engineering:** Country-specific anchor keyword rules for Stage 1 typing; schema-as-prompt Stage 2 extraction; migration from OCR-text+regex prompting to direct image-grounded single-pass VLM prompting; deterministic decoding at temperature 0.
- **Data Extraction / Document AI:** Vector-PDF table and block-span extraction (PyMuPDF `fitz`) chosen over OCR wherever the PDF carries a text layer; raster fallback via Wand 300 DPI + Google Cloud Vision only where the source is a scan (Rafid Sharjah); per-jurisdiction field contracts including camelCase/snake_case and single- vs. front/back-key variants.
- **System Design:** Decoupled pipeline stages; pluggable model engine decoupling business logic from vendors; separate controller path for police reports; asynchronous fan-out (`asyncio.gather`) and thread pools for CPU/IO-bound parsing; fallback behaviors (uncropped `doc` retention, skipped_images diversion).

## 6. Detailed Technical Contributions

**Features implemented**
- Automated multi-document detection and splitting with fallback retention of the full image as class `doc`.
- Two-stage anchor matching and dynamic strict schema decoding for 18+ document types across UAE, Qatar/Doha, Oman, Kuwait, and passports for India, Pakistan, Oman, Philippines, Egypt, Bangladesh.
- Six selectable execution modes via `model_name`: `LLAMA-LANGCHAIN`, `LLAMA-OPENAI`, `GEMINI-LANGCHAIN`, `GEMINI-OPENAI`, `QWEN3-LANGCHAIN`, `QWEN3-OPENAI`.
- Four police accident report parsers (Saaed Abu Dhabi, Dubai Police, Rafid Sharjah, Doha Traffic Police) with bilingual reshaping/translation.
- Health check `GET /` returning `{"status": "Running"}` (declared `async def root()` in `app.py:18-19`; no error responses declared).
- Configuration-level schema exceptions handled per jurisdiction: Kuwait vehicle registration keyed as a single `vehicle_registration` rather than front/back pairs; Doha residency permits anchored on the keywords `'ID. Card'`, `'residency permit'`, `'employer'`; Philippine passports keyed `'fillipino'` consistently across `document_anchors.py` and `output_keys.py`.
- Bilingual output: arabic_reshaper + `bidi.algorithm.get_display` correction plus GoogleTranslator English translation applied specifically to accident causes, driver remarks and property damages.

**Models used**
- YOLOv8 `yolo-multipage-OCR-cls.pt` (production); `best.pt` (referenced in debug.ipynb).
- Qwen3-VL-4B-Instruct on a private vLLM host (172.20.132.226); Gemini-3-Flash-Preview via Google GenAI; Dev-Llama-4-Scout-17B-16E-Instruct via Azure AI Services.

**Pipelines built**
- Identity pipeline: `page.render(scale=2)` → YOLO (conf 0.5, IoU 0.4) → crop → thumbnail 1536x1536 / JPEG q80 → Stage 1 anchor classification → `OUTPUT_DICT` guard → Stage 2 strict extraction → array envelope.
- Police pipelines: four per-authority parsers (Saaed, Dubai, Sharjah, Doha) as detailed in Section 3 Path B, sharing the controller, Arabic reshaping/translation, and validity-gate stages.

**APIs integrated**
- `router = APIRouter(prefix='/extract', tags=['OCR'])` and `pd_router = APIRouter(tags=['Police_docs'])` mounted in `app.py`; unmounted `routers/uae.py` (`POST /uae/`) with an unresolvable `services.qwen_langchain` import.
- Google Cloud Vision (`ocr_engine.py:24`, key from `.env:14`); Google GenAI; Azure AI Services; OpenAI-compatible vLLM; Google Translate via deep_translator.

**Data extraction methods**
- VLM image-grounded extraction into per-type schema fields, e.g., Oman `driving_license_front`: `license_number`, `issued_place_english`, `name_english`, `first_issued`, `expiry_date`; Doha `residency_permit_front`: `idNo`, `dob`, `expiry`, `nationalityEng`, `nationalityAra`, `nameEng`, `name_ar`; passports: `type`, `code`, `passport_number`, `given_name`, `fullname`, `date_of_birth`, `date_of_expiry`.
- PyMuPDF table/block extraction for vector PDFs; GCV OCR plus 2D coordinate sorting (10 px threshold) to reconstruct reading order on scans.

**Validation logic**
- Identity path: `doc_type` must exist in `OUTPUT_DICT[country]`, else the crop yields None; HTTP 400 when no crops or no valid documents.
- Police path: exactly one PDF file (`mime_type == 'application/pdf'`); document validity requires >3 non-empty response values (Sharjah line 770, Abu Dhabi line 520, Dubai line 343) or >0 (Doha line 734).
- Schema-level validation delegated to provider strict mode (`strict=True`, `additionalProperties=False`).

**Automation workflows**
- `asyncio.gather` fan-out over uploaded files/crops; ThreadPoolExecutor for concurrent party parsing; automatic diversion of non-document detections to `skipped_images/`.

**Optimization techniques**
- Single-pass VLM extraction replacing sequential OCR+LLM+regex (documented as high-latency); image downscaling to 1536x1536 and JPEG q80 to bound VLM payload; deterministic decoding at temperature 0; unified engine to remove code duplication.

**Performance improvements**
- Qualitative only: the source records that the OCR+regex prototype "showed high latency from sequential OCR + LLM passes" and Azure DI "exhibited variable latency," motivating the current design. No measured latency or accuracy figures are given.

**Experimentation lineage (debug.ipynb, 214 cells, with recorded keep/abandon verdicts)**
- Cells 1-11: Azure Document Intelligence custom-trained models `anoud_ocr_custom_uae_vb`, `anoud_ocr_custom_uae_vf`, `prebuild-anoud-ocr-ai` — extracted raw key-value pairs successfully but showed variable latency and high cloud API coupling. Abandoned, replaced by open VLM and local inference.
- Cells 27-46: Azure DI `prebuilt-layout` with `keyValuePairs` — produced text blocks and coordinate bounding boxes but struggled with multi-document scans. Abandoned; moved to `services/backup/azure_engine.py`.
- Cells 40-45: Qwen 2.5 and Qwen 0.6B local Ollama/vLLM prompt engineering — demonstrated structured JSON generation on local CPU/GPU. Evolved into the production Qwen3-VL-4B-Instruct engine.
- Cells 116-138: YOLOv8 splitting parameter tuning (iou=0.9 vs conf=0.5/iou=0.4) — stored outputs show successful bounding-box cropping of multiple ID cards from single high-resolution scans. Carried into production.
- Cells 191-214: Gemini-3-Flash-Preview and Gemini-3-Pro-Preview structured JSON schema inference via the Google GenAI SDK — valid JSON with strict field-level schema compliance. Carried into production as `GEMINI-OPENAI` / `GEMINI-LANGCHAIN`.
- Archived/legacy code: `services/backup/azure_engine.py` (async `azure_kv_extract`; AZURE engine commented out at `routers/extract.py:20`); `services/backup/gemini_langchain.py` (OCR text → regex post-processors, superseded by single-pass VLM); `qwen_langchain.py` + `qwen_openai.py` (functional but duplicated, unified into `llm_engine.py` via `config_dict`); `services/detector.py` `detect_document_sides` (keyword-frequency heuristic classification, now imported only by the backup file at `gemini_langchain.py:135`, superseded by VLM prompt classification at `llm_engine.py:144-158`).

## 7. Challenges and Solutions

1. **Composite scans with multiple ID cards per page (technical).** Azure DI prebuilt-layout "struggled with multi-document scans." Solution: custom YOLOv8 detector over 2.0x-rendered pages at conf 0.5 / IoU 0.4 (chosen over iou=0.9), with uncropped fallback. Alternatives: Azure DI layout (documented, abandoned); OpenCV contour-based segmentation (analyst suggestion).
2. **Hallucination and schema drift in LLM output (technical/business).** Solution: dynamic strict JSON schema from `OUTPUT_DICT` with `strict=True`, `temperature=0`, `additionalProperties=False`. Alternatives: OCR text + regex post-processors (documented in `services/backup/gemini_langchain.py`, superseded); Pydantic post-validation with retries (analyst suggestion).
3. **Vendor lock-in and data sovereignty (business).** Solution: pluggable engine with on-premise Qwen3-VL-4B-Instruct on a private vLLM host plus cloud Gemini and Azure Llama. Alternative: Azure DI custom models (`anoud_ocr_custom_uae_vb/vf`) — documented, abandoned for "high cloud API coupling."
4. **Heterogeneous police report formats (technical).** Solution: one parser per authority — PyMuPDF tables (Saaed), blocks + threads (Dubai), Wand 300 DPI + GCV + coordinate sort (Rafid Sharjah), blocks/tables + GCV (Doha). Alternative: route police reports through the VLM strict-schema engine (analyst suggestion; needs evaluation on dense tables).
5. **Reversed/disconnected Arabic in PDFs (technical).** Solution: arabic_reshaper + `bidi.algorithm.get_display`, then GoogleTranslator for narratives. Alternative: let a VLM read Arabic from the rendered page (analyst suggestion).
6. **Reading order from OCR coordinates (technical).** Solution: `coordinates_sort.py` grouping tokens into lines with a 10 px threshold. Alternative: GCV's block/paragraph hierarchy (analyst suggestion).
7. **Latency of sequential OCR→LLM passes (technical).** Solution: single-pass multimodal extraction, `asyncio.gather`, thread pools (5/6 workers). Alternatives: batching several crops into one VLM request, or caching Stage 1 results per page (analyst suggestions).
8. **Documented defects and limitations (technical):** (a) UAE anchor keys (`license_front`, `license_back`, `residence_front`, `residence_back`, `vehicle_front`, `vehicle_back` in `config/document_anchors.py`) do not match `OUTPUT_DICT` keys (`driving_license_front`, `driving_license_back`, `residency_permit_front/back`, `vehicle_registration_front/back` in `config/output_keys.py`), so the guard at `services/llm_engine.py:162` always returns None and `/extract/uae` always returns HTTP 400 'No valid documents found' — the source notes `config/document_anchors - Copy.py` held matching keys before being shortened, i.e. a regression introduced by an edit; (b) `police_report_controller.py:49-52` uses `if len(file_img) > 1` with a misleading "Please attach at least one file/document to proceed...!" message, restricting input to exactly one file, and line 65 accepts only `mime_type == 'application/pdf'` (all other types rejected as `Unsupported file type: <mime>`); (c) inconsistent validity gates — >3 non-empty response values for Sharjah (line 770), Abu Dhabi (line 520) and Dubai (line 343) versus >0 for Doha (`doha_police_report.py:734`); (d) Philippine passports keyed `fillipino`; (e) unmounted `routers/uae.py` (`APIRouter(prefix='/uae', tags=['UAE'])`, `POST /uae/`) never imported into `app.py` and importing a non-existent `services.qwen_langchain` (the file actually lives at `services/backup/qwen_langchain.py`); (f) unpinned dependencies with drift (ultralytics `==8.4.14` declared vs. 8.4.113 installed; FastAPI, Uvicorn, PyMuPDF, LangChain packages, google-cloud-vision and deep_translator all unpinned in `Requirements.txt`); (g) no automated tests, scripts or frameworks of any kind; (h) Google service-account key `OCR_key.json` stored in the project directory and referenced from `.env:14` and `ocr_engine.py:24` (analyst observation: secrets-management risk); (i) unreferenced assets left in the tree (`labels.json`, the four `skipped_images/original_*.pdf` test PDFs).
9. **Security/authentication posture (technical).** The only authentication documented anywhere in the system is the Google Cloud service-account credential (`OCR_key.json`, service account `ocrqicuser`) used for the Vision API. No API-level authentication, authorization, rate limiting, or transport security is described for the FastAPI endpoints — treat this as *not stated in documentation* rather than as an implemented control; adding gateway-level auth would be the first hardening step (analyst suggestion).

## 8. Impact Analysis

The document provides **no quantified accuracy, throughput, latency, or cost-savings figures**. Impact below is qualitative, as stated in the source.

- **Business impact:** Enables straight-through processing for policy onboarding and motor claims registration; centralizes document intelligence for the GCC jurisdictions (UAE, Qatar, Oman, Kuwait — the source says "five GCC nations" but lists four) and six passport nationalities in one API.
- **Productivity improvements:** Eliminates manual pre-processing of composite scans; automates extraction of liable/affected parties, license details, damaged parts, and accident causes from four police authorities.
- **Accuracy improvements:** Strict schema enforcement described as "zero-hallucination" and "guaranteeing schema compliance"; no measured accuracy reported.
- **Cost savings:** Not stated. Provider agility and on-premise inference are positioned as strategic, not costed.
- **Time savings:** Source states extraction "in seconds" (qualitative); prototypes were dropped for "high latency," implying the production design is faster, but no numbers are given.
- **User benefits:** Claims and underwriting staff receive bilingual, structured accident and identity data; integrators get a uniform JSON envelope, Swagger docs, and per-request model choice.
- **The only numbers the source actually states are configuration constants and artefact sizes, not outcomes:** conf 0.5 / IoU 0.4, render scale 2.0x, 1536x1536 thumbnail, JPEG q80, temperature 0, 300 DPI Wand rendering, 10 px coordinate-sort line threshold, ThreadPoolExecutor 5/6 workers, validity gates >3 (>0 Doha), ports 5057 and 5055, model weights 54.5 MB and 45.2 MB, 214 notebook cells, 18+ document types, 6 passport nationalities, 6 model modes, 9 functional endpoints. None of these is an accuracy, throughput, latency or cost measurement, and none should be presented as one.

## 9. Interview Discussion Points

- Why two VLM stages rather than one prompt? Stage 1 resolves `doc_type` so Stage 2 can bind an exact `OUTPUT_DICT` schema; explain the `strict=True`/`additionalProperties=False` guarantee and the membership guard.
- How were YOLO thresholds chosen? Cite the debug.ipynb comparison (iou=0.9 vs. conf=0.5/iou=0.4) and the uncropped `doc` fallback.
- Why abandon Azure Document Intelligence? Variable latency, cloud coupling, failure on multi-document scans — and how local Qwen3-VL on vLLM addresses sovereignty.
- Why support both AsyncOpenAI and LangChain? Unifying `qwen_langchain.py`/`qwen_openai.py` via `config_dict`, and the cost of maintaining two paths.
- How would you fix and prevent the UAE key mismatch? Config-consistency tests asserting anchor keys ⊆ `OUTPUT_DICT` keys per country; acknowledge the missing test suite.
- Why bespoke police parsers instead of the VLM? Vector PDFs allow lossless PyMuPDF table extraction; Rafid Sharjah scans need Wand 300 DPI + GCV + coordinate sorting.
- Arabic handling: what breaks without arabic_reshaper/bidi, and why translate only narratives.
- Concurrency: `asyncio.gather` vs. `ThreadPoolExecutor(max_workers=5/6)`; which stages are IO- vs. CPU-bound.
- Preprocessing: 2.0x render for detection vs. 1536x1536/JPEG q80 for VLM payload — resolution against token/latency cost.
- Validity gates: why >3 non-empty values, the Doha >0 inconsistency, and what a principled quality check looks like.
- Production hardening: unpinned requirements, ultralytics drift, secrets in-repo; what metrics you would instrument next (field-level accuracy per doc type, per-model latency, fallback rate).
- Why keep abandoned engines in-tree rather than delete them? `services/backup/azure_engine.py` plus the commented-out AZURE branch at `routers/extract.py:20` preserve a reversible decision — and the cost is dead code and confusing imports (`routers/uae.py`).
- Why schema shapes differ per jurisdiction (Kuwait's single `vehicle_registration` key, Doha's camelCase `idNo`/`nameEng` alongside snake_case elsewhere): fidelity to the physical document and to consuming systems, versus the maintenance cost of an inconsistent contract — and how you would normalise it behind a stable public schema.
- The police controller's one-file, PDF-only restriction: why the `len(file_img) > 1` guard is inverted relative to its message, what a correct guard looks like, and whether multi-file batch police intake is worth supporting.
- Where the two-stage design can fail closed: an anchor keyword miss produces no `doc_type`, the crop is silently dropped, and the endpoint reports a generic 400 — how you would surface per-crop diagnostics instead of an all-or-nothing error.
- Security: only the GCP service account is authenticated; the API itself has no documented auth — what you would put in front of an endpoint receiving passports and civil IDs.

## 10. Architecture Explanation Points

Draw two lanes. Lane 1: **Client → FastAPI (`/extract/{country}?model_name=`) → YOLOv8 Splitter → Unified VLM Engine → JSON array**. pypdfium2 renders at 2.0x; YOLOv8 (`yolo-multipage-OCR-cls.pt`, conf 0.5/IoU 0.4) crops each sub-document, shunts non-doc classes to `skipped_images/`, and falls back to the whole image on empty detection. The engine downsizes crops (1536x1536, JPEG q80), runs Stage 1 anchor classification and Stage 2 strict-schema extraction at temperature 0, and fans out with `asyncio.gather`. Lane 2: **Client → `/sharjah | /abu_dhabi | /uae | /doha` → police controller → PyMuPDF or Wand+GCV parsers → Arabic reshaping/translation → validity gate → JSON**.

Key decisions: segment before extracting so each VLM call sees one document; configuration-driven schemas to add jurisdictions without code; pluggable backends (on-premise Qwen3-VL for sovereignty, Gemini/Llama for cloud) selectable per request; bespoke parsers where vector PDF text is lossless.

Component boundaries as the source names them: **FastAPI Router** (`app.py`, `routers/extract.py`) owns request validation, `model_name` query parsing and async dispatch via `asyncio.gather`; **YOLOv8 Splitter** (`services/yolo_splitter.py`) owns pypdfium2 rendering, detection, cropping and non-doc diversion; **Multimodal VLM** (`services/llm_engine.py`) owns two-pass classification against `DOCUMENT_ANCHORS` and strict-schema extraction against `OUTPUT_DICT` across the three backends. Configuration (`config/document_anchors.py`, `config/output_keys.py`) is the only place a new jurisdiction is added. The police path shares none of this: `services/police_docs/police_report_controller.py` fronts four independent parser modules and its own `.env` (port 5055) beside the main service (port 5057) — the runtime relationship between the two ports is not explained in the source.

Trade-offs: two VLM calls per crop buy schema certainty at latency cost; two execution paradigms (AsyncOpenAI and LangChain) double the surface to maintain in exchange for vendor portability; bespoke parsers are precise but brittle to layout changes; strict mode depends on provider support; the configuration-driven schema catalog makes new document types cheap but has no mechanism enforcing that anchor keys and output keys stay in sync — exactly the failure that disabled `/extract/uae`; and the 2.0x render for detection versus the 1536x1536 / q80 downscale for inference is a deliberate two-resolution pipeline (detect on detail, infer on payload budget).

Next: fix the UAE key mismatch and add config-consistency and endpoint tests; unify validity gates; pin dependencies and containerize; move `OCR_key.json` to a secrets manager; add per-model accuracy/latency telemetry; evaluate routing police scans through the VLM engine.
