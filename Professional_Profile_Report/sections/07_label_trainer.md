# Project 7: TextMatch Label Trainer (Streamlit UI)

## 1. Project Overview

- **Project name:** TextMatch Label Trainer (Streamlit UI) — document no. AT/label-trainer/V1, dated 01-Sep-2026, technology tag "TextMatch / Streamlit".
- **Business domain:** Insurance — document classification for Qatar Insurance Group across four regional jurisdictions (UAE, DOHA, OMAN, KUWAIT).
- **Problem statement (inferred from the stated business benefits; the document does not describe the prior state directly):** Downstream detection and OCR extraction services classify documents by matching per-label keyword sets. Maintaining those dictionaries required engineering intervention and manual document inspection, with no empirical check against real samples before go-live; weak definitions risked production false positives, and upcoming Oman and Kuwait expansions needed a fast, safe onboarding path for new document categories.
- **Project objective:** Build a browser-based administrative tool letting non-technical operators upload sample documents, run local or cloud OCR, receive n-gram keyword suggestions, fuzzy-score proposed keywords against the samples, and — only if the average clears a configurable threshold (default 75%) — persist the label/keyword mapping to a region-scoped JSON store (`labels_DB/<region>_labels.json`) that production FastAPI pipelines read immediately without a restart.
- **Expected business outcome:** The document lists six benefits explicitly — four operational (No-Code Rule Authoring; Instant Production Deployment, i.e. changes are active in the detection and extraction APIs without downtime or deployments; Automated Discrimination Discovery via n-gram suggestions replacing manual document inspection; Empirical Acceptance Testing, guaranteeing new keyword sets meet a verified match threshold against real samples before entering production) and two strategic (Rapid Jurisdiction Expansion for the upcoming Oman and Kuwait onboarding; Controlled Classification Quality through strict threshold governance that prevents weak keyword definitions from causing production false positives). The document states quantified business metrics (operational hours saved, ROI figures, throughput benchmarks) are not recorded in project files.
- **Target users (stated in the document):** business analysts, operations teams, and document specialists who maintain classification dictionaries without writing code; system administrators and developers also use the UI to inspect OCR extraction performance and calibrate fuzzy thresholds — so the tool doubles as an OCR debugging console.

## 2. Role and Responsibilities

**Core responsibilities (inferred from the system's design and the PoC-to-production traceability in the document):**

- Designed the end-to-end architecture: Streamlit reactive frontend -> in-memory ingestion -> PyMuPDF rasterization -> dual OCR dispatch -> RapidFuzz scoring -> regional JSON persistence consumed by FastAPI detection/extraction services.
- Implemented the Streamlit application (`label_trainer_streamlit.py`): `st.session_state` management, the non-dismissible `@st.dialog('Select Region')` modal (`region_popup()`), sidebar expanders, file uploader, previews, suggestion pills, threshold slider, and the `extract_and_score()` submit flow.
- Implemented the utility module (`utils.py`) containing `suggest_keywords()` (n-gram harvesting and fuzzy clustering) and `keyword_score()` (RapidFuzz keyword matching with an 80 ratio cutoff).
- Integrated two OCR engines behind one interface: local RapidOCR (`rapidocr_onnxruntime`) via `rapid_ocr()` and Azure Document Intelligence `prebuilt-read` via `azure_ocr()`.
- Designed the hot-reload persistence contract: accepted labels serialize to `labels_DB/<region>_labels.json`, which the downstream FastAPI detection and extraction pipelines pick up without deployments.
- Ran the experimentation programme recorded in `test.ipynb` and carried each of the four trials into production with explicit file/line traceability: automated n-gram extraction (`predict_keyword`, cells 56–57) -> `utils.py:suggest_keywords()`, invoked by the "Get Keyword Suggestions" button at `label_trainer_streamlit.py:108`; keyword match scoring against labels (cells 50, 58–64) -> `utils.py:keyword_score()`, used by `extract_and_score()` at `label_trainer_streamlit.py:78`; local RapidOCR vs Azure OCR evaluation (cells 25, 38, 51) -> the sidebar "OCR Engines" radio toggle; acceptance-threshold calibration (cells 63–66) -> the slider at `label_trainer_streamlit.py:127` with default 75%.
- Configured deployment: Streamlit CLI launch commands and the Nginx 1.31.4 reverse-proxy gateway with WebSocket upgrade headers.
- Chose a CLI-run Streamlit process rather than an ASGI application server, so the app exposes no HTTP REST endpoints and communicates entirely over Streamlit's internal WebSocket protocol, reading and writing JSON files in `labels_DB/` directly.

**Supporting tasks (inferred):**

- Authored `README.md` launch instructions and `requirements.txt`.
- Technical documentation (verified-constants table with file/line citations, edge-case findings) exists for the project; its authorship is not stated.
- Defined input validation (label present, at least one file) and user feedback (toast, Accepted/Rejected banners, JSON echo).
- Handled first-run initialisation of an empty regional JSON file, label overwrite semantics, and logging of accepted writes.

**Estimated ownership level: AI Engineer (inferred).**
Rationale: the document shows a single ownership arc from notebook experiments (`test.ipynb` cells 25–66) through production functions (`utils.py`, `label_trainer_streamlit.py`) to deployment configuration (`nginx.conf`), spanning UI, OCR integration, fuzzy-NLP scoring, persistence design, and TextMatch FastAPI integration. Scope is moderate (one Streamlit script, a utility module, proxy config) with no evidence of team leadership, so "AI Engineer" fits better than Senior/Architect for this component alone, although the hot-reload contract with production pipelines shows system-level thinking (inferred).

## 3. End-to-End Workflow

**Processing stages:**

1. **Session bootstrap and region binding.** On load, if `st.session_state.region` is `None`, `region_popup()` renders a modal with a selectbox (UAE, DOHA, OMAN, KUWAIT) and a Confirm button; on confirm the region is stored in session state, the script re-runs, and a fade-out toast confirms. If `labels_DB/<region>_labels.json` is missing, an empty dictionary is written to disk.
2. **Sample ingestion and preview.** The uploader accepts multiple JPG/JPEG/PNG/PDF files (including multi-page PDFs) into memory buffers; images open with `PIL.Image`, PDFs have page 0 rendered to an RGB image by PyMuPDF (`fitz`) at 150 DPI (matrix 150/72) for visual preview and JPEG-encoded for OCR, and are shown with filename and page-count captions.
3. **OCR engine selection.** A sidebar radio toggles `CPU_OCR` (local RapidOCR, captioned faster/lower accuracy) or `AZURE` (Azure Document Intelligence prebuilt-read, captioned higher accuracy/slower).
4. **Keyword suggestion (optional).** "Get Keyword Suggestions" runs OCR over all uploads and calls `suggest_keywords()`: 3-, 2-, and 1-gram extraction, tokens shorter than 6 characters dropped, tokens required in >=2 documents when multiple files are uploaded, similar terms clustered with RapidFuzz `partial_ratio` at 80. Suggestions render as clickable pills; "Clear Suggestions" resets session state. With a single upload, clustering is bypassed and raw OCR lines (`texts[0]`) are returned.
5. **Label and keyword entry.** Operators type a label (e.g. `Driving License Front`) and comma-separated keywords (whitespace-stripped), or pick an existing label from the sidebar "Saved Labels" radio group to pre-fill both fields.
6. **Scoring and verdict.** Submit calls `extract_and_score()`: validates label and uploads (else `st.error()` halts), then per file rasterizes PDF page 0 at 150 DPI, JPEG-encodes at quality 95, invokes `azure_ocr()` or `rapid_ocr()`, and computes `keyword_score(file_ocr_lines, keywords)` (match when RapidFuzz ratio >= 80; score = matched / total * 100); scores are averaged.
7. **Persistence.** If `avg_score >= acceptance_threshold` (slider 0–100, step 1, default 75), a green "Accepted" banner appears, the label-keyword set is written to the regional JSON (overwriting a same-name label), the action is logged, and the saved JSON is echoed; otherwise a red "Rejected" message appears and nothing is written.
8. **Downstream activation.** The FastAPI detection and extraction pipelines read `labels_DB/<region>_labels.json` and use the new mapping immediately — no restart or deployment.

**Data flow:** uploaded bytes (in-memory buffers) -> RGB image (PIL for images; PyMuPDF page-0 render for PDFs) -> JPEG bytes at quality 95 (documented for PDF pages; the non-PDF encoding path is not detailed) -> OCR text lines per file -> n-gram suggestions and per-file percentage scores -> average -> `{"label": ..., "keywords": [...]}` written to `labels_DB/<region>_labels.json`.

**Named architecture components (the document's own component table, seven in total):** Streamlit Reactive Frontend (widgets, image previews, feedback messages, `st.session_state` for region and suggestions); Region Selection Modal (`@st.dialog('Select Region')`, non-dismissible, enforces regional context before any controls render); Document Preview & Ingestion (buffers uploads, `PIL.Image` for images, PyMuPDF page-0 rasterization at 150 DPI for PDFs); OCR Text Extraction Engine (dispatches image bytes to local RapidOCR or Azure Document Intelligence per the sidebar radio); Suggestion Clustering Module (`suggest_keywords()` harvesting high-frequency n-grams across uploaded documents); Fuzzy Scoring & Evaluation (`keyword_score()` across samples plus the average-vs-threshold check); Regional JSON File Store (serializes accepted labels and keyword lists to `labels_DB/<region>_labels.json` on disk).

**Runtime and interface model:** the application is executed by Streamlit's CLI runner rather than an ASGI application server, exposes no standard HTTP REST endpoints, and communicates over Streamlit's internal WebSocket protocol; all persistence is direct file I/O in `labels_DB/`. Every user interaction re-executes the script top to bottom while `st.session_state` preserves the selected region, suggestions, and form inputs.

**Integrations and dependencies:** Streamlit (session state, `@st.dialog`, `st.file_uploader`); Pillow and OpenCV; PyMuPDF/fitz; RapidOCR `rapidocr_onnxruntime`; Azure AI Document Intelligence `prebuilt-read`; RapidFuzz (`ratio` for matching, `partial_ratio` for clustering); Nginx 1.31.4 (version from the directory name `nginx-1.31.4/nginx-1.31.4/conf/nginx.conf`) with WebSocket upgrade; the sibling FastAPI detection/extraction services as consumers of the JSON store. All Python packages are unpinned in `requirements.txt`.

**Flow line:**
`Operator browser -> Nginx :9000 (WS upgrade) -> Streamlit :8000 (session_state, region modal) -> PyMuPDF page-0 raster @150 DPI / PIL -> JPEG q95 -> RapidOCR | Azure DI prebuilt-read -> suggest_keywords() n-grams / keyword_score() (ratio>=80) -> avg >= threshold (default 75) -> labels_DB/<region>_labels.json -> FastAPI detection & extraction pipelines (hot-reload)`

**Architecture Summary:** The Label Trainer is a human-in-the-loop configuration tool built on Streamlit's rerun-per-interaction model: state that must survive reruns (region, suggestions, form inputs) lives in `st.session_state`; everything else is recomputed. The region modal is a hard gate scoping every read/write to one JSON file, giving jurisdiction isolation without a database. OCR is pluggable at runtime so operators trade speed for accuracy. The n-gram suggester and fuzzy scorer live in `utils.py`, promoted from `test.ipynb`; whether the FastAPI services import that module is not stated — only that they read the shared `labels_DB` store, which is why persistence is file-based and changes are live instantly. The app runs under Streamlit's CLI behind Nginx, which must forward WebSocket upgrades.

## 4. Technologies and Tools Used

| Category | Technologies (as stated in the document) |
|---|---|
| Programming languages | Python (inferred from Streamlit, `utils.py`, `requirements.txt`, `.ipynb`) |
| Frameworks | Streamlit (unpinned in requirements.txt) |
| Libraries | Pillow (PIL), OpenCV, PyMuPDF (fitz), RapidOCR (rapidocr_onnxruntime), RapidFuzz (rapidfuzz), Azure AI Document Intelligence SDK — all unpinned in requirements.txt |
| AI/ML models | Azure Document Intelligence `prebuilt-read`; RapidOCR local ONNX models (runtime inferred from the `rapidocr_onnxruntime` package name; specific models not stated) |
| OCR tools | RapidOCR (CPU_OCR, local); Azure AI Document Intelligence (AZURE, cloud) |
| Databases | None — regional JSON file store `labels_DB/<region>_labels.json` |
| Cloud platforms | Microsoft Azure (Document Intelligence service) |
| APIs | Azure Document Intelligence prebuilt-read API (consumed); downstream FastAPI detection and extraction pipelines consume the JSON store. The app exposes no REST endpoints (Streamlit WebSocket protocol only) |
| DevOps tools | Nginx reverse proxy (1.31.4 per directory `nginx-1.31.4/nginx-1.31.4/conf/nginx.conf`; port 9000 -> 127.0.0.1:8000 with `proxy_set_header Upgrade $http_upgrade`) |
| Version control | Not stated in documentation |
| Deployment tools | Streamlit CLI (`streamlit run label_trainer_streamlit.py --server.port 8082 --server.address 0.0.0.0`; port 8000 on 127.0.0.1 behind Nginx); Nginx gateway |
| Document processing tools | PyMuPDF (PDF page-0 rasterization at 150 DPI, RGB), Pillow, OpenCV, JPEG encoding at quality 95 |
| Automation tools | Not stated in documentation (Jupyter notebook `test.ipynb` used for experimentation/PoC only) |

**Not stated in documentation** (deliberately absent so nothing is over-claimed): CI/CD pipelines, containerisation, orchestration, test frameworks, monitoring/observability, message queues, and any authentication, authorization, TLS, secrets- or credential-management scheme (the document never describes how the Azure Document Intelligence key/endpoint is supplied, nor any access control in front of the Streamlit app or the Nginx gateway). Library versions are also unavailable — the document's own technology table records every dependency as "Unpinned in requirements.txt", and the only version string anywhere is the Nginx directory name.

## 5. Technical Skills Demonstrated

- **Software Engineering:** Structured a Streamlit app around rerun semantics with explicit session-state ownership; separated reusable logic into `utils.py`; input validation with early halts; first-run initialisation of the JSON store; label overwrite semantics; logging of persistence events.
- **AI Engineering:** Built a human-in-the-loop tool for domain experts to author and empirically validate classifier configuration; integrated cloud and local OCR behind one dispatch; designed the hot-reload contract so configuration ships without redeploying inference services.
- **Machine Learning:** Threshold calibration — tested 50%, 65%, 75% cutoffs for false-acceptance behaviour in `test.ipynb` (cells 63–66), shipping 75% as default with an operator slider; benchmarked two OCR engines for keyword-matching accuracy (cells 25, 38, 51); compared two scoring formulations (`ratio` vs `token_sort_ratio` under matched/total*100) in cells 50 and 58–64. No model training, fine-tuning, or supervised evaluation dataset is described — the "ML" work here is empirical threshold/scorer selection and off-the-shelf OCR benchmarking, and the resulting classifier is a keyword/fuzzy rule set rather than a learned model.
- **NLP:** Implemented n-gram (3/2/1) frequency extraction with length (>=6 chars) and document-frequency (>=2 docs) filters and RapidFuzz `partial_ratio` clustering (cutoff 80); keyword-match scoring at ratio >= 80 with a matched/total*100 formula (`ratio` and `token_sort_ratio` both evaluated in `test.ipynb`; the constants table names "ratio" for production).
- **Computer Vision:** Image ingestion with PIL/OpenCV, PDF-to-image rasterization via PyMuPDF at 150 DPI, JPEG encoding at quality 95 for OCR input; OCR text extraction with RapidOCR and Azure prebuilt-read.
- **Data Engineering:** In-memory buffering of multi-file uploads; region-partitioned JSON persistence; consistent record schema `{"label", "keywords"}` consumed by other services.
- **Cloud:** Consumption of Azure AI Document Intelligence `prebuilt-read` as the high-accuracy OCR path.
- **MLOps:** Traceable experiment-to-production lineage (notebook cells cited against production file/line numbers); configuration-as-data with instant production activation; documented edge-case verification.
- **API Development:** Not demonstrated in this project as a producer (no REST endpoints); the app consumes the Azure OCR API and feeds FastAPI pipelines via the shared file store.
- **Prompt Engineering:** Not demonstrated in this project.
- **System Design:** Region-scoped workspace isolation via a mandatory modal; runtime-switchable OCR backends; threshold governance gate before persistence; file-based hot-reload integration; reverse-proxy deployment with WebSocket support; documented port mismatch and its remediation.

## 6. Detailed Technical Contributions

**Features implemented:**
- Header and region indicator in the main panel: application title plus an active-region caption (e.g. "Region: UAE").
- Non-dismissible region modal (`@st.dialog('Select Region')`, `region_popup()`) with selectbox (UAE, DOHA, OMAN, KUWAIT), Confirm, toast, and auto-creation of an empty `labels_DB/<region>_labels.json`.
- Multi-file uploader (JPG, JPEG, PNG, PDF; drag-and-drop via `st.file_uploader`) with thumbnail/page-0 previews and filename/page-count captions.
- Sidebar "Saved Labels" expander: sorted radio group from the regional JSON; selection pre-fills label and keyword inputs.
- Sidebar "OCR Engines" expander: `CPU_OCR` / `AZURE` radio with dynamic captions — "Faster but lower accuracy" for `CPU_OCR`, "Higher accuracy but a bit slower" for `AZURE`.
- "Get Keyword Suggestions" / "Clear Suggestions" buttons; suggestions as clickable pills in session state.
- Label input, comma-separated keyword input (whitespace-stripped), threshold slider (0–100, step 1, default 75, tooltip).
- Submit -> `extract_and_score()` -> green "Accepted" / red "Rejected" banner; persisted JSON echoed on acceptance. The document's worked example of the echoed record is `{"label": "Driving License Front", "keywords": ["expiry date", "driving license", "license no", "issue date", "united arab emirates"]}` — i.e. multi-word phrases, lower-cased, and jurisdiction-specific terms are all valid keywords.

**Models used:** Azure Document Intelligence `prebuilt-read` (cloud OCR); RapidOCR via `rapidocr_onnxruntime` (local CPU OCR). No trained classifier is described — classification is keyword/fuzzy-rule based (inferred: the document mentions no training, model artefacts, or learned weights anywhere).

**Pipelines built:**
- Suggestion pipeline: OCR all uploads -> `suggest_keywords()` (n-gram sizes 3, 2, 1 at `utils.py:82`; min token length 6 at `utils.py:118`; min multi-doc count 2 at `utils.py:117`; `partial_ratio` clustering cutoff 80 at `utils.py:122`).
- Scoring pipeline: `extract_and_score()` (`label_trainer_streamlit.py`, PDF handling lines 53–64) -> per-file `keyword_score()` (`utils.py:87`, ratio cutoff 80) -> average -> threshold verdict (`label_trainer_streamlit.py:127`) -> JSON write.

**PoC-to-production traceability (the document's experimentation table, four trials):**
- `test.ipynb` cells 56–57 — automated n-gram extraction (`predict_keyword`): 1-, 2- and 3-word n-grams from sample OCR text, filtered by length (>6) and frequency (>=2), clustered via fuzzy `partial_ratio`. Carried into `utils.py:suggest_keywords()`, invoked by the "Get Keyword Suggestions" button at `label_trainer_streamlit.py:108`.
- `test.ipynb` cells 50, 58–64 — keyword match scoring against labels: evaluated the matched/total*100 formula using both `token_sort_ratio` and `ratio` scorers. Carried into `utils.py:keyword_score()`, used by `extract_and_score()` at `label_trainer_streamlit.py:78`.
- `test.ipynb` cells 25, 38, 51 — local RapidOCR vs Azure OCR evaluation, benchmarked for keyword-matching accuracy. Carried into production as the sidebar "OCR Engines" radio toggle, leaving engine choice with the user.
- `test.ipynb` cells 63–66 — acceptance-threshold calibration across 50%, 65% and 75% cutoffs to evaluate false-acceptance rates. Carried into production as the interactive slider at `label_trainer_streamlit.py:127` with default 75%. (The document records which cutoffs were tested but not the measured false-acceptance rates.)

**APIs integrated:** Azure Document Intelligence prebuilt-read (via `azure_ocr()`); the sibling FastAPI detection and extraction services consume the JSON store (integration by shared file, not HTTP). The trainer itself publishes no HTTP API — it runs under the Streamlit CLI runner, not an ASGI server, and its only transport is Streamlit's internal WebSocket protocol, which is why the Nginx gateway must set `proxy_set_header Upgrade $http_upgrade` when mapping `http://<server-host>:9000/` to `http://127.0.0.1:8000`.

**Data extraction methods:** PyMuPDF page-0 rasterization at 150 DPI (`matrix 150/72`, `label_trainer_streamlit.py:56`); JPEG encoding at quality 95 (`label_trainer_streamlit.py:59`); PIL for raster images; OCR line extraction from either engine.

**Validation logic:** Label must be non-empty and at least one file uploaded (else `st.error()` and halt); file types constrained to JPG, JPEG, PNG and PDF, enforced at the widget by `st.file_uploader`; keywords parsed from a comma-separated string of alphanumeric terms and phrases, whitespace-stripped during parsing; keyword match requires RapidFuzz ratio >= 80; per-file score = matched keywords / total keywords * 100; average across files must be >= slider threshold to persist; matching label names overwrite prior definitions. First-run safety: if `labels_DB/<region>_labels.json` does not exist when the region is confirmed, an empty dictionary is initialised on disk so later reads and writes cannot fail on a missing file.

**Automation workflows:** Automated keyword discovery replaces manual document inspection; accepted labels go live in production detectors without a deployment step.

**Optimization techniques:** Runtime-selectable OCR engine (local for speed, Azure for accuracy); uploads held in in-memory buffers; document-frequency and length filters suppress noise in suggestions. (Analyst interpretation: page-0-only rendering also bounds OCR cost per PDF, though the document lists it as a limitation.)

**Performance improvements:** No benchmarks recorded; the document offers only qualitative guidance (RapidOCR "faster but lower accuracy", Azure "higher accuracy but a bit slower").

## 7. Challenges and Solutions

1. **Non-engineers needed to change classifier behaviour safely (business).** Solved with a no-code UI plus a mandatory empirical acceptance test: keywords are scored against real samples and persisted only when the average clears the threshold. Alternatives (analyst suggestion): CSV import with offline validation, or a pull-request config workflow — slower and less accessible to operations staff.
2. **Manual discovery of discriminating keywords was slow (business).** Solved with n-gram frequency clustering (`suggest_keywords()`), filtered by length and cross-document frequency, near-duplicates merged via `partial_ratio`. Alternatives (analyst suggestion): TF-IDF against other labels to favour discriminative over merely frequent terms; embedding-based clustering.
3. **OCR quality vs speed (technical).** Solved by exposing RapidOCR and Azure prebuilt-read as a runtime toggle, informed by benchmarking in `test.ipynb` cells 25, 38, 51. Alternative (analyst suggestion): automatic fallback to Azure when RapidOCR confidence is low.
4. **Choosing the acceptance threshold (technical).** Calibrated 50%, 65%, 75% cutoffs for false-acceptance rates (cells 63–66); shipped 75% default with an operator slider. Alternative (analyst suggestion): per-label or per-region thresholds.
5. **Making changes live without restarts (technical).** Solved by writing to the same `labels_DB/<region>_labels.json` the FastAPI pipelines read. Alternatives (analyst suggestion): a versioned config table in a database, or a config-service endpoint with cache invalidation — more infrastructure but with audit and rollback.
6. **Jurisdiction isolation (business).** Solved with a mandatory region modal binding all reads/writes to one file per region. Alternative (analyst suggestion): a single store keyed by region with role-based access, at the cost of a larger blast radius per write.
7. **OCR noise in matching (technical).** Tolerated with RapidFuzz ratio >= 80; `ratio` and `token_sort_ratio` were both evaluated (cells 50, 58–64). Alternative (analyst suggestion): exact matching after text normalisation — cheaper but brittle against OCR character errors.
8. **Documented limitations (honest talking points):**
   - **Port discrepancy:** README launches on 8082 while `nginx.conf` proxies 9000 -> 127.0.0.1:8000; the app must be launched with `--server.port 8000` behind the gateway.
   - **PDF page limitation:** `extract_and_score()` (lines 53–64) OCRs only page 0; later pages are not evaluated during training.
   - **Single-document suggestion fallback:** at line 110, with one upload `suggest_keywords()` is bypassed and raw OCR lines `texts[0]` are returned as suggestions (analyst assessment: these skip the length/frequency filters and are likely noisier).
   - **Unpinned dependencies:** all packages unpinned in `requirements.txt` (the document's own technology table records "Unpinned in requirements.txt" for every dependency), so an upstream release of RapidOCR, RapidFuzz or PyMuPDF can change scoring behaviour between deployments.
   - **Overwrite semantics:** the document states existing labels with matching names "will overwrite previous keyword definitions upon submission"; no confirmation step, versioning, or rollback is described (analyst assessment: this makes the write destructive and unauditable).
   - **No REST interface:** the app runs under the Streamlit CLI runner rather than an ASGI application server and exposes no HTTP REST endpoints, so it cannot be driven programmatically or covered by API-level tests; Streamlit WebSocket only.
   - **No authentication or access control described (gap, not a documented defect):** the document specifies no auth, TLS, or authorization in front of a tool that mutates live production classification config, and the README launch binds `--server.address 0.0.0.0`. How the Azure Document Intelligence endpoint/key is supplied is also not documented.
   - **No quantified evaluation published:** the threshold and OCR-engine trials are named but their measured false-acceptance rates and accuracy figures are not recorded, so the 75% default cannot be independently justified from the document alone.

## 8. Impact Analysis

- **Business impact:** Lets operations staff author, test, and deploy document-classification rules without engineering intervention, and supports onboarding new document categories for regional expansions (Oman, Kuwait).
- **Productivity improvements:** Automated n-gram suggestions replace manual document inspection; changes go live instantly with no deployment cycle. No figures recorded.
- **Accuracy improvements:** Threshold governance (default 75%) keeps weak keyword sets out of production, preventing false positives; operators can validate with the higher-accuracy Azure engine. No accuracy numbers recorded.
- **Cost savings:** Not recorded. The document states: "Quantified business metrics — including operational hours saved, ROI figures, and throughput benchmarks — are not recorded in project files and are Not specified in the project."
- **Time savings:** Not quantified; qualitatively, zero-downtime activation and no-code authoring remove engineering and deployment latency.
- **User benefits:** Business analysts, operations teams, and document specialists get a guided, region-scoped curation workflow with previews, suggestions, and immediate pass/fail feedback; administrators and developers inspect OCR output and calibrate thresholds.

## 9. Interview Discussion Points

- Why Streamlit for an internal admin tool, and how you managed its rerun model with `st.session_state` (region, suggestions, form inputs) and a `@st.dialog` gate.
- How `suggest_keywords()` works (n-grams 3/2/1, min length 6, >=2-document frequency, `partial_ratio` clustering at 80) and — analyst view — why frequency alone is a weak proxy for discriminativeness, though the document calls the output "discriminating n-grams".
- The scoring formula (matched/total*100 with ratio >= 80) and why you compared `ratio` and `token_sort_ratio` on OCR-noisy text.
- How you calibrated the 75% default from 50/65/75 trials and why you still exposed a slider.
- The RapidOCR vs Azure prebuilt-read trade-off, how you benchmarked it, and when you would automate the choice.
- The hot-reload design: file-based config shared with FastAPI pipelines — simple, but with risks (no versioning, silent overwrite, concurrent writes, no audit trail).
- The three documented defects (port mismatch 8082/8000/9000, page-0-only PDF OCR, single-file raw-line fallback) and how you would fix each, including why Nginx must forward WebSocket upgrades to the port Streamlit actually listens on.
- Why 150 DPI and JPEG quality 95 — OCR quality versus latency and payload size.
- Production hardening: pinned dependencies, per-label thresholds, multi-page sampling, config versioning/rollback, authentication.
- How the tool fits the wider TextMatch platform and whether `utils.py` scoring functions are shared with the FastAPI services (the document confirms only that the JSON store is shared).
- Why the region gate is a *non-dismissible* modal rather than a sidebar dropdown: the region binds every subsequent read and write, so a mis-set dropdown would silently write one jurisdiction's keywords into another's file. Forcing the choice before any control renders makes cross-jurisdiction contamination structurally impossible within a session.
- Why the acceptance score is `matched / total * 100` (a coverage metric) rather than the mean similarity of the best matches: coverage answers "do all the keywords I claim actually appear in real samples", and it is explainable to a non-technical operator. Trade-off: it is insensitive to *how well* each keyword matched once past the ratio >= 80 cut, and a single unusable keyword costs a fixed fraction of the score regardless of how central it is.
- Why 80 was used both for the keyword match cut (`ratio`, `utils.py:87`) and the suggestion clustering cut (`partial_ratio`, `utils.py:122`), and whether those two cutoffs should really move together — they solve different problems (tolerating OCR character noise vs merging near-duplicate n-grams).
- Why the tool runs under the Streamlit CLI instead of an ASGI server, and what that costs: no REST surface means no programmatic label loading, no API-level regression tests, and the Nginx layer must forward WebSocket upgrades rather than plain HTTP.
- The build-vs-buy framing: this is a *no-code configuration console over a rule-based classifier*, not a trained model. Why a keyword/fuzzy rule set was the right level of machine intelligence for jurisdiction-specific insurance documents (explainable, auditable per label, editable by non-engineers, no labelled training corpus needed) — and at what document volume or category count you would switch to a learned text classifier.
- The un-discussed operational risk to raise honestly: the document describes no authentication in front of a tool that writes live production config, and no concurrency control on the shared regional JSON — two operators submitting at once is undefined behaviour in the design as documented.
- The dual audience of the UI (the document names both): operations staff authoring rules, and administrators/developers using the same screen to inspect OCR extraction quality and calibrate fuzzy thresholds — the tool is simultaneously a config editor and a debugging console, and why that shaped the sidebar layout.
- What you would measure if you rebuilt it: the document publishes no false-acceptance rate, no OCR accuracy delta between RapidOCR and Azure, and no throughput figures — how you would instrument accepted/rejected verdicts and downstream detection outcomes to close that loop.

## 10. Architecture Explanation Points

An operations analyst opens the app through Nginx (port 9000, WebSocket upgrade forwarded to Streamlit on 8000). A modal forces a region choice that binds the session to one file, `labels_DB/<region>_labels.json`. They drag in sample images or PDFs (PDF page 0 rendered by PyMuPDF at 150 DPI, JPEG-encoded at quality 95) and pick an OCR engine: local RapidOCR for speed or Azure prebuilt-read for accuracy. "Get Keyword Suggestions" runs OCR on every sample and calls `suggest_keywords()`: 3/2/1-grams, tokens of at least 6 characters present in at least two documents, clustered with RapidFuzz `partial_ratio` at 80. The analyst edits the label and keyword list, sets a threshold (default 75%), and submits. `extract_and_score()` OCRs each file, calls `keyword_score()` (matched if ratio >= 80; score = matched/total*100), averages across files, and writes to the regional JSON only if the average clears the threshold. Because the FastAPI detection and extraction services read the same file, the label is live immediately.

Structurally the document decomposes this into seven components — Streamlit reactive frontend, region selection modal, document preview and ingestion, OCR text extraction engine, suggestion clustering module, fuzzy scoring and evaluation, and the regional JSON file store — arranged as a single left-to-right pipeline (region selection -> Streamlit session state and reactive runtime -> PyMuPDF/OCR -> RapidFuzz scoring and threshold check -> `labels_DB/*.json` update). There is no database, no queue, and no service boundary inside the tool: the entire system is one Streamlit process plus a directory of JSON files, which is precisely what makes hot reload free.

Key decisions and why: **file-based config** rather than a database, because the FastAPI detection and extraction pipelines already read `labels_DB/`, so a write *is* the deployment — zero downtime, zero coordination; **a non-dismissible region modal** rather than a dropdown, because region scopes every read and write and a wrong dropdown value would corrupt another jurisdiction's dictionary; **a runtime OCR toggle** rather than one fixed engine, because the notebook benchmark (cells 25, 38, 51) showed a genuine speed/accuracy trade-off with no dominant winner, so the choice was pushed to the operator instead of hard-coded; **an empirical acceptance gate** rather than trusting operator judgement, because keywords that look right are not necessarily present in real OCR output; **an adjustable slider on top of a calibrated 75% default**, because calibration (cells 63–66) fixed a sensible default while leaving room for label-specific strictness; and **`utils.py` promoted from `test.ipynb`** so the shipped scorer is literally the experimented scorer (whether the FastAPI services import that module or only read the JSON store is not stated).

Trade-offs accepted (analyst assessment): simplicity over auditability — the write overwrites a same-name label with no versioning, rollback, confirmation, or concurrency control; page-0-only OCR bounds cost and latency per PDF but means multi-page evidence never influences training; frequency-based suggestion is cheap and needs no corpus, but frequency is not discriminativeness — a term common to *every* document scores just as well as one unique to this label; the coverage-style score (matched/total*100) is explainable but blind to match quality above the 80 cut; and running under the Streamlit CLI keeps deployment to one command at the cost of any programmable API surface. Unaddressed in the documentation: authentication in front of a tool that mutates production config, and how the Azure credentials reach the process.

Next: fix the port mismatch (8082 in README vs 8000 behind the 9000 gateway), sample multiple PDF pages, replace the single-file raw-line fallback so filters still apply, pin dependencies, add versioned config with rollback, per-label or per-region thresholds, authentication and an audit trail on writes, and consider TF-IDF or cross-label contrastive scoring so suggestions favour discriminative terms over merely frequent ones.
