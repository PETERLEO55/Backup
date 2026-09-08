# Project 6: KYC Document Validation System (FastAPI / RapidOCR / DeepFace / OpenCV / Streamlit)

## 1. Project Overview

- **Project name:** KYC Document Validation System (document no. AT/KYC/V1, dated 01-Sep-2026), delivered for Qatar Insurance Group.
- **Business domain:** Insurance customer onboarding / Know-Your-Customer (KYC) identity-document verification. The bundled `document_anchors.py` covers UAE, Qatar/Doha, Oman and Kuwait documents (driving licences, residency permits, vehicle registrations) plus passports from India, Pakistan, Oman, Philippines, Egypt and Bangladesh, "reflecting a Gulf insurance/onboarding context".
- **Problem statement (inferred from the stated operational benefits):** Identity documents submitted during onboarding must be checked by hand for mandatory fields, expiry status, image quality, and whether the document photo matches the applicant. The document frames the pain as manual data entry, manual review and "human reading", with no programmatic hook for downstream CRM or policy platforms.
- **Project objective:** Build a Python application that accepts a document image and optionally a selfie, runs one or more of five validation modes — keyword matching, expiration-date detection, image-resolution checking, facial biometric comparison, and region-of-interest (ROI) text extraction — and returns structured JSON over a REST endpoint, with browser, Streamlit and desktop front-ends for internal operators.
- **Expected business outcome:** Reduce manual review via configurable keyword checks, give an immediate valid/expired signal, gate low-quality images before OCR, enable programmatic identity verification against a live photo, support field-level validation, and expose a REST API that CRM and policy platforms can call. The document states that quantified performance figures are not specified.
- **Target users:** "Internal" (stated verbatim under Target Users).
- **Technology header (as stated):** FastAPI / RapidOCR / DeepFace / OpenCV / Streamlit. The Technology Stack table lists twelve rows — FastAPI (Web Framework), Uvicorn (ASGI Server), Jinja2 via FastAPI (Template Engine), RapidOCR / ONNX Runtime (OCR Engine), DeepFace + ArcFace model (Face Verification), OpenCV `cv2` (Image Processing), RapidFuzz (Fuzzy Matching), Streamlit (Streamlit UI), PyQt5 (Desktop GUI), NumPy (Numerical Arrays), Pillow/PIL (Image I/O, Streamlit UI) and Python (Language) — and declares **no version for any of them**: every row reads "Not specified", with Uvicorn noted as "imported in main.py", FastAPI as "imported without version pin", and the Python row adding only that an interpreter at `C:\Users\peter.leo\.local\bin\python3.14.exe` "confirms 3.14 is present on this machine" (which is a fact about the documenting machine, not a declared project target).
- **Backend & API (as stated):** Framework FastAPI; Jinja2 templates directory `templates/` (`templates = Jinja2Templates(directory='templates')`). The source document's section 3 heading "Swagger UI Version" is left empty, so no Swagger/OpenAPI version is recorded.

## 2. Role and Responsibilities

**Core responsibilities (inferred from what was built):**

- Designed the synchronous request-response architecture: multipart form with a `type` parameter, FastAPI dispatch to a helper function, JSON response.
- Implemented `main.py` with four routes: `GET /`, `POST /upload`, `POST /process`, `GET /image/{filename}`.
- Built the five processing functions: `extract_text()` via RapidOCR (ONNX Runtime), `compare_faces()` via DeepFace ArcFace, `keyword_match()` with RapidFuzz pre-correction, `find_expiry()` via Python `re`, `check_resolution()` via OpenCV.
- Selected OCR engine and face model through experiments in `testing.ipynb`: EasyOCR (cells 14-15) vs RapidOCR (cells 16-18), InsightFace `buffalo_l` (cells 9-10) vs DeepFace (cell 3 `extract_faces`, production `verify`), `sentence_transformers` all-mpnet-base-v2 (cell 21) vs RapidFuzz (cell 20). The Azure Form Recognizer + RapidOCR **dual-engine** design lived in `KYC 19-03-2026/main-old.py`, not in the notebook; the notebook contributed only the Azure *pre-processing* steps (cells 4-8: resize, RGB conversion, JPEG save, EXIF inspection).
- Implemented the ROI pipeline with OpenCV (crop by `x, y, w, h`, persist to `cropped_images/`, OCR the crop).
- Built three front-ends: the primary HTML/JS + Jinja2 canvas UI (`templates/index.html`), a Streamlit UI (`streamlit.py`), and a PyQt5 desktop app (`pyqt_app.py`).
- Defined business thresholds: keyword pass >= 65 %, fuzzy correction partial_ratio >= 70, resolution gate 640 x 480, ROI minimum > 10 px per dimension.
- Iterated across versions (`KYC 19-03-2026` to `Final KYC 20-03-2026`): removed the `/api/` prefix, dropped the Azure dependency and `/cleanup`, inlined `utils.py` helpers into `main.py`. The earliest `utils-old.py` imported `document_anchors.DOCUMENT_ANCHORS` and its `similarity_search` returned a plain string (`'Confidence : X and Time taken : Y'`, line 91); it was superseded by `utils.py`, which returns a structured dict, and finally by helpers inlined in `main.py` with no `utils` import. `main-old.py` exposed `/api/upload`, `/api/process-ocr` (Azure + RapidOCR), `/api/extract-region`, `/api/image/{filename}` and `/api/cleanup`; production replaced these with `/upload`, `/process` and `/image/{filename}`. `main - Copy.py` is a backup in which `/process` was commented out and then reinstated; its handler matches production verbatim.
- Authored the system documentation (architecture, endpoint reference, constants, UI guide, experimentation log) (inferred; the document does not name its author).

**Supporting tasks:**

- UUID-named local storage in `uploads/` and `cropped_images/`.
- Edge-case handling in `/process`: missing second image, zero-dimension region, out-of-bounds region, invalid type, exception-to-JSON.
- HTML UI behaviours: immediate `POST /upload` on file selection, drag-to-select red rectangle on `<canvas>`, display-to-original coordinate scaling, auto-submit and clipboard copy for ROI text.
- Launch instructions exactly as documented: FastAPI + Jinja2 UI — `uvicorn main:app --reload` (Swagger at `localhost:8000/docs`); HTML UI — `uvicorn main:app --reload` (`localhost:8000`); Streamlit UI — `streamlit run streamlit.py`. No launch command is documented for the PyQt5 desktop app, and no production ASGI configuration (workers, TLS, process manager, container) appears anywhere — only the `--reload` development server.
- Curated `document_anchors.py` (GCC coverage) even though production `main.py` no longer imports it (authorship inferred: the document confirms the file exists and that `KYC 19-03-2026/utils-old.py` imported `document_anchors.DOCUMENT_ANCHORS`, but never names who wrote it).
- Maintained the experiment record in `16-03 Face similarity/testing.ipynb`: kernel `endtoend`, 28 code cells (cells 11-13 and 22-27 are empty; several other cells have no stored outputs), per-cell outcome and carried-into-production status.
- The documentation includes screenshot sections for the Initial UI, Expiration UI, Resolution UI, Face Similarity and ROI UI (headings present in the source; image content not reproduced in the text extract).

**Estimated ownership level: AI Engineer (inferred).**
Rationale: the evidence points to a single engineer owning the full stack — backend, three front-ends, CV/OCR logic, biometric verification, and the experimentation that drove model selection — across roughly a dozen components (four routes, five functions, three UIs, one notebook, several superseded modules). But the system has no database, no external service and no cloud deployment, the document mentions no CI/CD or test suite (inferred from silence), and it carries a documented unhandled `FileNotFoundError` on `GET /image/{filename}`, so it is a PoC-grade internal tool rather than a hardened platform. That places it above "Developer" (model and threshold choices were the candidate's) but below "Solution Architect" or "Technical Lead" (no distributed design or team coordination evidenced).

## 3. End-to-End Workflow

**Processing stages:**

1. **Client submission.** Operator uses the HTML/Jinja2, Streamlit or PyQt5 UI. In the HTML UI, choosing a file triggers `POST /upload`; the image is then previewed in an `<img>` element under a `<canvas>` overlay (served via `GET /image/{filename}` — inferred). The Streamlit UI is a self-contained browser UI with a multi-column layout, image display and result panels; the PyQt5 app is a desktop UI. Both carry duplicated helper logic (`keyword_match`, `find_expiry`, `check_resolution`, `compare_faces`); the document's Figure 1 routes all three clients through the FastAPI server, but whether Streamlit/PyQt5 call the API or run the helpers in-process is not stated explicitly.
2. **Mode selection.** A `<select>` offers Keyword Match, Expiration Date, Resolution Check, Face Similarity, Select Region (Auto-copy); it shows/hides the selfie input, keyword input and canvas.
3. **Dispatch.** `POST /process` receives `type`, `image1`, optional `image2`, `keywords`, `region_x/y/w/h` as multipart form data; bytes are read directly, so `/upload` need not be called first.
4. **Decode.** Images are loaded with `cv2.imread` / `cv2.imdecode` (the document states production "document loading uses cv2.imread / cv2.imdecode", in contrast with notebook cell 0 which used PIL); that the decode yields NumPy arrays is inferred from the stack table listing NumPy under "Numerical Arrays".
5. **Branch by type:**
   - `resolution` — flag "Low Resolution" if `min(w,h) < 480 OR max(w,h) < 640`.
   - `keyword_match` — RapidOCR lines -> RapidFuzz `partial_ratio >= 70` correction -> server-side comma split -> case-insensitive substring match -> `matched = percentage >= 65`.
   - `expiration` — RapidOCR lines -> regex `\b\d{2}[-/]\d{2}[-/]\d{4}\b` -> strip separators, parse `%d%m%Y` -> latest date vs today.
   - `face_similarity` — `DeepFace.verify(model_name="ArcFace", enforce_detection=False)` on `image1` vs `image2`.
   - `region_selection` — validate dimensions and bounds -> OpenCV crop -> save UUID file to `cropped_images/` -> RapidOCR on the crop.
6. **Response.** JSON dict tagged with `type`; errors returned as `{"error": "..."}` rather than raised.
7. **Display.** HTML UI renders JSON with two-space indentation; ROI text is auto-copied to the clipboard.

**Data flow:** Image bytes travel as multipart/form-data; decoded to NumPy arrays for OpenCV/RapidOCR/DeepFace; OCR yields text lines plus bounding boxes and confidences; keyword and expiry results derive from concatenated `all_text` (capped `[:500]`); face results carry `match` (bool), `confidence` (float), `distance` (float). Uploads and crops persist on the local filesystem only; there is no database.

**Integrations and dependencies:** RapidOCR (ONNX Runtime), DeepFace/ArcFace, OpenCV, RapidFuzz, NumPy, Pillow (Streamlit UI), Jinja2, Uvicorn, Streamlit, PyQt5. No external service calls exist in production (the Azure Form Recognizer path was removed).

```
Client (HTML/Jinja2 | Streamlit | PyQt5) -> multipart POST /process -> FastAPI dispatch on `type`
  -> [RapidOCR | RapidFuzz + keyword_match | re.find_expiry | cv2.check_resolution | DeepFace.verify(ArcFace) | cv2 crop -> RapidOCR]
  -> uploads/ & cropped_images/ (UUID files) -> JSON -> UI result-box / clipboard
```

**Architecture Summary:** A single-process, synchronous, four-route FastAPI service wraps five stateless validation functions behind one polymorphic `/process` endpoint keyed by a `type` form field. All inference runs in-process with no external service calls (ONNX Runtime OCR, ArcFace embeddings); CPU-only execution is inferred from the notebook's "Neither CUDA nor MPS are available" log. Business rules are pure Python with explicit thresholds. Storage is the local filesystem with UUID naming. Helpers are duplicated across the three front-ends "with consistent logic" (the document's wording) so rules behave the same regardless of interface. The design favours simplicity and zero external dependencies over scalability: no queue, no persistence layer, no authentication (not mentioned anywhere in the document — inferred), no async offloading of inference.

## 4. Technologies and Tools Used

| Category | Technologies (as stated in the document) |
|---|---|
| Programming languages | Python (version not specified in project; the document notes `python3.14.exe` is present on the documenting machine), JavaScript and HTML (canvas UI) |
| Frameworks | FastAPI (not pinned), Streamlit, PyQt5, Jinja2 (via FastAPI); no version is declared for any entry in the document's Technology Stack table |
| Libraries | OpenCV (`cv2`), RapidFuzz, NumPy, Pillow (Streamlit UI), `re`, `uuid`, `pathlib`; experimentation only: EasyOCR, InsightFace, `sentence_transformers`, Matplotlib |
| AI/ML models | ArcFace (via `DeepFace.verify`); trialled and abandoned: InsightFace `buffalo_l`, `all-mpnet-base-v2`, Azure Form Recognizer `prebuilt-read` |
| OCR tools | RapidOCR (ONNX Runtime) in production; EasyOCR and Azure Form Recognizer trialled and dropped |
| Databases | None — "There is no database"; local filesystem only |
| Cloud platforms | None in production; Azure Form Recognizer in `KYC 19-03-2026/main-old.py` (hardcoded endpoint/key, lines 33-34), removed |
| APIs | Own REST API: `GET /`, `POST /upload`, `POST /process`, `GET /image/{filename}`; Swagger UI at `localhost:8000/docs` (the document's "Swagger UI Version" heading is left empty) |
| DevOps tools | Not stated in documentation |
| Version control | Not stated in documentation (versioning evidenced only by dated folders and a `main - Copy.py` backup) |
| Deployment tools | Uvicorn (`uvicorn main:app --reload`), Streamlit CLI (`streamlit run streamlit.py`); no container or server deployment stated |
| Document processing tools | RapidOCR, OpenCV (decode, crop, dimension read), regex date parser, RapidFuzz text correction |
| Automation tools | Not stated in documentation (client-side auto-submit and clipboard copy for ROI only) |

## 5. Technical Skills Demonstrated

- **Software Engineering:** FastAPI dispatch pattern on a `type` field; `uuid.uuid4()` + suffix filenames; error-to-JSON handling; refactoring across versions (route prefix removal, inlining helpers, dropping the `utils` and Azure Form Recognizer imports); edge-case documentation with line references.
- **AI Engineering:** Local on-device inference (RapidOCR on ONNX Runtime, DeepFace ArcFace) chosen after comparative experiments; `enforce_detection=False` set in all implementations (inferred purpose: avoid hard failures when the detector misses a small document portrait); ML output combined with deterministic rules (fuzzy correction then keyword scoring).
- **Machine Learning:** ArcFace embedding verification (distance/confidence); evaluated InsightFace `buffalo_l` cosine similarity and `all-mpnet-base-v2` semantic matching; the semantic matcher was abandoned in favour of RapidFuzz (rationale inferred: short identity fields are lexical).
- **NLP:** RapidFuzz `partial_ratio` post-correction (threshold 70); case-insensitive substring matching with 65 % pass; regex date extraction and `%d%m%Y` parsing; response truncation policy.
- **Computer Vision:** OpenCV decode, ROI crop with bounds validation, orientation-agnostic resolution gating; notebook experiments in morphological close + contour card localisation and OpenCV face detection; canvas-to-image coordinate scaling.
- **Data Engineering:** `uploads/` / `cropped_images/` layout with UUID naming; multipart ingestion; JPEG preprocessing (resize 640x480, RGB, EXIF inspection) in the Azure experiment; earlier `/api/cleanup` endpoint.
- **Cloud:** Prototyped Azure Form Recognizer (`prebuilt-read`, parallel asyncio tasks, 50 % IoU merge) and consciously removed it; no cloud in production.
- **MLOps:** Not demonstrated beyond an experiment-to-production traceability table that flags one parameter drift (expiry regex).
- **API Development:** Four-endpoint REST design with `Form` and `UploadFile` parameters, per-mode response schemas, `FileResponse` serving, auto-generated Swagger.
- **Prompt Engineering:** Not demonstrated in this project (no LLM used).
- **System Design:** Interchangeable client layer (web, Streamlit, desktop) over one processing layer and a filesystem storage layer; deliberate helper duplication for framework independence; explicit constants table.

## 6. Detailed Technical Contributions

**Features implemented**
- `GET /` renders `templates/index.html` via `Jinja2Templates(directory='templates')`; responds `200 OK` with `Content-Type: text/html` carrying the full UI (canvas region selection, selfie upload, keyword input). Error responses are "not handled explicitly; FastAPI returns 500 on unhandled exceptions."
- `POST /upload` accepts `file: UploadFile` (image; "any extension accepted; UUID prefix applied"), saves `uploads/<uuid>.<ext>`, returns `{"success": true, "filename": "<uuid>.<ext>"}`. Edge case: a filename without suffix yields a UUID with no extension (`Path('').suffix` is empty).
- `POST /process` fields: `type` (required), `image1` (required), `image2` (required only for `face_similarity`, "ignored for other types"), `keywords` (default ""), `region_x/y/w/h` (default 0). Invalid `type` -> `{"error": "Invalid type"}`; any exception -> `{"error": "<message>"}`.
- All three UIs "are confirmed by import statements and dependency usage in the respective files" (document, section 5).
- `GET /image/{filename}` returns `FileResponse` for `uploads/{filename}` with no existence check (documented `FileNotFoundError`; `KYC 19-03-2026/main.py` checked `filepath.exists()`).
- HTML UI: mode `<select>`; `image/*` input triggering upload; canvas drag-to-draw red rectangle; on `mouseup`, if width and height > 10 display px (`index.html` line 234), coordinates scale to the original image and submit automatically; JSON shown via `JSON.stringify(..., 2)`; ROI text copied to clipboard.
- Streamlit UI (`streamlit.py`) covering all five modes — described as "an alternative self-contained browser UI" with a "browser-based multi-column layout", image display and result panels; PyQt5 desktop UI with the same > 10 px guard (`pyqt_app.py` line 257). The HTML/Jinja2 interface is named "the primary web interface" and all three UIs are "present in the production folder".
- No file-type or MIME validation is documented on `POST /upload` — "Any extension accepted; UUID prefix applied" — and the original filename is discarded (`uuid.uuid4()` plus the original suffix is all that is stored), so uploads carry no client-supplied name into `uploads/` (analyst observation from the stated behaviour).

**Models used**
- ArcFace via `DeepFace.verify` (`main.py` line 83), `enforce_detection=False` "in all implementations"; output `match`, `confidence`, `distance`.
- RapidOCR (ONNX Runtime) for all text extraction, returning text, bounding boxes and confidences (notebook cell 18 stored the "full bounding box + confidence array"). Production surfaces only the text (`all_text`, `text`); no response schema exposes the per-line confidences or boxes (analyst observation). Cell 17's stored output on a UAE driving-licence front shows the raw token quality the downstream rules must absorb: "united arab emirates", "driving license", "license no", "2332173", "namoomin hasan uppinangady ili", "natleality india", "10061987", "14042016".
- Trialled in `testing.ipynb` with recorded outcomes: cell 0 loaded `single_doc check.png` via PIL and converted to grayscale (output mode=L, size 631x387; production uses `cv2.imread`/`cv2.imdecode` instead); cell 7 inspected EXIF metadata of a test JPEG (dpi=96, jfif_version=(1,1)) and cell 8 displayed a PIL image resized to 640x480, both feeding the Azure Form Recognizer preprocessing later dropped; cells 14-15 ran EasyOCR (`en`) on an insurance-claim image, with the text result truncated in the stored output; cell 19's `doc_expiry` prototype (optional-separator regex) returned `['Valid']` for the test document; cell 20's RapidFuzz prototype was carried into production as the `fuzzy_match` / `fuzzy()` helper with threshold 70. Cells 3, 9-10, 20 and 21 have no stored output — note that this includes cell 20, the RapidFuzz prototype that *was* carried into production, so neither side of the RapidFuzz-vs-embeddings comparison recorded a measured result in the notebook. Cells 11-13 and 22-27 are empty.

**Pipelines built**
- Keyword: OCR -> RapidFuzz `partial_ratio >= 70` (`main.py` line 24, `fuzzy_match` / `fuzzy()` helper) -> comma split -> substring match -> `found`, `not_found`, `score` (int), `percentage` (float), `matched = percentage >= 65` (line 37) -> `all_text[:500]` (lines 195 and 203; "no ellipsis is appended").
- Expiry: OCR -> regex (line 44) -> strip separators, `%d%m%Y` (line 48) -> latest date vs today -> `status` in `Valid | Expired | No expiry date found`.
- Resolution: dimension read -> `min(w,h) < 480 OR max(w,h) < 640` (line 65) -> `Good Quality (WxH)` or `Low Resolution (WxH)`.
- ROI (`main.py` lines 142-175): non-zero `region_w`/`region_h` -> `region_x + region_w <= width` and `region_y + region_h <= height` -> `cv2` crop -> `cropped_images/<uuid>` -> OCR -> `{text, words, region:{x,y,width,height}, cropped_image_path}`.
- Face: `image1` vs `image2` -> `DeepFace.verify(model_name="ArcFace")` -> `{match, confidence, distance}`.

**APIs integrated**
- None external in production. Prototype only: Azure Form Recognizer `prebuilt-read` in `main-old.py`, run in parallel asyncio tasks with RapidOCR and merged by 50 % IoU overlap.

**Data extraction methods**
- Full-image OCR; ROI OCR on OpenCV crops; regex date capture; fuzzy-corrected keyword lookup.

**Validation logic**
- Keyword `>= 65 %` (`ceil(0.65 * N)` keywords, strict `>=`); expiry latest DD[-/]MM[-/]YYYY vs today; resolution 640 x 480 via `min`/`max`; region zero-dimension guard (`Invalid region dimensions`), out-of-bounds guard (`Region out of bounds. Image size: WxH, Region: x,y WxH`), client-side > 10 px; face missing `image2` -> `{"error": "Second image required"}`.

**Automation workflows**
- Auto-upload on file selection; auto-submit on ROI mouseup; auto-copy of ROI text. No scheduled or batch automation.

**Optimization techniques**
- ONNX Runtime OCR (RapidOCR) adopted instead of EasyOCR; the document records only that EasyOCR fell back to CPU (cell 14: "Neither CUDA nor MPS are available - defaulting to CPU") and that RapidOCR was "carried into production" — the CPU-efficiency rationale is inferred, not stated.
- Dropped the Azure dual-engine design for a single local engine, which removes the network dependency and the hardcoded credential in `main-old.py` (benefit inferred; not measured).
- `all_text` capped at 500 characters; fuzzy pre-correction to reduce OCR-noise false negatives.

**Performance improvements**
- Not quantified; the document states no benchmark results, logs or metrics exist in the project folder.

## 7. Challenges and Solutions

1. **OCR noise breaking exact keyword matching (technical).** Extracted text such as "natleality india" (cell 17) fails an exact match. Solution: RapidFuzz `partial_ratio >= 70` pre-correction plus a 65 % pass threshold. Alternative considered: `all-mpnet-base-v2` semantic matching (cell 21), "abandoned in favour of RapidFuzz" (the document gives no reason; that the lexical approach is lighter is inferred). Note that neither cell 20 (RapidFuzz) nor cell 21 (sentence_transformers) stored any output, so the choice is not backed by a recorded comparison.

2. **Choosing an OCR engine (technical).** EasyOCR defaulted to CPU with pin_memory warnings (cells 14-15); the Azure path carried a hardcoded endpoint and key (`main-old.py` lines 33-34). Solution: RapidOCR on ONNX Runtime, run on a UAE driving licence and "compared against EasyOCR" (cells 16-18) and carried into production. The document does not state why RapidOCR won; the CPU/no-network/no-credentials rationale is inferred. Alternatives: EasyOCR (cells 14-15), Azure `prebuilt-read` + RapidOCR merged at 50 % IoU (`main-old.py`).

3. **Face verification on small document portraits (technical).** Detectors can miss the face on a document photo (inferred rationale). Solution: `DeepFace.verify` with ArcFace and `enforce_detection=False` so a distance is returned instead of an exception. Alternatives: InsightFace `buffalo_l` cosine similarity (cells 9-10), `DeepFace.extract_faces` with opencv backend (cell 3), raw OpenCV detection (cell 2).

4. **Field-level validation without a layout model (technical/business).** Solution: operator-drawn ROI, scaled to original coordinates, cropped and OCR'd. Alternative attempted: automatic card localisation via morphological close + contours (cell 1 returned the full extent, "Card location: 0 0 627 387", so it was dropped). Analyst suggestion: anchor-based field detection using `document_anchors.py` or a small detector to remove the manual step.

5. **Expiry regex drift (technical, documented limitation).** Notebook and earlier `utils.py` used optional separators (`[-/]?`); production requires them — the document calls this "a parameter drift between experiment and production". Separator-less OCR output such as `10061987` (cell 17) would therefore not match in production and fall through to "No expiry date found" (inferred consequence). Analyst suggestion: restore optional separators with day/month range validation.

6. **Unhandled missing file on `GET /image/{filename}` (technical, documented bug).** No existence check before `FileResponse`; the earlier `KYC 19-03-2026/main.py` checked `filepath.exists()`. Not fixed in production. Analyst suggestion: reinstate the check and return 404.

7. **Hardcoded Azure credentials in a superseded file (business/security).** `main-old.py` lines 33-34. Production removed the Azure path. Analyst suggestion: purge from history and use environment variables.

8. **Consistency across three UIs (business).** Solution: helper logic (`keyword_match`, `find_expiry`, `check_resolution`, `compare_faces`) duplicated in each UI "with consistent logic" so business rules can be tested independently of any single interface (stated). Trade-off (analyst view): duplication risks drift, as already seen in the expiry regex. Analyst suggestion: a shared validators package, which the earlier `utils.py` had begun to provide.

9. **Ambiguous resolution rule (technical, documented).** The narrative states the image "is considered good quality when min dimension (height) >= 640 px AND min dimension (width) >= 480 px", while the Implementation Constants table records the actual check as `min(w,h) < 480 OR max(w,h) < 640` (`main.py` line 65) — the code is orientation-agnostic (a 640x480 image passes in either orientation) whereas the prose reads as a portrait-only rule. Not solved: the document itself flags the discrepancy in the constants table. Analyst suggestion: state the orientation-agnostic rule explicitly in the UI and the docs so operators know in advance what will be rejected.

10. **No metrics (business).** Analyst suggestion: a labelled GCC document/selfie set with field-level OCR accuracy, expiry recall, and face-verification FAR/FRR at the ArcFace threshold.

## 8. Impact Analysis

- **Business impact:** An internal, programmatic path to validate KYC documents across five checks plus a REST endpoint for CRM and policy platforms. The document states in full: "Quantified performance figures (processing time, accuracy rates, throughput): Not specified in the project. No benchmark results, log files, or metrics are present in the project folder."
- **Documented Strategic Benefits (source section 4), verbatim in substance:** (a) *GCC document coverage* — `document_anchors.py` covers UAE, Qatar/Doha, Oman and Kuwait driving licences, residency permits and vehicle registrations plus passports from India, Pakistan, Oman, Philippines, Egypt and Bangladesh, "reflecting a Gulf insurance/onboarding context"; (b) *framework-agnostic processing layer* — the helpers are "duplicated across all three UIs with consistent logic, meaning business rules can be tested independently of any single interface"; (c) *extensible REST API* — "the FastAPI endpoint structure allows downstream systems (CRM, policy platforms) to call the API without going through the browser UI".
- **Productivity improvements (qualitative):** Automated extraction removes manual data entry; configurable keyword checks reduce manual review; ROI extraction with clipboard copy shortens field checks.
- **Accuracy improvements:** Not measured. Fuzzy pre-correction and the 65 % threshold aim to reduce false negatives; the resolution check, in the document's own words, "can be used to reject low-quality images before processing, reducing downstream OCR errors".
- **Cost savings:** Not stated. Structurally, production has no cloud or API cost after the Azure dependency was dropped.
- **Time savings:** Not measured. Expiry check yields an immediate valid/expired signal "without human reading".
- **User benefits:** Three UI options "enabling deployment in web-server and local-desktop scenarios without code changes" (document wording); the result box renders the raw JSON response via `JSON.stringify` with 2-space indentation, so an operator sees exactly what the API returned (the "transparency" framing is analyst view); ROI text is auto-copied to the clipboard, "supporting field-level validation workflows"; internal users are the stated target.

## 9. Interview Discussion Points

(The design rationales below are analyst reasoning prepared for interview use; the document records what was built, not why, except where quoted.)

- Why one polymorphic `/process` endpoint rather than five routes? A single multipart contract keeps the UI simple; the cost is weaker per-mode OpenAPI typing and a shared `{"error": ...}` shape.
- Why keep `POST /upload` separate from `POST /process`? `/upload` exists to store and preview the image (UUID filename returned, served back via `GET /image/{filename}`), while `/process` reads bytes straight from the multipart body so "the /upload endpoint does NOT need to be called first". Discuss the consequence: uploads persist with no cleanup route in production.
- Small documented edge cases worth volunteering: a file with no suffix is stored as a bare UUID; `all_text` is cut at 500 characters with no ellipsis; `GET /` has no explicit error handling (FastAPI returns 500 on unhandled exceptions).
- Why RapidOCR over EasyOCR and Azure Form Recognizer? CPU-only environment (cell 14 log), removal of network dependency and the hardcoded key, UAE driving-licence validation in cells 16-18.
- Why RapidFuzz `partial_ratio` at 70 rather than embeddings? Identity fields are short and lexical (inferred); neither the RapidFuzz cell (20) nor the `all-mpnet-base-v2` cell (21) stored any output, so no measured benefit was recorded either way. Acknowledge that 70 and 65 are heuristics with no documented tuning.
- What does `enforce_detection=False` do and what risk does it carry? It avoids exceptions when no face is found but can yield a distance on a non-face crop, so `confidence`/`distance` must be interpreted carefully.
- Walk through the expiry logic and its documented drift, including the `10061987` failure mode.
- Explain the resolution rule precisely (`min(w,h) < 480 OR max(w,h) < 640`) and why orientation-agnostic gating is preferable.
- Describe canvas-to-image coordinate scaling and the three ROI guards (> 10 px, zero-dimension, out-of-bounds).
- What would you fix first? The `GET /image/{filename}` existence check, then extracting shared helpers into a module.
- Why no database? Internal PoC scope; discuss the audit trail and retention policy a regulated KYC process needs.
- How did you evaluate abandoned experiments? Be candid that several cells have no stored outputs and the contour approach returned the full frame.
- Security gaps: no auth, uploads persist after `/cleanup` was dropped, the Azure key in a superseded file.
- How would you measure success? FAR/FRR for face match, field-level OCR accuracy on GCC documents, expiry-detection recall.
- Why three front-ends instead of one? The document's stated benefit is deployment "in web-server and local-desktop scenarios without code changes"; the cost is triplicated helper logic with no shared module after `utils.py` was inlined into `main.py`, which is exactly how the expiry-regex drift became possible.
- How the 65 % rule behaves on short keyword lists (inferred arithmetic, worth volunteering): with 3 keywords, 2 found = 66.7 % passes; with 2 keywords, 1 found = 50 % fails, so both are effectively required. A percentage threshold is coarse on the small keyword sets an operator is likely to type — and the document records the constant (`main.py` line 37) without a rationale.
- Be precise about where the 500-character cap applies: `[:500]` is documented on the `all_text` field of the `keyword_match` and `expiration` responses (`main.py` lines 195, 203, "no ellipsis is appended") — i.e. on what is returned to the client, not on what is matched (inferred). Do not let it be read as a matching limit.
- The version story as evidence of engineering judgement: `KYC 19-03-2026` (`/api/` prefix on all five routes, Azure `prebuilt-read` + RapidOCR in parallel asyncio tasks merged at 50 % IoU, `/api/cleanup`, `utils-old.py` whose `similarity_search` returned a formatted string `'Confidence : X and Time taken : Y'`) → `Final KYC 20-03-2026` (flat routes, one local engine, helpers inlined, structured dicts). Say what each removal bought (no network, no credential, fewer moving parts) and what it cost (no cleanup route, so uploads accumulate).
- RapidOCR already returns bounding boxes and confidences (cell 18) but production returns only text — an obvious next step is confidence-based flagging of unreliable fields, and box coordinates would enable the anchor-based field detection `document_anchors.py` was presumably built for (analyst suggestion).
- What the documentation itself does not contain, said plainly: no pinned library versions (every stack row is "Not specified"), no tests, no CI/CD, no authentication or retention policy, no PyQt5 launch command or interface guide, an empty "Swagger UI Version" heading, and UI sections that are screenshots only. Owning those gaps is more credible than papering over them.

## 10. Architecture Explanation Points

Reproduce the document's own Figure 1 ("End-to-End Architecture"), a four-layer left-to-right chain: **Client / UI — Web / Streamlit / PyQt5** ➔ **FastAPI Server — validation router & dispatch** ➔ **AI / Processing Layer — RapidOCR / DeepFace / OpenCV** ➔ **Storage Layer — local uploads / crops & JSON**. Expand each: the client layer is three interchangeable front-ends sharing one multipart contract (HTML/Jinja2 canvas UI is "the primary web interface", Streamlit is a multi-column browser UI, PyQt5 is the desktop app); the server declares four routes with `/process` dispatching on `type`; the processing layer holds the five functions `extract_text`, `keyword_match`, `find_expiry`, `check_resolution`, `compare_faces`; the storage layer is `uploads/` and `cropped_images/` with UUID names, with "no external database or cloud storage configured in production code".

Trace one request: browser uploads a document, previews via `/image/{filename}`, operator picks "Face Similarity" and adds a selfie; `/process` decodes both images with OpenCV, calls `DeepFace.verify` with ArcFace and `enforce_detection=False`, and returns `{match, confidence, distance}`. Then trace the ROI path to show crop-then-OCR with bounds checks.

Key decisions: local in-process inference with no external service calls (CPU-only inferred from the notebook log; absence of cloud cost and credentials in production follows from removing the Azure path); deterministic rules with explicit thresholds (70 fuzzy, 65 % keyword, 640 x 480); fuzzy correction before matching; duplicated helpers for framework independence; a separate `/upload` for preview versus direct-bytes `/process` for validation.

Trade-offs: synchronous inference in the request thread limits concurrency; no persistence means no audit trail; duplication risks drift (already seen in the expiry regex); `enforce_detection=False` trades hard failures for possibly meaningless distances; errors are returned as `{"error": "..."}` with HTTP 200 semantics rather than raised, which is simple for the UI but gives callers no status-code signal (inferred from the documented response shapes); the `[:500]` cap is on the returned `all_text`, not on the text that is matched (inferred); `POST /upload` accepts any extension with no type validation; and only a `--reload` development server is documented, so nothing about workers, TLS or process management has been decided.

Next improvements: reinstate the file-existence check on `GET /image/{filename}` and return 404; extract a shared validation package (the direction `utils.py` had started before helpers were inlined); add authentication, upload retention and a replacement for the dropped `/cleanup` route; move inference to a worker queue; pin dependency versions (none is declared anywhere in the stack table); surface RapidOCR's per-line confidences and boxes rather than text alone; document a PyQt5 launch path and interface guide; build a labelled GCC document/selfie set to measure accuracy and tune the 70 / 65 / 640x480 constants; revisit anchor-based automatic field localisation using `document_anchors.py` to replace manual ROI selection.
