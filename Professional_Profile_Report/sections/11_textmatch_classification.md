# Project 11: Document Detection API (TextMatch Classification)

## 1. Project Overview

- **Project name:** TextMatch Document Detection API (document no. AT/classification/V2, dated 01-Sep-2026). Service self-identifies at `/health` as "TextMatch Document Detection API", version "1.0.0". Note the mismatch between the document's `V2` revision marker and the hard-coded `1.0.0` the service reports; the document gives no changelog or release history, so what changed in V2 is not recorded.
- **Business domain:** Insurance document intake for Qatar Insurance Group. The service triages customer-uploaded identity and motor documents (driving licenses, Emirates ID / Resident ID front and back, vehicle licenses, police reports) across Abu Dhabi, Dubai, Sharjah and Qatar, with regional keyword configurations for UAE, DOHA, OMAN and KUWAIT.
- **Problem statement:** Upstream automated client applications, customer onboarding portals, mobile upload backends and document gateway microservices (the four target-user classes the document names) need to know, immediately at upload time, (a) what kind of document a file contains, (b) whether it has expired, and (c) whether the image is good enough to process, before the file is routed to human underwriters or to the heavier field-extraction models. Running the full extractor suite for that first check is slow and wastes compute (inferred from the document's "fast turnaround for intake validation" and "without waiting for downstream extractors"); a single photo or scanned PDF may also contain several documents that must be separated first (the document refers to "multi-document camera captures or scanned files").
- **Project objective:** Build a lightweight, high-throughput FastAPI microservice that does detection only: split multi-document images and multi-page PDFs into individual document crops with a YOLO segmentation model, OCR each crop locally (or via Azure on demand), classify the crop by fuzzy keyword overlap against regional label sets, verify expiry dates, score image quality, persist an audit record, and return a uniform JSON contract, with no field extraction and no calls to downstream systems.
- **Expected business outcome:** Rapid document triage at intake, automated re-capture prompts for blurry/dark/low-contrast uploads, reduced Azure Document Intelligence transaction costs by defaulting to local CPU OCR, an audit trail in `output.json` for quality tracking and regulatory auditing, and a decoupled architecture in which detection can scale independently of extraction under burst traffic. The document states explicitly that throughput rates, cost-savings percentages, ROI and empirical accuracy rates are not recorded in the repository.

## 2. Role and Responsibilities

**Core responsibilities (inferred from what was built)**

- Designed the triage-only microservice architecture that deliberately separates classification from the full extractor suite so detection can scale independently.
- Built the FastAPI application (`document_detection_api.py`): `POST /detect` and `GET /health`, multipart parameter validation, request-tracking UUID (`source_id`) generation, `root_path='/detection'` for Nginx reverse-proxy routing, and a custom `get_openapi` override so Swagger renders binary array file uploads correctly.
- Integrated an Ultralytics YOLO segmentation model (`yolo-multipage-OCR-cls.pt`) for document region detection and cropping (`utils.py:detect_document_crops`, `imgsz=320`, `conf=0.25`).
- Implemented multi-page PDF ingestion with PyMuPDF (fitz) rendering each page to an RGB buffer at 150 DPI (matrix 150/72) before segmentation.
- Implemented a pluggable OCR layer with runtime selection between local CPU RapidOCR (`CPU_OCR`, default) and Azure AI Document Intelligence `prebuilt-read` (`AZURE`), including a thread-local RapidOCR singleton (`_ocr_thread_local`) in `utils.py:rapid_ocr`.
- Designed the fuzzy keyword classifier (`utils.py:keyword_score`): OCR text normalization, 1-3 word n-gram generation, RapidFuzz `token_sort_ratio` for multi-word phrases and `ratio` for single words with an 80 cutoff, and percentage match scoring against `labels_DB/<region>_labels.json`.
- Implemented validity and quality inspection: regex date parsing against the current system date for expiry, and OpenCV grayscale metrics for blur, darkness, over-exposure, low contrast and resolution.
- Built concurrent crop processing with a `ThreadPoolExecutor` (`max_workers=10`) and an append-only audit store (`output.json`) recording timestamp, reference ID and result array per transaction.

**Supporting tasks (inferred)**

- Authored regional keyword configuration files (`labels_DB/UAE_labels.json`, DOHA, OMAN, KUWAIT) that drive classification.
- Ran proof-of-concept experimentation in `test.ipynb` (71 cells): YOLO cropping and multi-page splitting trials (cells 4, 13, 31, 41-43; the recorded outcome is "Validated bounding box extraction" alongside the stored duration figures), RapidOCR CPU benchmarking (cells 25, 38, 51, 60; stored logs confirm local text-line extraction on CPU without cloud credentials), RapidFuzz n-gram scoring trials (cells 50, 58-64; n-gram matching evaluated on OCR lines against label dictionaries), and classification cutoff calibration (cells 63-69; a 50% acceptance cutoff was tested for label candidate matching).
- Recorded a PoC-to-production traceability table mapping each notebook trial to the production function that carries it (`utils.py:detect_document_crops`, `utils.py:rapid_ocr`, `utils.py:keyword_score`, and the thresholds in `document_detection_api.py`), including an explicit "carried into production with drifted parameters" entry for the cutoff change.
- Defined the development launch path (`python document_detection_api.py`, Uvicorn on port 8081 with `reload=True`) and the production launch path (`uvicorn document_detection_api:app --host 0.0.0.0 --port 8081`) and the Nginx gateway mapping (`http://<server-host>:9000/detection/` to internal port 8081).
- Defined error handling: HTTP 400 for a missing regional config, HTTP 500 for a config load failure, and per-crop inline `status: "Error"` with the exception string in `feedback`.
- Produced the technical documentation (architecture, endpoint reference, verified implementation constants, PoC-to-production traceability table).

**Estimated ownership level: AI Engineer (end-to-end owner of a single production microservice; inferred)**

Rationale: the document describes one cohesive service that the candidate carried from notebook experimentation (`test.ipynb`) into a production-launched FastAPI app behind an Nginx gateway, spanning six components (endpoint layer, YOLO segmentation, OCR layer, fuzzy classifier, validation/quality inspector, audit store) and two external dependencies (a local YOLO model, Azure Document Intelligence). The candidate made architecture-level decisions (triage-only decoupling, pluggable OCR to control cloud cost, thread-local OCR singletons for concurrency), which is Senior-leaning. However, there is no evidence of team leadership, containerization, CI/CD or observability beyond `/health` and a local JSON audit file; PyTorch is unpinned; and the notebook cutoff (50%) drifted from production (65%/90%) without a documented recalibration. Within the broader 15-project QIG suite this service is described as a lightweight complement to "the full extractor suite", so AI Engineer with full ownership of this component is the defensible label; Senior AI Engineer is arguable when the suite is considered as a whole.

## 3. End-to-End Workflow

**Processing stages**

1. **Ingest:** Client sends `multipart/form-data` to `POST /detect` with `files` (one or more JPEG/PNG images or multi-page PDFs), required `reference_id`, optional `ocr_type` (`CPU_OCR` default or `AZURE`), optional `region` (`UAE` default, `DOHA`, `OMAN`, `KUWAIT`).
2. **Config load and validation:** The service resolves `labels_DB/<region>_labels.json`; a missing file returns HTTP 400 `{"detail": "Config not found for region: <region_code>"}`, a load failure returns HTTP 500 `{"detail": "Config Error: <exception>"}`. A request-tracking UUID (`source_id`) is generated.
3. **Rasterization:** PDF inputs are rendered page by page to RGB buffers at 150 DPI with PyMuPDF (matrix 150/72); image inputs skip this step (inferred from the document's "rasterized if necessary").
4. **Segmentation:** Each page/image is passed through `yolo-multipage-OCR-cls.pt` (Ultralytics YOLO, `imgsz=320`, `conf=0.25`) to find bounding boxes for identity cards and licenses; each box becomes an isolated crop.
5. **Concurrent crop processing:** Crops are dispatched to a `ThreadPoolExecutor` with 10 workers. Each worker runs OCR (thread-local RapidOCR on CPU, or Azure `prebuilt-read` on the image bytes), normalizes the text, builds 1-3 word n-grams, and computes percentage keyword-match scores per label with RapidFuzz (`token_sort_ratio` / `ratio`, cutoff 80).
6. **Classification decision:** The best-scoring label becomes `document_type` (inferred; the document states percentage scores are computed per label but does not spell out the selection rule); a score below 90% appends `Low confidence: X.X%` to `feedback`; a score below 65% sets `suggestions` to `Add more keywords`, otherwise `All good`.
7. **Validity check:** Regex date parsing extracts expiry dates from the OCR text and compares them with the current system date to set `status` (e.g., `Valid`).
8. **Quality check:** OpenCV grayscale metrics flag `Blurry Image` (Laplacian variance < 80), `Too Dark` (brightness < 50), `Overexposed` (brightness > 220), `Low Contrast` (std dev < 30) and `Low Resolution` (w*h < 500,000 px) into `feedback`.
9. **Audit and respond:** The transaction (timestamp, reference ID, result array) is appended to `output.json`; the consolidated record (`source_id`, `reference_id`, `timestamp` in `YYYY-MM-DD HH:MM:SS` form per the documented example `2026-08-27 07:05:14`, `results[]` with `filename`, `document_type`, `status`, `feedback`, `confidence`, `suggestions`) is returned as JSON. The documented example shows `confidence` as an integer percentage (`94`), `feedback` as an empty string when no issue is flagged, and `document_type` values such as `Driving License Front`. Any crop-level exception yields `status: "Error"` with the error string in `feedback` rather than failing the whole request.

**Data flow**

Multipart bytes -> PyMuPDF RGB page buffers / decoded images -> YOLO bounding boxes -> per-crop image arrays -> OCR text lines -> normalized n-grams -> per-label percentage scores -> classification, validity and quality flags -> result dicts -> appended to `output.json` on local disk and returned as the JSON response body.

**Integrations and dependencies**

- Ultralytics YOLO with local weights `yolo-multipage-OCR-cls.pt` on PyTorch CPU wheels (torch, torchvision, unpinned, CPU extra-index-url).
- RapidOCR (`rapidocr_onnxruntime`) for local CPU OCR; Azure AI Document Intelligence `prebuilt-read` as the cloud alternative.
- PyMuPDF (fitz) for PDF rasterization; OpenCV (`opencv-python-headless`) for quality metrics; RapidFuzz for fuzzy scoring.
- FastAPI + Uvicorn [standard] as the web layer; Nginx as the reverse proxy (port 9000 -> 8081, path `/detection/`).
- Regional configuration in `labels_DB/<region>_labels.json`; audit persistence in `output.json`.
- Security and credentials: the document describes no authentication, authorization, rate limiting or TLS for the service or the Nginx layer, and names no mechanism for supplying Azure Document Intelligence endpoint/key configuration. Treat both as "not stated in documentation" rather than as absent by design.

**Flow line**

The document's own Figure 1 ("TextMatch Document Detection API: End-to-End Architecture") draws five boxes: Client/Caller uploads files -> YOLO Cropper splits documents -> OCR Engine extracts text -> Rapid-Fuzzy validates -> JSON returned. The expanded flow below adds the deployment and side-check detail stated elsewhere in the document.

`Client -> Nginx (:9000/detection/) -> FastAPI (:8081, POST /detect) -> PyMuPDF rasterize (150 DPI) -> YOLO crop (imgsz=320, conf=0.25) -> ThreadPoolExecutor(10) -> RapidOCR (CPU) | Azure prebuilt-read -> RapidFuzz keyword_score (cutoff 80) -> Regex expiry + OpenCV quality -> output.json -> JSON`

**Architecture Summary**

The service is a FastAPI microservice (single process, inferred: only one Uvicorn launch command is documented) with a stage-per-component pipeline. It is intentionally scoped to detection only: it never extracts field values and never calls downstream systems, which the document credits with "fast turnaround for intake validation" and removes coupling to the heavier extractor models. Compute-heavy stages are arranged for CPU execution: YOLO runs at a 320 px inference size, OCR defaults to an ONNX-runtime RapidOCR instance held as a thread-local singleton so ten worker threads can OCR crops concurrently without re-initializing the engine (the reuse rationale is inferred; the document states only that the singleton is thread-local), and Azure OCR is an opt-in per-request switch. Classification is configuration-driven rather than model-driven: the fuzzy keyword classifier reads regional JSON label sets, so supporting a new region or document type is a config change rather than a retraining exercise (inferred consequence; the document describes the config files but not this extensibility benefit). Validity and quality checks are deterministic OpenCV and regex logic with explicit, documented thresholds. All transactions are appended to a local `output.json` for audit, and the whole service sits behind Nginx under a `/detection` root path.

## 4. Technologies and Tools Used

| Category | Technologies (as stated in the document) |
| --- | --- |
| Programming languages | Python (service entry point `document_detection_api.py`, `utils.py`, `test.ipynb`) |
| Frameworks | FastAPI (web framework, `root_path='/detection'`, custom `get_openapi`); Uvicorn [standard] (ASGI server, port 8081); PyTorch (torch, torchvision, unpinned CPU wheel via extra-index-url) |
| Libraries | Ultralytics YOLO; OpenCV (`opencv-python-headless`); PyMuPDF (`pymupdf` / `fitz`); RapidOCR (`rapidocr_onnxruntime`); RapidFuzz (`rapidfuzz`); `concurrent.futures.ThreadPoolExecutor`; regular expressions; UUID generation. No versions pinned in the document. |
| AI/ML models | `yolo-multipage-OCR-cls.pt` (local Ultralytics YOLO segmentation model, imgsz=320, conf=0.25); RapidOCR ONNX models (CPU); Azure AI Document Intelligence `prebuilt-read`. The technology-stack table records the YOLO entry as a configuration value (`model_name: yolo-multipage-OCR-cls.pt`), implying the weights are referenced by a configurable name (inferred). Weight provenance (custom-trained vs pretrained), training data and class list are not stated in documentation. |
| OCR tools | RapidOCR (default, local CPU); Azure AI Document Intelligence prebuilt-read (optional, cloud) |
| Databases | Not stated in documentation (persistence is an append-only `output.json` file on local disk) |
| Cloud platforms | Microsoft Azure (Azure AI Document Intelligence only; hosting platform not stated) |
| APIs | Own REST API: `GET /health`, `POST /detect` (served under `/detection/` via Nginx); Azure Document Intelligence API (consumed) |
| DevOps tools | Nginx reverse proxy (port 9000 -> 8081). No CI/CD, containerization or monitoring tooling stated beyond the `/health` endpoint intended for container healthchecks. No authentication, authorization, rate limiting or TLS termination is described for the service or the Nginx layer, and no Azure credential/configuration mechanism is named - all "Not stated in documentation". |
| Version control | Not stated in documentation (the text refers to "the repository" but names no VCS) |
| Deployment tools | Uvicorn (`uvicorn document_detection_api:app --host 0.0.0.0 --port 8081`); Nginx gateway |
| Document processing tools | PyMuPDF (PDF rasterization at 150 DPI); OpenCV (grayscale quality metrics); Ultralytics YOLO (document cropping) |
| Automation tools | Not stated in documentation (in-process automation only: ThreadPoolExecutor with 10 workers; Swagger UI version not specified, the document noting that FastAPI serves Swagger UI assets dynamically without an explicit frontend version in project manifests) |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Modular Python service (`document_detection_api.py` + `utils.py`); thread-safe engine reuse via thread-local singletons; explicit error contracts (400/500 plus per-item inline errors so one bad crop does not fail a batch); externalized configuration in `labels_DB/`; clearly separated development (`reload=True`) and production launch commands.
- **AI Engineering:** Assembling a CV model, two OCR engines and a fuzzy text classifier into one inference pipeline with runtime engine selection; carrying PoC notebook parameters (imgsz, conf, DPI, cutoffs) into production code with a traceability table.
- **Machine Learning:** Applying a local Ultralytics YOLO segmentation model at inference (`imgsz=320`, `conf=0.25`) and calibrating decision thresholds empirically (50% notebook cutoff, 65% and 90% in production). No model training is described in the document, and whether the weights were custom-trained or pretrained is not stated - the technology stack records only "Local model (model_name: yolo-multipage-OCR-cls.pt)".
- **NLP:** OCR text normalization, 1-3 word n-gram generation, fuzzy string matching with RapidFuzz `token_sort_ratio` and `ratio` (cutoff 80), percentage keyword-overlap scoring against per-region label dictionaries, regex date extraction for expiry.
- **Computer Vision:** YOLO document region detection and cropping from multi-document captures; OpenCV grayscale image-quality assessment (Laplacian-variance blur, brightness-based darkness/over-exposure, standard-deviation contrast, pixel-count resolution); PDF-to-image rasterization at 150 DPI.
- **Data Engineering:** Multi-format ingestion (JPEG, PNG, multi-page PDF) into a uniform crop stream; append-only JSON audit log with timestamp and reference ID; regional JSON config management.
- **Cloud:** Optional integration with Azure AI Document Intelligence `prebuilt-read` as a per-request OCR backend; cost-aware defaulting to local CPU OCR.
- **MLOps:** Limited: health endpoint for container healthchecks, PoC-to-production parameter traceability, local model artifact management. No CI/CD, model registry, or monitoring described.
- **API Development:** FastAPI multipart endpoint accepting `list[UploadFile]` plus form fields; UUID request tracking; `root_path` for reverse-proxy deployment; custom OpenAPI schema override for binary array uploads; health endpoint returning service name and version.
- **Prompt Engineering:** Not demonstrated in this project (no LLM is used; classification is fuzzy keyword matching).
- **System Design:** Triage-only decoupling from the extractor suite so detection scales independently; pluggable OCR abstraction; concurrent per-crop processing with a bounded 10-worker pool; configuration-driven multi-region support; Nginx gateway routing.

## 6. Detailed Technical Contributions

**Features implemented**

- `GET /health` returning `{"status": "OK", "service": "TextMatch Document Detection API", "version": "1.0.0"}` for monitoring and container healthchecks; takes no request parameters and declares no error responses.
- `POST /detect` accepting `files: list[UploadFile]` (required), `reference_id: string` (required), `ocr_type: string` (optional, `CPU_OCR` | `AZURE`), `region: string` (optional, `UAE` | `DOHA` | `OMAN` | `KUWAIT`), returning `source_id` (UUID), `reference_id`, `timestamp`, and `results[]` of `{filename, document_type, status, feedback, confidence, suggestions}`.
- Multi-document splitting and multi-page PDF ingestion; pluggable OCR; regional fuzzy classification; expiry verification; five image-quality flags; concurrent execution; audit persistence.
- FastAPI configured with `root_path='/detection'` and a `get_openapi` override to render binary array file uploads correctly in Swagger.
- Endpoint-layer duties the document assigns explicitly: exposing the two routes (published under the root path as `/detection/detect` and `/detection/health`), validating multipart parameters, and generating request-tracking UUIDs.
- The seven capabilities the document names as the delivered feature set: triage-only document classification, YOLO document splitting, multi-page PDF ingestion, pluggable OCR support, regional fuzzy matching, automated quality and expiry analysis, and concurrent worker execution with audit records serialized to `output.json`.
- Response-contract observations read off the documented schema (the document offers no commentary on them): `results[]` is a flat array whose entries carry `filename` but no crop index, page number or bounding-box coordinates, so multiple crops taken from a single uploaded file are not individually identifiable in the response; `confidence` appears as an integer (`94`) in the documented example while the confidence-warning string is formatted to one decimal (`Low confidence: X.X%`).
- Not implemented / not stated: no authentication or authorization on `POST /detect`, no upload size or file-count limit, no retention or rotation policy for `output.json`, and no defined response for a page in which YOLO detects no document or for an unsupported file type.

**Models used**

- `yolo-multipage-OCR-cls.pt` (Ultralytics YOLO, local weights) for identity-card and license bounding boxes, run at 320 px input with confidence cutoff 0.25.
- RapidOCR (`rapidocr_onnxruntime`) CPU models as the default text engine; Azure Document Intelligence `prebuilt-read` as the cloud alternative. PoC evidence: `test.ipynb` cells 25, 38, 51, 60 store logs confirming local text-line extraction on CPU without cloud credentials, carried into `utils.py:rapid_ocr`.

**Pipelines built**

- Ingest -> rasterize (PyMuPDF, 150/72 matrix) -> segment (YOLO) -> parallel OCR + keyword scoring (ThreadPoolExecutor, 10 workers) -> classification decision -> regex expiry check -> OpenCV quality check -> append to `output.json` -> JSON response. Key functions named in the document: `utils.py:detect_document_crops`, `utils.py:rapid_ocr`, `utils.py:keyword_score`.

**APIs integrated**

- Azure AI Document Intelligence `prebuilt-read` (dispatching crop image bytes when `ocr_type=AZURE`).
- Own service exposed through Nginx at `http://<server-host>:9000/detection/` mapped to internal port 8081.

**Data extraction methods**

- OCR line extraction per crop (RapidOCR or Azure); text normalization; 1-3 word n-gram generation; regex date parsing for expiry dates. No field-value extraction by design.

**Validation logic**

- Regional config existence check (HTTP 400) and load check (HTTP 500).
- Keyword score gates: RapidFuzz match accepted at ratio/token_sort_ratio >= 80; `suggestions = 'Add more keywords'` when score < 65% else `'All good'`; `feedback += 'Low confidence: X.X%'` when score < 90%.
- Expiry: regex-parsed dates compared with the current system date to derive `status`.
- Quality: `Blurry Image` if Laplacian variance < 80; `Too Dark` if brightness < 50; `Overexposed` if brightness > 220; `Low Contrast` if std dev < 30; `Low Resolution` if w*h < 500,000 px.
- Per-crop exception isolation: `status = 'Error'`, `feedback = <error string>`.

**Automation workflows**

- Automatic per-request audit append to `output.json` (timestamp, reference ID, result array).
- Automated re-capture prompting enabled downstream by the quality flags (the service emits the flags; the prompt itself is the caller's responsibility).

**Optimization techniques**

- Triage-only scope (no extraction, no downstream calls), which the document credits with "fast turnaround for intake validation".
- Small YOLO inference size (320 px); the CPU-speed rationale is inferred, the document records only the value.
- Thread-local RapidOCR singleton (`_ocr_thread_local`) under 10 concurrent workers; the avoid-per-call-initialization rationale is inferred, the document states only that the singleton is thread-local.
- Local CPU OCR as default, which the document states "minimizes outbound Azure Document Intelligence API transaction costs"; CPU-only PyTorch wheels via extra-index-url.
- Headless OpenCV build (`opencv-python-headless`); the dependency-footprint rationale is inferred, the document only names the package.

**Performance improvements**

- The document records PoC notebook timings only: 0.125 s for single images and 1.074 s for PDF pages in YOLO cropping trials (`test.ipynb` cells 4, 13, 31, 41-43). No production throughput or latency benchmarks are recorded.

## 7. Challenges and Solutions

1. **Multiple documents in one capture / multi-page PDFs (technical).** Customers upload photos containing several cards or scanned PDFs with many pages. Solution: PyMuPDF renders every page at 150 DPI, then `yolo-multipage-OCR-cls.pt` detects each document region and produces isolated crops that are processed independently. Alternatives: classical contour/edge-based card detection or processing whole pages without cropping (analyst suggestion); the document records that YOLO cropping was validated in the notebook and carried forward.

2. **Cloud OCR cost versus availability of a cloud engine (business and technical).** Every Azure Document Intelligence call is a billable transaction. Solution: a pluggable OCR layer with local RapidOCR on CPU as the default and Azure `prebuilt-read` selectable per request via `ocr_type`, which the document credits with reducing compute and cloud costs. Alternatives: Tesseract or PaddleOCR as local engines, or an automatic fallback to Azure when local confidence is low (analyst suggestion).

3. **Classifying noisy OCR output (technical).** OCR text from phone captures contains spelling errors and broken tokens, so exact keyword matching fails (problem framing inferred; the document describes the fuzzy-matching solution, not the failure mode it addresses). Solution: normalize text, build 1-3 word n-grams, and score with RapidFuzz `token_sort_ratio` (multi-word phrases) and `ratio` (single words) with an 80 cutoff, yielding percentage match scores per label. Alternatives: a trained text classifier or embedding similarity over OCR text, or an image-level classification head (analyst suggestion); those would need labelled data the document does not mention.

4. **Regional document diversity (business).** Documents differ across Abu Dhabi, Dubai, Sharjah, Qatar, Oman and Kuwait. Solution: keyword definitions live in `labels_DB/<region>_labels.json`, selected by the `region` parameter, giving a uniform response contract across regions. Alternative: a single merged label set, which would raise cross-region confusion (analyst suggestion).

5. **Throughput on CPU-only hardware (technical, challenge framing inferred).** Solution: a `ThreadPoolExecutor` with 10 workers processes crops concurrently, and RapidOCR is held in a thread-local singleton (`_ocr_thread_local`) so each thread reuses its own engine (reuse behavior inferred from "thread-local singleton"). Alternatives: async I/O for Azure calls, multiprocessing for CPU-bound OCR, or GPU inference (analyst suggestion).

6. **Poor-quality uploads reaching underwriters (business).** Solution: OpenCV grayscale metrics flag blur, darkness, over-exposure, low contrast and low resolution with fixed thresholds so callers can prompt re-capture at intake. Alternative: a learned image-quality model (analyst suggestion), at the cost of training data and latency.

7. **Threshold drift between PoC and production (documented limitation).** The notebook tested a 50% acceptance cutoff, while production enforces 65% for suggestions and 90% for confidence warnings; the document itself calls these "drifted parameters". The honest position is that the production values were chosen during hardening and lack a recorded recalibration study; a labelled validation set would be the fix.

8. **Audit persistence on local disk (documented design, inferred limitation).** All transactions append to `output.json`. This satisfies the audit requirement but is a single-file store on one host with no stated locking, rotation or query capability. Alternative: a database table or append-only log service keyed by `source_id` (analyst suggestion).

9. **Unpinned dependencies (documented limitation).** PyTorch/torchvision are unpinned (CPU extra-index-url) and no library versions are recorded, and the Swagger UI version is unspecified; reproducible builds would need a lock file (analyst suggestion).

10. **Swagger rendering of binary array uploads (technical).** FastAPI's default OpenAPI output did not format `list[UploadFile]` correctly for the documentation UI (inferred from the document's statement that the app "overrides get_openapi to format binary array file uploads correctly"). Solution: override `get_openapi` in the app. Alternatives: not stated in the document.

11. **No authentication, upload limits or data-retention policy described (documented gap).** The document specifies no authentication, authorization, rate limiting, TLS termination, maximum file size or maximum file count for `POST /detect`, and no retention, rotation or access control for `output.json` - even though every stored record derives from customer identity documents (Emirates/Resident IDs, driving licenses, police reports) for an insurance client. No solution is stated in the source; the honest interview position is to name the gap and propose gateway-level authentication, request size/count caps, and an encrypted, rotated audit store with a defined retention period (analyst suggestion).

12. **Undefined edge-case behavior (documented gap).** The document does not state what the service returns when YOLO detects no document region in a page, when an uploaded file is neither a supported image (JPEG/PNG) nor a PDF, or which `status` values exist beyond the documented `Valid` and `Error`. Nor does it state whether `output.json` writes are guarded against concurrent requests. Next step: define and document explicit no-detection, unsupported-format and status-vocabulary contracts, and serialize or lock the audit write (analyst suggestion).

## 8. Impact Analysis

The document states explicitly: "Quantified business metrics—including throughput rates, cost savings percentages, ROI figures, and empirical accuracy rates—are not recorded in the repository." All impact below is qualitative except where notebook figures are quoted.

The document lists six named benefits, all qualitative: four operational (Rapid Document Triage, Upfront Quality Assurance, Reduced Compute & Cloud Costs, Audit Trail Tracking) and two strategic (Decoupled Architecture, Unified Cross-Emirate Ingestion).

- **Business impact:** Instant identification of whether an upload is a driving license, Emirates ID, vehicle license or police report without waiting for downstream extractors; a uniform JSON contract across Abu Dhabi, Dubai, Sharjah and Qatar; an audit trail for quality tracking and regulatory auditing.
- **Productivity improvements (inferred):** Underwriters and specialized extractors receive only pre-classified, validity-checked and quality-screened documents; bad uploads are caught at intake rather than mid-workflow. The document states only that the service validates uploads "before routing to human underwriters or specialized extractors".
- **Accuracy improvements:** No production accuracy figures. PoC notebook fuzzy scores are recorded as examples (Resident ID Front: 83.3%, Resident ID Back: 66.7%).
- **Cost savings:** Qualitative only: defaulting to local RapidOCR CPU execution "minimizes outbound Azure Document Intelligence API transaction costs". The CPU-only PyTorch wheels imply no GPU hosting requirement (inferred; the document does not discuss hosting cost).
- **Time savings:** No production latency figures. Notebook cell outputs record 0.125 s for single images and 1.074 s for PDF pages in YOLO cropping trials.
- **User benefits:** Upstream portals and mobile backends can prompt customers to re-capture blurry, dark or low-contrast images immediately; callers get a `source_id` and `reference_id` for tracking; the decoupled service can scale independently under burst traffic.

## 9. Interview Discussion Points

- Why detection-only? Be ready to explain the decoupling rationale: fast intake validation, independent scaling under burst traffic, and no coupling to extractor models or downstream systems.
- Walk through the YOLO stage: why a segmentation model for multi-document captures, why `imgsz=320` and `conf=0.25` on CPU, and what happens when YOLO misses a document or returns overlapping boxes.
- PDF handling: 150 DPI (matrix 150/72) is the documented rendering value, but the document records no rationale for it - be ready to justify it yourself as a legibility-versus-rasterization-time trade-off (analyst framing), and to cite the notebook observation that PDF pages took 1.074 s versus 0.125 s for single images.
- Pluggable OCR: how the `ocr_type` switch is implemented, why RapidOCR is the default, when a caller should choose Azure `prebuilt-read`, and how you would add automatic fallback.
- Fuzzy classification design: text normalization, 1-3 word n-grams, `token_sort_ratio` versus `ratio`, the 80 cutoff, and how percentage scores map to `confidence`, `feedback` ("Low confidence: X.X%" below 90%) and `suggestions` ("Add more keywords" below 65%).
- Threshold drift: the notebook tested 50% while production uses 65%/90%. Expect to be asked how you would build a labelled validation set and calibrate thresholds properly, and how you would monitor score distributions in production.
- Concurrency: why a `ThreadPoolExecutor` with 10 workers rather than async or multiprocessing, the GIL implications for ONNX-runtime OCR, and why the RapidOCR engine is a thread-local singleton.
- Image-quality metrics: how Laplacian variance, brightness, standard deviation and pixel count are computed with OpenCV on the grayscale crop, how the thresholds (80, 50/220, 30, 500,000 px) were set, and their limits on different capture devices.
- Expiry verification: regex date parsing risks (day/month order, Arabic numerals, multiple dates on a card) and how `status` is derived against the system date.
- Error contract: HTTP 400 for a missing regional config, HTTP 500 for a config load failure, per-crop `status: "Error"` so a bad crop does not fail a batch.
- Operational gaps: `output.json` as the audit store, unpinned PyTorch, no containerization or CI described. Be candid, and describe what production hardening you would add.
- Deployment: FastAPI `root_path='/detection'`, Uvicorn on 8081, Nginx on 9000; the `get_openapi` override for binary array uploads.
- PoC-to-production traceability: be able to cite the `test.ipynb` evidence (cells 4, 13, 31, 41-43 for YOLO cropping timings; cells 25, 38, 51, 60 showing RapidOCR ran on CPU without cloud credentials; cells 50, 58-64 for the 83.3% / 66.7% Resident ID scores; cells 63-69 for the 50% cutoff) and explain why stored cell outputs are not a production benchmark and how you would turn them into one.
- Audit trail: what each `output.json` record holds (timestamp, reference ID, result array, `source_id`), how the document positions it for quality tracking and regulatory auditing, and its concurrency and rotation limits as a single local file.
- Request contract details: `files` as `list[UploadFile]` with JPEG, PNG and multi-page PDF; `reference_id` required; `ocr_type` and `region` optional with `CPU_OCR` and `UAE` defaults; `confidence` returned as an integer percentage (94 in the documented example) and `feedback` empty when nothing is flagged.
- Security and compliance, which the document does not cover at all: no authentication, authorization, rate limiting, upload size or file-count cap, TLS detail, or retention/PII policy is described, yet the service ingests customer identity documents and appends every result to an unrotated local `output.json`. Name the gap plainly and describe the controls you would add (auth at the Nginx gateway, size and count caps, encryption plus retention on the audit store, and a credential mechanism for the Azure endpoint/key, which is also unstated).
- Undefined behaviors worth pre-empting: what the response looks like when YOLO finds no document region in a page, what happens when a file is neither a supported image nor a PDF, whether concurrent requests can corrupt `output.json`, and what `status` values exist beyond `Valid` and `Error` - none are stated in the document.
- The model artifact itself: the document calls it a segmentation model, names it `yolo-multipage-OCR-cls.pt` (a classification-style suffix) and uses it to produce bounding boxes for identity cards and licenses. Expect questions on what the model actually is, how it was obtained or trained, its class list, and how it was evaluated - the document records none of this, only "Local model (model_name: yolo-multipage-OCR-cls.pt)".
- Response identity and traceability: `results[]` carries `filename` but no crop index, page number or bounding box, so a caller cannot tell which crop of a multi-document upload a given result describes. Offer this as a concrete, self-identified improvement.
- Service versioning: `/health` returns a hard-coded `"version": "1.0.0"` while the documentation is revision `AT/classification/V2` with no changelog. Be ready to say how you would tie the reported version to the release and what V2 changed.
- Where the service sits in the wider QIG estate: it is explicitly the lightweight counterpart to "the full extractor suite" and makes no downstream system calls, so be ready to describe the handoff - what the caller does with `document_type`, `status` and the quality flags before invoking an extractor.

## 10. Architecture Explanation Points

Start with the boundary: this is a triage microservice that answers three questions about an upload (what document, is it valid, is the image usable) and nothing else. Name the callers the document lists: upstream automated client applications, customer onboarding portals, mobile upload backends and document gateway microservices. Draw the client on the left hitting Nginx on port 9000 at `/detection/`, proxied to a FastAPI app on 8081. Inside the app draw a linear pipeline: PyMuPDF rasterizes PDF pages at 150 DPI; an Ultralytics YOLO model (`yolo-multipage-OCR-cls.pt`, 320 px, conf 0.25) crops each document out of the page; crops fan out to a 10-thread pool. Each thread runs OCR (thread-local RapidOCR on CPU by default, or Azure `prebuilt-read` if the caller asks), then a fuzzy keyword classifier that builds 1-3 word n-grams and scores them with RapidFuzz against a per-region JSON label set at an 80 match cutoff. Add two side checks: regex expiry parsing against today's date, and OpenCV quality flags for blur, darkness, over-exposure, contrast and resolution. Results fan in to one JSON record with a `source_id` UUID and the caller's `reference_id`, appended to `output.json` and returned.

Key design decisions to call out: decoupling detection from extraction so it scales independently and stays fast (the document's own "Decoupled Architecture" strategic benefit); making OCR pluggable so the caller chooses per request between the local CPU engine - the default, which the document credits with minimizing outbound Azure transaction costs - and the managed cloud engine (the document does not compare the two engines' accuracy, so do not claim Azure is more accurate); making classification config-driven so new regions and document types need a JSON edit rather than retraining (inferred consequence - the document describes `labels_DB/<region>_labels.json` but never claims this benefit); bounded concurrency with thread-local engines; and a single uniform JSON contract across Abu Dhabi, Dubai, Sharjah and Qatar (the document's "Unified Cross-Emirate Ingestion" benefit).

Trade-offs to acknowledge: fuzzy keyword matching is transparent and needs no training data but is sensitive to OCR quality and label curation (analyst assessment; the document states the mechanism, not the trade-off); fixed quality thresholds are simple but device-dependent; the local JSON audit file is easy but not scalable, not query-able and not stated to be concurrency-safe; production thresholds (65%/90%) drifted from the notebook (50%) without a recorded calibration study; the response identifies results only by `filename`, not by crop or page; the document describes no authentication, upload limits or retention policy for a service that ingests identity documents; and there is no measured accuracy or throughput.

What to improve next: a labelled validation set and threshold calibration; automatic Azure fallback on low local-OCR confidence; replace `output.json` with a database and define a retention/access policy for the identity data it holds; add authentication and upload size/count limits at the Nginx gateway; return a crop index and bounding box per result; define explicit no-detection and unsupported-format responses; pin dependencies and containerize; add metrics and score-distribution monitoring; consider an image-level classifier alongside keyword scoring for low-text documents.
