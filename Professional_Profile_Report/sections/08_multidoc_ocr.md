# Project 8: Multi-doc All-in-one OCR: Classifier + Extractor

## 1. Project Overview

- **Project name:** Multi-doc All-in-one OCR: Classifier + Extractor (document title "Multi-doc: Classifier + Extractor"; service self-identifies as "UAE OCR: Classifier + Extractor" v1.0.0; Figure 1 is titled "UAE OCR: Classifier + Extractor: End-to-End Architecture"; doc no. AT/multidoc-ocr/V1, 01-Sep-2026; client Qatar Insurance Group).
- **Business domain:** Insurance — automated intake of customer identity documents (driving licenses, resident IDs), vehicle licenses/registrations, police reports (Sharjah old template, Sharjah new template, Abudhabi police report, Dubai police report), and doctor licenses for policy, underwriting, and claims workflows across UAE and Qatar (DOHA), with OMAN and KUWAIT pre-registered. Named target users are automated upstream enterprise systems — insurance core policy systems, underwriting intake workflows, claims processing portals submitting customer identity and vehicle documents for straight-through extraction — plus technical operations and management review teams, who use the service health endpoints and `output.json` audit logs for operational oversight.
- **Problem statement:** Upstream insurance systems receive mixed bundles of documents as JPEG/PNG images or multi-page PDFs. One image often holds several documents (e.g. a Qatar card with front and back on one sheet), the type is unknown in advance, and each type needs a different field extractor. Routing was a multi-step manual process, unreadable or expired documents surfaced late, and no single integration contract existed across emirates (inferred from the stated business benefits).
- **Project objective:** One FastAPI REST service (`POST /ocr/uae_allinone_ocr`) that segments uploaded files into document crops (YOLO), OCRs them (RapidOCR or Azure Document Intelligence), classifies each by fuzzy n-gram keyword matching against per-region JSON dictionaries, checks expiry and image quality, forwards each crop to its extractor microservice, and returns a unified JSON payload plus an `output.json` audit record.
- **Expected business outcome:** Straight-through extraction for policy, underwriting, and claims portals; early rejection of poor-quality submissions; automated expiry verification; one consistent upstream contract; vendor-neutral OCR redundancy for restricted networks. The document states that quantified business metrics (throughput, cost savings, ROI, accuracy) are not recorded.

## 2. Role and Responsibilities

**Core responsibilities** (inferred from the system design and code references in the document):

- Designed the end-to-end pipeline: FastAPI gateway -> YOLO cropper -> OCR engine -> fuzzy classifier -> validity/quality inspector -> extractor dispatcher -> JSON/audit store.
- Built the FastAPI gateway in `uae_allinone_ocr.py`: `root_path='/ocr'` for Nginx routing, `custom_openapi` override so multipart array file fields emit `format: binary`, request UUIDs, multipart validation, `ThreadPoolExecutor(max_workers=10)`.
- Integrated the YOLO segmentation model `yolo-multipage-OCR-cls.pt` (`imgsz=320`, `conf=0.25`) in `utils.py:detect_document_crops` / `_run_yolo_on_image` to identify bounding boxes for identity cards and licenses, with full-frame fallback and the police-report crop-discard rule (`utils.py` lines 244-246); the DOHA bypass lives in the gateway (`uae_allinone_ocr.py` line 172).
- Implemented the dual OCR abstraction: `utils.py:rapid_ocr` (thread-local singleton `_ocr_thread_local`) and `utils.py:azure_ocr` (`prebuilt-read`), selected per request via `ocr_type`.
- Designed the fuzzy keyword classifier (`utils.py:keyword_score`): 1-3 word n-grams, RapidFuzz `ratio` / `token_sort_ratio` with cutoff 80, percentage scoring per label, and hot-reloaded `labels_DB/<region>_labels.json` configuration.
- Implemented validity (regex dates vs system time) and OpenCV quality inspection with explicit thresholds.
- Built the dispatch layer: an in-memory routing table built from `EXTRACTOR_URLS` configured in `.env` (14 mappings), HTTP POST of file buffers with `CONNECT_TIMEOUT=3.0s` / `READ_TIMEOUT=30.0s`, concurrent execution on ThreadPoolExecutor worker threads, inline error statuses.
- Defined the response contract (`source_id`, `reference_id`, `timestamp`, `results[]`) and the `output.json` audit log (the Audit & Persistence Store component appends every processed transaction, timestamp, reference ID, and result array to `output.json` on local disk).
- Ran POC experimentation in `test.ipynb` (71 cells) and carried validated approaches into production, with cell-level traceability recorded per trial: YOLO cropping and multi-page splitting (cells 4, 13, 31, 41-43), Azure custom extraction (cells 9-12), local RapidOCR CPU execution (cells 25, 38, 51, 60), RapidFuzz keyword scoring (cells 50, 58-64), automated keyword suggestion `predict_keyword` (cells 56-57), concurrent downstream extractor dispatch (cells 18-23, 27), and classification cutoff thresholding (cells 63-69).

**Supporting tasks** (attribution to the candidate is inferred; the document names no owner):

- Authored `requirements.txt` (CPU-only PyTorch wheel via extra-index-url; every dependency unpinned) and `.env` extractor configuration (inferred).
- Pre-registered OMAN and KUWAIT in the Region Enums so a new jurisdiction activates by supplying a `labels_DB` JSON dictionary.
- Built `utils.py:suggest_keywords` (from the notebook's `predict_keyword`) and integrated it into the Label Trainer interfaces.
- Defined deployment: dev (`python uae_allinone_ocr.py`, Uvicorn 8083, `reload=True`), production (`uvicorn uae_allinone_ocr:app --host 0.0.0.0 --port 8083`, optionally with multiple workers), Nginx gateway reverse-proxy access at `http://<server-host>:9000/ocr/` mapped to internal port 8083.
- Exposed `GET /health` as a health and smoke-test monitoring endpoint for monitoring systems and container healthchecks (no error responses declared); produced documentation with a verified constants table and line-referenced edge-case findings (inferred).

**Estimated ownership level:** Senior AI Engineer (with solution-architecture scope, inferred).

Rationale: seven architectural components named in the document, two interchangeable OCR vendors, a custom-named YOLO model, a region-extensible classification scheme, thread-safety engineering for ONNX sessions, 14 downstream extractor mappings, a reverse-proxied production deployment, and POC-to-production traceability across a 71-cell notebook. The candidate defines the contract upstream insurance systems consume and the hub tying the extractor microservices together. Team size and formal title are not stated; whether the candidate trained the YOLO model is not stated.

## 3. End-to-End Workflow

**Processing stages:**

1. **Request intake.** Client sends `multipart/form-data` to `POST /ocr/uae_allinone_ocr` with `files` (list[UploadFile], required), `reference_id` (required), `ocr_type` (`CPU_OCR` default or `AZURE`), and `region` (`UAE` default, `DOHA`, `OMAN`, `KUWAIT`).
2. **Config load and request ID.** Gateway generates a request UUID (returned as `source_id`, inferred from the sample response) and "loads regional keyword configuration" from `labels_DB/<region>_labels.json`; the document states the keyword sets hot-reload without an application restart, so the load is per request (inferred). Missing file -> HTTP 400 `{"detail": "Config not found for region: <region_code>"}`; load failure -> HTTP 500 `{"detail": "Config Error: <exception>"}`.
3. **Rasterization.** JPEG/PNG buffers pass through unchanged (inferred); PDFs are rasterized by PyMuPDF (`fitz`) at 150 DPI (matrix 150/72) into per-page buffers before segmentation.
4. **YOLO segmentation.** `yolo-multipage-OCR-cls.pt` runs at `imgsz=320`, `conf=0.25`; if no box meets the threshold, the full frame is used. For `region='DOHA'` (line 172) YOLO is bypassed and the whole image is one crop. If any crop's class contains `police` (utils.py 244-246), all crops are replaced by `{'class': 'police_report', 'file_bytes': file_bytes}`.
5. **OCR.** Each crop runs in a worker thread (concurrent crop execution at `uae_allinone_ocr.py` line 179): RapidOCR via a thread-local ONNX session on CPU (thread-local sessions are described as mutex isolation), or Azure Document Intelligence `prebuilt-read` (described in the document as "higher accuracy cloud extraction"). If crop bytes start with `%PDF` (lines 75-83), only page 0 is converted to JPEG for OCR and "any secondary pages of the PDF are not processed for keyword classification"; the document titles this finding "Police Report PDF First-Page Only", and the link to the police-report rule — the collapsed full-frame crop carries the original file bytes, which for a PDF upload are the PDF itself — is an analyst inference.
6. **Fuzzy classification.** OCR lines are normalized into 1-, 2-, 3-word n-grams and matched against per-label keyword lists with RapidFuzz (`ratio` single words, `token_sort_ratio` phrases, cutoff 80). The top-scoring label becomes `document_type` (inferred); for DOHA, all labels >= 65% are comma-joined.
7. **Validity and quality.** Regex-parsed dates are evaluated against the current system date to determine validity; the sample response shows `status: "Valid"`, and `Not valid` is a documented status value (mapping of expiry outcome to `Valid` / `Not valid` is inferred). OpenCV grayscale metrics add `Blurry Image`, `Too Dark`, `Overexposed`, `Low Contrast`, or `Low Resolution` flags (thresholds in Section 6). Scores < 90% append `Low confidence: X.X%` to `feedback`; < 65% set `suggestions` to `Add more keywords`, else `All good`.
8. **Extractor dispatch.** `document_type` is resolved through the in-memory routing table loaded from `EXTRACTOR_URLS` in `.env` (e.g. `UAE_DRIVING_LICENSE_FRONT_URL` -> `http://192.168.86.213:8000/uae/license_front`); file buffers (crop bytes) are POSTed over HTTP with 3.0 s connect / 30.0 s read timeouts and the returned schema fields are packaged into `extractor_response`. Because the hub forwards image bytes rather than the OCR text it already produced, each document is effectively OCR'd twice — once for classification, once inside the extractor (inferred). If processing fails for a crop or the extractor is unreachable, `results[].status` is set to `Error` or `Not valid` and `extractor_response` holds the error description or `Endpoint URL not configured`; because these are documented as inline statuses rather than HTTP errors, the request as a whole is inferred to still return 200.
9. **Aggregation and audit.** Per-crop results (`filename`, `page`, `document_type`, `extractor_response`, `status`, `feedback`, `confidence`, `suggestions`) are assembled under `results[]`, appended with `timestamp` and `reference_id` to `output.json`, and returned as JSON.

**Data flow:** multipart bytes -> image/page buffers -> YOLO boxes -> cropped byte buffers -> OCR text lines -> n-gram tokens -> per-label percentage scores -> `document_type` -> HTTP POST of crop bytes -> extractor schema JSON -> merged result dict -> `output.json` append and HTTP response.

**Integrations and dependencies:** Ultralytics YOLO + PyTorch (CPU), RapidOCR (`rapidocr_onnxruntime`), Azure AI Document Intelligence (`prebuilt-read`), RapidFuzz, OpenCV (`opencv-python-headless`), PyMuPDF, FastAPI/Uvicorn, Nginx, and 14 document-type-to-URL extractor mappings resolving to 13 distinct endpoints on `192.168.86.213:8000` — the Sharjah old and Sharjah new templates both route to `/police_report/rafid` (enumerated in Section 6).

**Flow line:**

`Client -> Nginx :9000/ocr/ -> FastAPI :8083 (UUID, region config) -> PyMuPDF 150 DPI (PDF) -> YOLO yolo-multipage-OCR-cls.pt (imgsz 320, conf 0.25) -> RapidOCR (thread-local) | Azure prebuilt-read -> RapidFuzz n-gram scoring vs labels_DB/<region>_labels.json -> expiry regex + OpenCV quality -> ThreadPoolExecutor(10) HTTP POST -> extractor microservice -> merged results[] -> output.json + JSON response`

**Architecture Summary:** The service is an orchestration hub in front of per-document extractor microservices (the document's Figure 1 labels the six stages: FastAPI "validates & logs" -> YOLO Cropper "splits documents" -> OCR Engine "RapidOCR / Azure" -> Document type "navigates to endpoint" -> Extractor dispatcher "another api is hosted" -> JSON format "extracts & output.json"). It holds no database; the only persisted state is the `output.json` audit file. It concentrates the cross-cutting concerns every extractor would otherwise duplicate: multi-page/multi-document splitting, OCR engine selection, document-type identification, expiry and quality gating, and audit logging. Classification is rule-driven (per-region keyword dictionaries) rather than model-driven; the document's stated rationale is zero-downtime rule tuning and modular regional scalability, and the accuracy trade-off versus a trained classifier is an analyst interpretation. Concurrency is crop-level with a bounded thread pool and thread-local ONNX sessions; routing is externalized to `.env`, so extractors can be relocated or added without code changes (inferred).

## 4. Technologies and Tools Used

| Category | Technologies (as stated in the document) |
|---|---|
| Programming languages | Python |
| Frameworks | FastAPI (unpinned); Uvicorn [standard] ASGI server (unpinned); PyTorch (torch, torchvision, CPU wheel via extra-index-url, unpinned) |
| Libraries | Ultralytics YOLO; OpenCV (`opencv-python-headless`); PyMuPDF (`pymupdf` / `fitz`); RapidOCR (`rapidocr_onnxruntime`); RapidFuzz; `ThreadPoolExecutor` (Python standard library, named in the document) |
| AI/ML models | Ultralytics YOLO segmentation model `yolo-multipage-OCR-cls.pt` (imgsz 320, conf 0.25); Azure Document Intelligence `prebuilt-read`; Azure custom extraction model `anoud_ocr_uae_PR_sharjahold` (POC only, abandoned) |
| OCR tools | RapidOCR (`rapidocr_onnxruntime`, local CPU); Azure AI Document Intelligence (`prebuilt-read`) |
| Databases | Not stated in documentation (persistence is file-based: `labels_DB/<region>_labels.json` dictionaries and an append-only `output.json` on local disk) |
| Cloud platforms | Microsoft Azure (AI Document Intelligence) |
| APIs | REST: `GET /health`, `POST /uae_allinone_ocr` (externally `/ocr/uae_allinone_ocr` via `root_path`); 14 downstream extractor mappings resolving to 13 distinct HTTP endpoints on `http://192.168.86.213:8000/...`; Azure Document Intelligence API (listed in the technology stack table as an unpinned `requirements.txt` dependency); OpenAPI/Swagger UI with `custom_openapi` override |
| DevOps tools | Nginx reverse proxy (`:9000/ocr/` -> `:8083`); `.env` configuration; `requirements.txt`; `/health` for container healthchecks (containerization itself not stated) |
| Version control | Not stated in documentation (a "repository" is referenced but no VCS named) |
| Deployment tools | Uvicorn (`uvicorn uae_allinone_ocr:app --host 0.0.0.0 --port 8083`, optional multiple workers); Nginx |
| Document processing tools | PyMuPDF (PDF rasterization at 150 DPI); OpenCV (grayscale metrics, image handling); Ultralytics YOLO (document cropping) |
| Automation tools | Jupyter notebook `test.ipynb` (71 cells) for POC; `suggest_keywords` / Label Trainer interfaces for keyword dictionary maintenance |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Modular split into `uae_allinone_ocr.py` (gateway/orchestration) and `utils.py` (cropping, OCR, scoring); thread-local singleton for ONNX sessions; bounded `ThreadPoolExecutor(max_workers=10)`; explicit connect/read timeouts; HTTP 400/500 plus per-item inline error statuses; environment-driven configuration.
- **AI Engineering:** Multi-model pipeline (YOLO segmentation + two OCR engines + fuzzy classifier) with runtime engine selection, confidence fallback, and traceable POC-to-production migration.
- **Machine Learning:** Applied a YOLO segmentation model with tuned inference parameters (`imgsz=320`, `conf=0.25`); evaluated and abandoned an Azure custom extraction model in favor of generic `prebuilt-read` plus external extractors. Model training is not described.
- **NLP:** Text normalization, 1-3 word n-grams, RapidFuzz matching (`ratio`, `token_sort_ratio`, `partial_ratio`), percentage label scoring, and keyword suggestion filtered by length (> 6) and frequency (>= 2).
- **Computer Vision:** PDF rasterization at 150 DPI, YOLO document cropping, OpenCV grayscale metrics (Laplacian variance, mean brightness, standard deviation, pixel-count resolution).
- **Data Engineering:** JPEG/PNG/PDF ingestion into uniform page buffers; per-region JSON config store with hot reload; append-only JSON audit log.
- **Cloud:** Azure AI Document Intelligence `prebuilt-read` as a per-request selectable OCR backend (documented as "higher accuracy cloud extraction"), with the local RapidOCR path providing operational continuity in restricted networks — POC logs "confirm successful local text line extraction on CPU without cloud credentials or network latency"; the POC also exercised an Azure custom extraction model (`anoud_ocr_uae_PR_sharjahold`).
- **MLOps:** Constants verified against code; POC-to-production traceability with documented parameter drift (50% -> 65%/90%); `/health` endpoint; hot-reloadable rule sets. No CI/CD, model registry, or monitoring stack described.
- **API Development:** FastAPI multipart endpoint with list[UploadFile], `root_path` proxy mounting, `custom_openapi` override for binary array schemas, versioned health endpoint, consistent response envelope.
- **Prompt Engineering:** Not demonstrated in this project.
- **System Design:** Hub-and-spoke microservice orchestration; `.env` routing table of 14 document-type mappings (13 distinct extractor URLs); crop-level concurrency; vendor-neutral OCR abstraction; region-extensible classification via Region Enums plus config files; reverse-proxied deployment (Nginx `:9000/ocr/` -> Uvicorn `:8083`); no database — the only persisted state is the local `output.json` audit file.

## 6. Detailed Technical Contributions

**Features implemented**

- `GET /health` returning `{"status": "OK", "service": "UAE OCR: Classifier + Extractor", "version": "1.0.0"}` — a health and smoke-test monitoring endpoint, no request parameters, no error responses declared.
- `POST /uae_allinone_ocr` (mounted under `/ocr`) with the multipart contract given in Section 3, stage 1: `files` (list[UploadFile], required), `reference_id` (string, required, external tracking/transaction reference), `ocr_type` (optional, `CPU_OCR` default / `AZURE`), `region` (optional, `UAE` default / `DOHA` / `OMAN` / `KUWAIT`, matching `labels_DB/<region>_labels.json`).
- Multi-page PDF handling via PyMuPDF at 150 DPI; per-page `page` index in results.
- Region-aware behavior: DOHA bypasses YOLO and supports multi-label `document_type` (labels >= 65% comma-joined).
- Police-report handling: any YOLO crop class containing `police` collapses all crops into a single full-frame `police_report` crop; PDF police reports OCR page 0 only.
- Quality feedback strings (`Blurry Image`, `Too Dark`, `Overexposed`, `Low Contrast`, `Low Resolution`), confidence warnings, and keyword suggestions returned per crop.
- Append-only audit to `output.json` with `timestamp`, `reference_id`, and the results array.

**Models used**

- `yolo-multipage-OCR-cls.pt` (Ultralytics YOLO segmentation), `imgsz=320`, `conf=0.25`, full-frame fallback.
- RapidOCR ONNX models via `rapidocr_onnxruntime` (CPU), one session per worker thread (`_ocr_thread_local`).
- Azure Document Intelligence `prebuilt-read`.
- Azure custom model `anoud_ocr_uae_PR_sharjahold` (POC cells 9-12), extracted `incidentType` and collision data from Sharjah police reports; abandoned in production.

**Pipelines built**

- `utils.py:detect_document_crops` / `_run_yolo_on_image` -> `utils.py:rapid_ocr` or `utils.py:azure_ocr` -> `utils.py:keyword_score` -> validity/quality inspection -> concurrent dispatch (`uae_allinone_ocr.py` line 179) -> aggregation.
- `utils.py:suggest_keywords`: n-grams from sample OCR text, filtered by token length (> 6) and frequency (>= 2), clustered via `partial_ratio`, surfaced in the Label Trainer interfaces.
- POC-to-production trail preserved in `test.ipynb` (71 cells): YOLO cropping / multi-page splitting (cells 4, 13, 31, 41-43) -> `detect_document_crops` / `_run_yolo_on_image`; Azure custom extraction (cells 9-12) -> abandoned; local RapidOCR CPU execution (cells 25, 38, 51, 60) -> `rapid_ocr` plus the `_ocr_thread_local` singleton added in production; RapidFuzz scoring (cells 50, 58-64) -> `keyword_score`; `predict_keyword` (cells 56-57) -> `suggest_keywords`; concurrent extractor dispatch with `ThreadPoolExecutor(max_workers=10)` and verified start/finish timing (cells 18-23, 27) -> `uae_allinone_ocr.py` line 179; cutoff thresholding at 50% (cells 63-69) -> production 65% / 90%.

**APIs integrated**

- 14 detected-document-type mappings resolved from `.env` `EXTRACTOR_URLS` variables, all on `http://192.168.86.213:8000`. UAE (10): Driving License Front -> `UAE_DRIVING_LICENSE_FRONT_URL` -> `/uae/license_front`; Driving License Back -> `/uae/license_back`; Resident ID Front -> `/uae/residence_front`; Resident ID Back -> `/uae/residence_back`; Vehicle License Front -> `/uae/vehicle_front`; Vehicle License Back -> `/uae/vehicle_back`; Sharjah old template -> `/police_report/rafid`; Sharjah new template -> `/police_report/rafid` (same endpoint as the old template); Abudhabi police report -> `/police_report/saaed`; Dubai police report -> `/police_report/dubai`. DOHA (4): Driving license -> `/doha/driving_license`; Resident license -> `/doha/residency_permit`; Vehicle license -> `/doha/vehicle_registration`; Doctor license -> `/doctor_license`. The 14 mappings resolve to 13 distinct URLs.
- Azure AI Document Intelligence (`prebuilt-read`).

**Data extraction methods**

- OCR text lines from RapidOCR or Azure; regex-based date extraction for expiry; downstream extractors return schema fields (example: `license_number`, `date_of_birth`, `expiry_date`) packaged into `extractor_response`.

**Validation logic**

- Config existence check (400) and load check (500) per region.
- Expiry: recognized dates compared to current system date -> `status` `Valid` / `Not valid`.
- Keyword acceptance: RapidFuzz score >= 80 per n-gram; label score < 65% -> `Add more keywords`; < 90% -> `Low confidence: X.X%` in `feedback`.
- Image quality thresholds: Laplacian var < 80, brightness < 50 or > 220, std dev < 30, w*h < 500,000 px.
- Dispatch: unmapped type -> `extractor_response` = `Endpoint URL not configured`; crop failure or unreachable extractor -> `status` `Error` or `Not valid` with the error description in `extractor_response`.

**Automation workflows**

- Single-call straight-through processing replacing manual routing; hot-reload of `labels_DB` JSON without restart; automated keyword suggestion for dictionary growth; per-transaction audit logging.

**Optimization techniques**

- Thread-local RapidOCR sessions to eliminate ONNX mutex contention; `ThreadPoolExecutor(max_workers=10)` for concurrent crop OCR/classification/dispatch; YOLO inference at 320 px; CPU-only PyTorch wheels and `opencv-python-headless` (lean server footprint, inferred); bounded 3.0 s / 30.0 s timeouts so slow extractors cannot hold worker threads indefinitely (inferred).

**Performance improvements**

- Notebook cell outputs record YOLO cropping at 0.125 s per single image and 1.074 s per PDF page (POC measurements). No production throughput or latency figures are recorded.

## 7. Challenges and Solutions

1. **Multiple documents per image and multi-page PDFs (technical).** Solution: PyMuPDF rasterizes each page at 150 DPI, then YOLO `yolo-multipage-OCR-cls.pt` isolates crops; if no box passes `conf=0.25`, the full frame is evaluated. Alternatives (analyst suggestion): contour/edge-based segmentation, fixed grid splitting, or per-page classification without cropping, which fails on mixed sheets.

2. **OCR vendor dependence and restricted networks (business).** Solution: runtime `ocr_type` switch between local RapidOCR and Azure `prebuilt-read` ("vendor-neutral OCR redundancy"). Alternatives: Tesseract or PaddleOCR as the local engine (analyst suggestion).

3. **ONNX runtime mutex contention under concurrency (technical).** Solution: thread-local singleton `_ocr_thread_local` so each of the 10 workers owns a session. Alternatives: a process pool or a single-session request queue (analyst suggestion), each adding memory or latency.

4. **Classifying document type across regions with tunable rules (technical).** Solution: fuzzy n-gram keyword matching (cutoff 80) against per-region JSON dictionaries, hot-reloadable and extendable through `suggest_keywords`. An Azure custom extraction model (`anoud_ocr_uae_PR_sharjahold`) was trialled and abandoned (Section 6). Alternatives: an image-level CNN classifier, an Azure custom classification model, or a fine-tuned text classifier (analyst suggestion), all requiring labeled data per region.

5. **Low-quality document submissions (business).** Solution: OpenCV guardrails (blur, exposure, contrast, resolution) return actionable feedback for early rejection. Alternative: automatic enhancement (deskew, contrast normalization) before OCR (analyst suggestion).

6. **Downstream extractor failures (technical).** Solution: `.env` routing with `Endpoint URL not configured` for unmapped types, 3.0 s / 30.0 s timeouts, and inline `Error` statuses so one bad crop does not fail the request. Alternatives: retries with backoff, circuit breaker, async HTTP client (analyst suggestion).

7. **Regional document variance (business).** Qatar cards show front and back on one sheet. Solution: DOHA bypasses YOLO and comma-joins labels >= 65%; OMAN and KUWAIT are pre-registered so only a labels JSON is needed. Alternative (analyst suggestion): a DOHA-specific YOLO model trained on two-sided sheets.

**Documented limitations and drift:**

- Police-report PDFs are classified from page 0 only ("any secondary pages of the PDF are not processed for keyword classification").
- Any `police` crop discards all other crops on the sheet and replaces them with `{'class': 'police_report', 'file_bytes': file_bytes}`, so a license bundled with a police report is lost (inferred consequence).
- POC tested a 50% cutoff; production enforces 65% (suggestions) and 90% (confidence warning).
- All dependencies unpinned in `requirements.txt` (every row of the technology stack table reads "Unpinned"); every extractor URL in `.env` points to the single host `192.168.86.213:8000`, so that host is a single point of failure (inferred).
- Audit persistence is a local `output.json` file; no authentication, rate limiting, or retry policy described; expiry relies on host system time and regex parsing.
- OMAN and KUWAIT are pre-registered as regions and accepted as `region` values, but the routing table contains extractor mappings only for UAE and DOHA, so those jurisdictions would classify without a downstream extractor until URLs are added (inferred from the routing table).
- For DOHA the classifier can emit a comma-joined multi-label `document_type`, while the routing table keys on single document types; how a combined type resolves to an extractor is not described (inferred gap).
- The document records no evaluation set, so the fuzzy cutoff (80), the 65%/90% thresholds, and the OpenCV quality thresholds are unvalidated against measured accuracy (inferred).

## 8. Impact Analysis

- **Business impact:** One REST call unifies segmentation, classification, validation, and extraction for core policy, underwriting intake, and claims portals, with a standardized contract across UAE emirates and Qatar. The document explicitly states: "Quantified business metrics — including throughput rates, cost savings percentages, ROI figures, and empirical accuracy rates — are not recorded in the repository."
- **Productivity improvements:** Eliminates multi-step manual routing; operators tune keyword rules via JSON hot-reload without restarts; keyword suggestion tooling shortens dictionary maintenance. Qualitative only.
- **Accuracy improvements:** No empirical accuracy figures. Notebook examples record fuzzy label scores of 83.3% (Resident ID Front) and 66.7% (Resident ID Back) on sample OCR text; these are illustrative POC outputs, not evaluation results.
- **Cost savings:** Not recorded. Qualitatively, the local RapidOCR CPU path avoids cloud OCR dependence and GPU hardware (inferred; not quantified).
- **Time savings:** POC cell outputs record YOLO cropping at 0.125 s per single image and 1.074 s per PDF page. No end-to-end latency or throughput figures exist.
- **User benefits:** Upstream systems receive per-document type, extracted fields, validity status, quality feedback, confidence, and suggestions in a single response; technical operations and management review teams get `/health` and an `output.json` audit trail for operational oversight.
- **Benefits as enumerated in the document.** Operational: (1) Consolidated Processing Pipeline — eliminates multi-step manual routing by unifying segmentation, classification, validation, and field extraction into a single REST call; (2) Pre-Extraction Quality Guardrails — detects blur, low resolution, and exposure anomalies before forwarding, allowing callers to reject unreadable documents early; (3) Automated Document Expiry Verification — evaluates expiration dates against system time without human intervention; (4) Thread-Safe Local Concurrency — isolates RapidOCR ONNX sessions per worker thread to avoid mutex serialization bottlenecks during high-volume ingestion; (5) Zero-Downtime Rule Tuning — hot-reloads JSON keyword sets from `labels_DB` without application restarts. Strategic: (6) Modular Regional Scalability — new jurisdictions (OMAN, KUWAIT) are pre-registered in Region Enums and enabled by supplying `labels_DB` JSON dictionaries; (7) Vendor-Neutral OCR Redundancy — dual local CPU (RapidOCR) and cloud (Azure) OCR ensures continuity in restricted networks; (8) Standardized Upstream Integration — a single consistent contract for diverse identity and police report documents across multiple emirates. All eight are stated qualitatively; the document adds that quantified metrics are "Not specified in the project".

## 9. Interview Discussion Points

- Why rule-based fuzzy keyword classification instead of a trained classifier: zero-downtime tuning and config-driven regional extensibility; acknowledge the accuracy ceiling and the 65%/90% thresholds.
- The thread-local RapidOCR session design: what mutex contention looked like, why thread-local beats a shared session, memory trade-off with 10 workers.
- YOLO parameters (`imgsz=320`, `conf=0.25`), the full-frame fallback, and why DOHA bypasses YOLO for Qatar cards.
- The dual OCR abstraction: when to select `AZURE` vs `CPU_OCR`, cost and data-residency, and how the downstream interface stays identical.
- The police-report crop-discard rule and first-page-only PDF behavior as known limitations, and what you would change.
- The `.env` routing table for 14 extractors, error semantics (`Endpoint URL not configured`, inline `Error` vs HTTP 400/500), the 3.0 s / 30.0 s timeouts, and thread-pool behavior when several extractors are slow.
- OpenCV quality thresholds (Laplacian variance 80, brightness 50/220, std dev 30, 500,000 px) and how they were chosen (the document records no evaluation set).
- POC-to-production traceability: what was carried forward, what was abandoned (`anoud_ocr_uae_PR_sharjahold`), and the 50% -> 65%/90% threshold drift.
- Production hardening gaps (unpinned dependencies, file-based audit, no auth, no retries, a single extractor host at `192.168.86.213:8000`) and what a v2 would look like.
- The `custom_openapi` override: why FastAPI's default schema for `list[UploadFile]` needed `format: binary`, and how `root_path='/ocr'` keeps generated URLs correct behind the Nginx `:9000/ocr/` -> `:8083` proxy.
- Why the hub POSTs raw crop bytes to extractors instead of passing the OCR text it already has: extractor autonomy and format-agnostic contracts versus paying for OCR twice per document (inferred trade-off) — and what an OCR-text-passing contract would change.
- Why both Sharjah templates (old and new) are classified separately but route to the same `/police_report/rafid` endpoint: classification granularity for audit and future divergence versus routing granularity today (inferred).
- Ingestion and rasterization choices: 150 DPI (matrix 150/72) as the legibility-versus-memory point for card-sized documents, and YOLO `imgsz=320` as the speed-versus-small-text trade-off on CPU (inferred rationale; the document records the values, not the reasoning).
- Error-handling taxonomy: HTTP 400 for a missing region config versus HTTP 500 for a config load failure, against per-crop inline `Error` / `Not valid` statuses that keep one bad document from failing a whole batch.
- Region extensibility in practice: OMAN and KUWAIT are accepted region values with no extractor URLs yet — what the rest of the rollout for a new jurisdiction actually involves.
- The DOHA comma-joined multi-label `document_type` versus a routing table keyed on single types, and how you would resolve a two-sided Qatar card to one or two extractor calls.
- How you would build an evaluation set to measure classification accuracy and end-to-end latency, since the document records none — labelled crops per region, confusion analysis on the fuzzy scorer, and per-stage timing.
- What the POC proved and what it did not: cell-level traceability (71 cells) is strong on feasibility and timing (0.125 s / 1.074 s YOLO cropping) but has no held-out accuracy measurement.

## 10. Architecture Explanation Points

Start with the entry: Nginx on port 9000 proxies `/ocr/` to a FastAPI/Uvicorn process on 8083, mounted with `root_path='/ocr'`. One endpoint, `POST /uae_allinone_ocr`, takes files plus `reference_id`, `ocr_type`, and `region`. Draw the pipeline left to right: (1) PyMuPDF turns PDFs into 150 DPI page images; (2) a YOLO segmentation model cuts each page into document crops, with a full-frame fallback and a DOHA bypass; (3) each crop enters a 10-thread pool where OCR runs locally (RapidOCR, thread-local ONNX session) or on Azure `prebuilt-read`; (4) the text is n-grammed and fuzzy-matched against a per-region JSON keyword dictionary to yield a `document_type` and percentage confidence; (5) regex expiry checks and OpenCV quality metrics attach `status`, `feedback`, and `suggestions`; (6) the type is looked up in the `.env` routing table and crop bytes are POSTed to the matching extractor with 3 s / 30 s timeouts; (7) extractor fields are merged into `results[]`, appended to `output.json`, and returned.

The seven named components map onto that flow: FastAPI Gateway Layer (REST entry point, multipart validation, request UUIDs, concurrency orchestration), YOLO Segmentation Engine (bounding boxes for identity cards and licenses, bypassed for DOHA), OCR Processing Layer (thread-local RapidOCR on CPU or Azure `prebuilt-read`), Fuzzy Keyword Classifier (normalization, 1-3 word n-grams, percentage scores against regional JSON definitions), Validation & Quality Inspector (regex expiry versus system date, OpenCV grayscale blur/contrast/brightness/resolution), Extractor Forwarding Dispatcher (`.env` routing over `ThreadPoolExecutor` worker threads), and Audit & Persistence Store (append of transaction, timestamp, reference ID, and result array to `output.json` on local disk). The service holds no database.

Key design decisions: config-driven classification for hot-reload and regional expansion; dual OCR for vendor neutrality; crop-level concurrency with thread-local engines; externalized extractor routing; a stable response envelope (`source_id`, `reference_id`, `timestamp`, `results[]`) so callers integrate once for every document type; and error containment — HTTP 400/500 only for region-config problems, everything per-document surfaced inline as `status` / `feedback` / `suggestions`. Trade-offs: keyword matching is cheap and tunable but less accurate than a trained classifier and its thresholds (80 cutoff, 65%, 90%) are unvalidated; thread-local sessions cost memory across 10 workers; the hub adds a network hop and a second OCR pass per document because crop bytes rather than OCR text are forwarded (inferred); DOHA's YOLO bypass trades segmentation for whole-sheet multi-label classification; the 3.0 s / 30.0 s timeouts bound worst-case latency but a slow extractor still occupies a pool thread for up to 33 s; `imgsz=320` favors CPU throughput over small-text recall; and the police-report collapse rule is a deliberate simplification that sacrifices any other document on the same sheet. Next improvements: pin dependencies, replace `output.json` with a database, add retries/circuit breakers and an async HTTP client, OCR all pages of police-report PDFs, route crops independently when a police report is present, resolve DOHA multi-label types to explicit extractor calls, spread extractors beyond the single `192.168.86.213:8000` host, supply `labels_DB` dictionaries and extractor URLs for OMAN and KUWAIT, add authentication and metrics, and build an evaluation set to measure classification accuracy and end-to-end latency.
