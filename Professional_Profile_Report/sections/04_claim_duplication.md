# Project 4: Claim Duplication Detector API

## 1. Project Overview

- **Project name:** Claim Duplication Detector API (document no. AT/duplicate/V1, dated 01-Sep-2026); companion Streamlit front-end `claimapp.py`, titled "Claim Duplication Detector" (browser tab "Claim Duplication Checker").
- **Business domain:** Insurance — medical claims intake and pre-screening for Qatar Insurance Group; the sole named target user is the Medical claim department.
- **Problem statement (inferred — the document states capabilities, not a problem):** Claim submissions arrive as batches of PDFs in which the same page can recur (the document cites "re-scanned or re-exported copies of the same page"), blank pages need to be identified, and pages may have been AI-generated or "modified or edited before submitting into claim". Without tooling, reviewers would have to page through every document, and no page-level audit record would exist.
- **Project objective:** Expose a single REST endpoint, `POST /analyze`, that accepts one or more PDFs in one multipart request, renders every page to an image, and returns an explicit per-page verdict — `Unique`, `Blank Page`, or `Duplicate` (with the matched source page) — plus an aggregated summary. The Streamlit UI adds a per-row AI-vs-human label and softmax confidence from a Hugging Face classifier and a `File_modified` flag from PDF metadata.
- **Expected business outcome:** Automated pre-screening of claim batches: "AI, Blank and duplicate pages" flagged before human review, duplicates matched across all files in the batch, and an auditable run record in `api.log`. The document explicitly limits benefit claims to "where supported by implemented functionality or recorded project evidence" and provides no accuracy, cost, or time-saving figures.

## 2. Role and Responsibilities

**Core responsibilities** (inferred from the system's contents; the document does not name a role):

- Designed and implemented the FastAPI service in `main.py`: `app = FastAPI(title="Claim Duplication Detector API")`, an `async def analyze_documents(...)` handler on `POST /analyze` accepting `files: List[UploadFile] = File(...)`, and a custom OpenAPI schema generator.
- Built the PDF rasterization stage with PyMuPDF (`fitz.open(stream=..., filetype="pdf")`, `page.get_pixmap(matrix=fitz.Matrix(2, 2))`) writing 2x PNGs to a per-request temporary directory deleted on completion.
- Implemented blank-page detection in OpenCV (grayscale read, threshold 240, `cv2.countNonZero`, `white_ratio > 0.99`).
- Implemented cross-document duplicate detection with Pillow + ImageHash (`imagehash.phash`), a request-scoped `global_hashes` dictionary, Hamming distance `< 5`, first-match-wins.
- Integrated the Hugging Face model `Ateeqq/ai-vs-human-image-detector` on PyTorch for per-page AI-vs-human classification with softmax confidence.
- Implemented two documented forensic fallbacks: AI generation (frequency threshold 0.62 — spectral peak concentration / 0.62, clamped to 1.0) and AI editing (ELA weight 0.5 — `min(ela_score * 1.5, 1.0)` from a second, independent ELA pass).
- Implemented the PDF-metadata `File_modified` check (`creationDate` vs `modDate`, `claimapp.py:78`).
- Defined the JSON response contract (`status`, `summary.*`, `results[]`) and the 400/500 error model.
- Built the Streamlit UI (`claimapp.py`) and dual-sink logging to stdout and `api.log`.

**Supporting tasks:**

- Authored `README.md` (Python version requirement "3.8 or higher", Swagger UI and Postman as tested clients, an example response for 2 documents / 4 pages / 1 blank / 1 duplicate).
- Wrote `generate_doc.py`, which carries the documented `curl` invocation: `curl -X POST "http://127.0.0.1:8000/analyze" -H "accept: application/json" -H "Content-Type: multipart/form-data" -F "files=@claim_doc_1.pdf" -F "files=@claim_doc_2.pdf"`.
- Declared dependencies with version floors in `requirements.txt` — FastAPI >=0.100.0, Uvicorn >=0.22.0, python-multipart >=0.0.6, PyMuPDF (fitz) >=1.22.0, opencv-python-headless >=4.7.0.0, ImageHash >=4.3.1, Pillow >=9.5.0, PyTorch (torch, torchvision) >=2.1; Hugging Face transformers is listed in the stack table with version `-`, and Streamlit does not appear in that table at all (no lockfile or virtual environment committed; exact installed versions not specified).
- Documented development (`uvicorn main:app --host 127.0.0.1 --port 8000 --reload`) and production (`uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4`) launch commands, plus `streamlit run claimapp.py` for the UI (the file holding the Uvicorn commands is not stated).
- Git hygiene: `api.log` is git-ignored.

**Estimated ownership level: AI Engineer (inferred).**
Rationale: the document describes one module (`main.py`) plus one UI script (`claimapp.py`) covering every layer — HTTP handling, PDF rendering, classical CV, perceptual hashing, a pretrained classifier, forensic fallbacks, metadata analysis, UI, logging, docs — with no mention of a team or separate ML owner: end-to-end ownership. It falls short of Senior AI Engineer or Solution Architect because the scope is one endpoint with no database or external service, no evaluation dataset, tests, containerization, or CI mentioned anywhere in the document, and unresolved documented inconsistencies (Section 7). It exceeds Developer because the core value comes from selecting and tuning CV/ML techniques.

## 3. End-to-End Workflow

**Processing stages**

1. **Upload.** A client (Swagger UI and Postman are the documented tested clients; curl is documented in `generate_doc.py`) sends `multipart/form-data` with one or more `files` fields to `POST /analyze`. The Streamlit app (`claimapp.py`) runs its own batch loop (its spinner "wraps the whole batch loop"; PDF metadata is read at `claimapp.py:46`); whether it calls `POST /analyze` or re-implements the pipeline is not stated.
2. **Validation.** If `not files` or every `f.filename == ""`, return HTTP 400 `"Please upload at least one PDF file to analyze."`.
3. **PDF open and rasterization.** Each upload opens in memory via `fitz.open(stream=..., filetype="pdf")`; every page renders via `page.get_pixmap(matrix=fitz.Matrix(2, 2))` to PNG in a per-request temp directory.
4. **Blank check.** OpenCV reads the PNG in grayscale (`cv2.imread`), thresholds at 240 (`cv2.threshold`; pixels above 240 are treated as white), counts white pixels with `cv2.countNonZero`, and marks `Blank Page` when `white_ratio > 0.99`; blank pages are neither hashed nor used as match targets.
5. **Perceptual hash and compare.** For non-blank pages, `imagehash.phash` (via Pillow) is compared against every entry in the request-scoped `global_hashes` map spanning all previously processed non-blank pages of all files. If `phash - old_hash < 5`, the page is `Duplicate` with `Similar To = "<filename> (Page <n>)"` of the first match (loop breaks); otherwise `Unique`. Non-blank pages become match targets for later pages (whether duplicate pages' hashes are also stored is not stated).
6. **AI-vs-human classification.** The document states the detector runs "over every rendered page" (so it is not gated by the blank check); `Ateeqq/ai-vs-human-image-detector` returns a label and softmax confidence (2 decimals, suffixed `' %'`) that surface only as `claimapp.py` table columns, not in the `POST /analyze` schema. Verified constants list two forensic fallbacks: generation (frequency threshold 0.62: spectral peak concentration / 0.62, clamped to 1.0) and editing (ELA weight 0.5: `min(ela_score * 1.5, 1.0)` from a second, independent ELA pass); their host module is not stated.
7. **Metadata check.** `creationDate` vs `modDate`, read once per document (`claimapp.py:46`), sets `File_modified` (`claimapp.py:78`), a `claimapp.py`-only column.
8. **Aggregation and response.** `summary.documents_processed` (`len(files)`), `summary.total_pages`, `summary.blank_pages`, `summary.duplicate_pages`; `results[]` holds one object per page in processing order. The temp directory is removed.
9. **Audit.** Events are logged to stdout and `api.log`. Any exception yields HTTP 500 `"Error analyzing documents: <exception text>"`.

**Data flow**

- In: multipart PDF bytes → in-memory byte stream per file.
- Intermediate: PNG page images (2x scale) in a request-scoped temp directory; grayscale arrays in OpenCV; pHash values in `global_hashes`, each tied to its source filename and page so `Similar To` can be populated (key structure not stated); classifier input tensors (inferred from the PyTorch runtime).
- Out: JSON (`status`, `summary`, `results[]`) from the API; a Streamlit table adding `File_modified`, `AI label`, `AI confidence`; log lines to stdout and `api.log`.

**Integrations and dependencies**

- Libraries: FastAPI, Uvicorn, python-multipart, PyMuPDF, opencv-python-headless, ImageHash, Pillow, PyTorch (torch, torchvision), Hugging Face transformers — version floors in Section 4, all declared as `requirements.txt` constraints by the document's stack table. Streamlit is required to run `claimapp.py` but is absent from that table and therefore has no documented version constraint (analyst observation).
- Model: `Ateeqq/ai-vs-human-image-detector` (Hugging Face Hub; label set comes from the model config).
- Explicitly none: no database, message queue, external API call, or persistent storage. `api.log` is the only file written outside the request temp directory.

**Flow line**

`Client (Swagger/Postman/curl) -> FastAPI POST /analyze -> PyMuPDF render 2x PNG -> OpenCV blank check (240 / 0.99) -> Pillow+ImageHash pHash vs global_hashes (<5) -> JSON summary + per-page results -> api.log`; the Streamlit app additionally surfaces `[HF AI-vs-human classifier + PDF metadata File_modified]` per row, and the ELA/spectral fallback constants are documented without a stated host module.

The document's Figure 1 names five stages: Client ("Upload Multiple PDFs in single request") → FastAPI Gateway ("Validates and routes requests") → PyMuPDF Rendering ("Renders pages to 2x images") → OpenCV + Metadata + ImageHash + AI detector ("Detects blanks, duplicates and fake images") → JSON Recommendations ("Returns final output").

**Architecture Summary**

A deliberately minimal, stateless analysis service: uploads are byte streams, rendered pages live in a temporary directory removed on completion, and the duplicate index (`global_hashes`) exists for one request only. The analysis layer orders a cheap check before a more expensive one — pixel-ratio blank detection gates perceptual hashing — while the AI-vs-human detector is documented as running over every rendered page (not gated by the blank check). No shared state means horizontal scale via Uvicorn workers (`--workers 4`); the trade-off is that duplicate detection is bounded by the batch, never by historical submissions.

## 4. Technologies and Tools Used

| Category | Technologies (as stated in the document) |
| --- | --- |
| Programming languages | Python — README: "Python 3.8 or higher"; bytecode `main.cpython-310.pyc` indicates CPython 3.10 |
| Frameworks | FastAPI >=0.100.0; PyTorch (torch, torchvision) >=2.1 ("ML runtime"); Hugging Face transformers (version cell is `-`, i.e. not stated); Streamlit — used by `claimapp.py` and launched with `streamlit run claimapp.py`, but absent from the document's technology stack table, so no version constraint is documented |
| Libraries | PyMuPDF (fitz) >=1.22.0 ("PDF processing & Metadata"); opencv-python-headless >=4.7.0.0 ("Computer Vision"); ImageHash >=4.3.1 ("Perceptual hashing"); Pillow >=9.5.0 ("Imaging"); python-multipart >=0.0.6 ("Multipart parsing"). All values are `requirements.txt` constraints; the document states exact installed versions are unspecified (no lockfile or virtual environment committed) |
| AI/ML models | `Ateeqq/ai-vs-human-image-detector` (Hugging Face) |
| OCR tools | Not stated in documentation (no OCR is performed; matching is image-based) |
| Databases | Not stated in documentation (document states none exists) |
| Cloud platforms | Not stated in documentation |
| APIs | `POST /analyze` (multipart/form-data REST endpoint); auto-generated OpenAPI schema via a custom schema generator; Swagger UI and Postman documented as tested clients |
| DevOps tools | Not stated in documentation (the document states no lockfile or virtual environment is committed; containers and CI are never mentioned) |
| Version control | Git (inferred from "api.log ... is git-ignored") |
| Deployment tools | Uvicorn >=0.22.0 — dev: `uvicorn main:app --host 127.0.0.1 --port 8000 --reload`; prod: `uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4`; UI: `streamlit run claimapp.py` |
| Document processing tools | PyMuPDF (PDF open from bytes, page rendering, metadata read) |
| Automation tools | Not stated in documentation |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Single-module FastAPI service with an async handler; explicit 400/500 error model; per-request temp-directory lifecycle; dual-sink logging (stdout + `api.log`); dependency floors in `requirements.txt`; README documenting the Python floor, tested clients and an example response; `generate_doc.py` carrying the documented `curl` form (its wider purpose is not stated in the document); separate dev (`--reload`, 127.0.0.1:8000) and prod (0.0.0.0:8000, `--workers 4`) run commands; `api.log` kept out of version control.
- **AI Engineering:** Integration of a pretrained Hugging Face image classifier (`Ateeqq/ai-vs-human-image-detector`) with softmax-confidence reporting; layered fallback scoring (spectral peak concentration / 0.62; ELA `min(ela_score * 1.5, 1.0)`).
- **Machine Learning:** Inference-only use of a transformers vision model on PyTorch; label mapping via `model.config.id2label`. No training or fine-tuning is documented.
- **NLP:** Not demonstrated in this project.
- **Computer Vision:** Grayscale thresholding and pixel-ratio analysis for blank detection (threshold 240, ratio 0.99); perceptual hashing with pHash and Hamming-distance matching (< 5); 2x rasterization before analysis; Error Level Analysis and frequency-domain (spectral) analysis for tamper/generation detection.
- **Data Engineering:** In-memory byte-stream handling of multi-file uploads; request-scoped hash index; aggregation into a fixed JSON schema and a typed Streamlit table (TextColumn/NumberColumn configuration).
- **Cloud:** Not demonstrated in this project.
- **MLOps:** Not demonstrated in this project (no model versioning, monitoring, evaluation set, or CI is documented).
- **API Development:** Multipart file-list endpoint (`List[UploadFile] = File(...)`); OpenAPI customization; documented curl/Swagger/Postman usage; explicit response contract with per-page `Similar To` references.
- **Prompt Engineering:** Not demonstrated in this project.
- **System Design:** Stateless, request-scoped design with no external dependencies; cheap-to-expensive check ordering (blank gate before hashing — the AI detector is documented as running over every rendered page); horizontal scale-out via Uvicorn workers; content-based rather than metadata-based matching; REST + auto-generated OpenAPI schema as the integration surface.

## 6. Detailed Technical Contributions

**Features implemented**

- `POST /analyze` accepting one or more PDFs (`files` repeatable field, `Content-Type: multipart/form-data`, no other parameters).
- Per-page classification into `Unique`, `Blank Page`, or `Duplicate`, with `Similar To` set to `"<filename> (Page <n>)"` or `"-"`.
- Batch summary: `documents_processed`, `total_pages`, `blank_pages`, `duplicate_pages`.
- Cross-document duplicate matching within a request via a single `global_hashes` map.
- AI-vs-human page label and confidence (Streamlit only; the detector runs over every rendered page; neither column has a `column_config` entry).
- `File_modified` flag from PDF metadata (Streamlit only): `'No'` when `creationDate` equals `modDate`, otherwise `'Yes'` (`claimapp.py:78`); read once per document (`claimapp.py:46`) and repeated on every page row of that document; no `column_config` entry, so it renders at default width.
- Audit logging to stdout and `api.log` (file names, page counts, blank detections, duplicate matches, final totals).
- Documented example response (README.md): `documents_processed: 2`, `total_pages: 4`, `blank_pages: 1`, `duplicate_pages: 1`, with `claim_invoice_2.pdf` page 1 reported as `Duplicate` of `claim_invoice_1.pdf (Page 1)`.
- Streamlit UI: `st.set_page_config(page_title="Claim Duplication Checker", layout="wide")`, `st.title("Claim Duplication Detector")`, `st.file_uploader("Upload PDF Claims", accept_multiple_files=True)` (documented intent `type=["pdf"]`, but created without `type`), `st.button("Analyze Documents")`, `st.spinner("Analyzing documents... This may take a moment.")` wrapping the whole batch loop, and `st.warning("Please upload at least one PDF file to analyze.")` plus a WARNING log line on empty submission; result table with `Document` (TextColumn large), `Page` (NumberColumn small), `Status` and `Similar To` (TextColumn medium).

**Models used**

- `Ateeqq/ai-vs-human-image-detector` (Hugging Face transformers, PyTorch runtime). Output: `model.config.id2label[predicted_class]` and softmax probability of the predicted class as a percentage string rounded to 2 decimals with trailing `' %'` (`claimapp.py:30`).

**Pipelines built**

- Render → blank gate → hash/compare → aggregate, per page in file order within the request. Blanks short-circuit hashing ("Blank wins over duplicate, because a blank page is never hashed"); comparison breaks on the first match.

**APIs integrated**

- None external (the document states no external API call exists); the service exposes the REST endpoint and an OpenAPI schema. The Hugging Face model `Ateeqq/ai-vs-human-image-detector` is loaded via transformers (how it is fetched or cached is not stated). Documented request contract: `Content-Type: multipart/form-data`, field `files` (repeatable, type File, required), expected format PDF, no other parameters; documented curl: `-H "accept: application/json" -H "Content-Type: multipart/form-data" -F "files=@claim_doc_1.pdf" -F "files=@claim_doc_2.pdf"` against `http://127.0.0.1:8000/analyze`.

**Data extraction methods**

- PDF bytes → pages via PyMuPDF; page → PNG at `fitz.Matrix(2, 2)`; PNG → grayscale array (OpenCV); PNG → pHash (Pillow + ImageHash); PDF metadata → `creationDate`/`modDate`.

**Validation logic**

- Request-level: 400 when `not files or all(f.filename == "" for f in files)`, with `detail` = "Please upload at least one PDF file to analyze." Because the guard is `all(...)`, a mixed batch containing one empty filename still proceeds (analyst observation). The request contract admits no other parameters ("Other parameters: None"), and the expected format is PDF only by virtue of `fitz.open(..., filetype="pdf")` — no explicit content-type or magic-byte check is documented.
- Batch-level: `summary.documents_processed` is `len(files)`, i.e. the number of files *received*, not the number successfully parsed (analyst observation).
- Page-level: blank if `white_ratio > 0.99` after threshold 240; duplicate if pHash Hamming distance `< 5`; hashing scope is non-blank pages only.
- Metadata-level: `File_modified = 'Yes'` when `creationDate != modDate`.
- Forensic-level: AI generation fallback (frequency threshold 0.62, spectral peak concentration / 0.62 clamped to 1.0); AI editing fallback (ELA weight 0.5, `min(ela_score * 1.5, 1.0)`, second independent ELA pass).
- Error-level: "Any exception raised during analysis" → 500 with `detail` = "Error analyzing documents: <exception text>". The handler is documented as catching any exception for the whole request, so a single unreadable PDF fails the entire batch rather than being skipped per file, and the document does not say whether the temp directory is still removed on that path — it states only that the directory "is removed when the request completes" (analyst observations).

**Automation workflows**

- Not stated in documentation beyond the single-call batch analysis itself.

**Optimization techniques**

- Blank detection precedes hashing, so blanks cost no hash computation and never pollute the match index (the AI detector, by contrast, is documented as running over every rendered page).
- In-memory PDF opening (`stream=`) so uploads are not written to disk (inferred); the temp directory holds the rendered PNGs and is removed when the request completes.
- Stateless design allows `--workers 4` multi-process serving.

**Performance improvements**

- No before/after figures are given; the sole evidence is the `api.log` run dated 2026-06-04 (4 files, 9 pages, 1 blank page, 3 duplicates, log timestamps spanning approximately 7 seconds).

## 7. Challenges and Solutions

**1. Matching duplicates that are not byte-identical (technical).** Re-scanned or re-exported pages differ in bytes and metadata. Solution: perceptual hashing on the rendered 2x image with Hamming distance `< 5` so small scan differences still match. Alternatives (analyst suggestion): exact checksums (fail on re-scans), SSIM or ORB/SIFT matching (robust to crops/rotation, far slower), CNN/CLIP embeddings (heavier runtime).

**2. Detecting blank pages cheaply (technical).** Solution: grayscale threshold 240 and white-pixel ratio above 0.99, placed before hashing so blanks never enter the index. Alternatives (analyst suggestion): OCR text-presence check (slower), pixel-variance or entropy tests (robust to grey scanner backgrounds a fixed threshold misses).

**3. Cross-file rather than per-file matching (technical/business).** Solution: one `global_hashes` map per request so page 1 of file 2 can match page 1 of file 1. Limitation: no persistent storage, so historical submissions are never consulted. Alternative (analyst suggestion): a persistent hash index keyed by claim/member ID with a BK-tree for sub-linear Hamming search.

**4. Flagging AI-generated or edited pages (technical).** Solution: Hugging Face classifier plus two forensic fallbacks (generation: frequency threshold 0.62, spectral peak concentration / 0.62 clamped to 1.0; editing: ELA weight 0.5, `min(ela_score * 1.5, 1.0)` from a second independent ELA pass) and the metadata `File_modified` check. Documented limitation: the label set comes from the model config, and no accuracy on claim documents is reported. Alternative (analyst suggestion): ensemble voting across classifier and forensic scores, or a detector fine-tuned on scanned medical documents.

**5. Documented inconsistencies and limitations (technical).**
- The Streamlit uploader is documented as `type=["pdf"]` but is created without the `type` argument, so it accepts any file type.
- The API response (`Document`, `Page`, `Status`, `Similar To`) omits AI label, AI confidence, and `File_modified`, which exist in `claimapp.py` only although the executive summary lists them as key capabilities.
- First match wins: only one `Similar To` reference per page; blank pages that are also duplicates are reported only as blank.
- Linear comparison against all prior hashes, O(n^2) per batch (analyst observation).
- The 500 handler returns raw exception text; no authentication is documented.
- The "Swagger UI Version" section is empty (heading present, no content); the dependency count is stated as "seven" while the stack table lists nine non-Python technologies. The seven map exactly onto the non-ML rows (FastAPI, Uvicorn, python-multipart, PyMuPDF, opencv-python-headless, ImageHash, Pillow), which suggests torch/torchvision and transformers are not actually in `requirements.txt` (analyst observation; the transformers version cell is `-`). Streamlit is needed to run `claimapp.py` yet appears in neither the stack table nor the dependency count (analyst observation). README says Python 3.8+ while the compiled bytecode `main.cpython-310.pyc` shows CPython 3.10; no lockfile or virtual environment is committed, so exact installed versions are unknown.
- Request validation is thin: `filetype="pdf"` in `fitz.open` is the only format gate documented, the 400 guard fires only when *all* filenames are empty, and no file-size limit, page-count limit, or rate limit is documented (analyst observation).
- One unhandled exception fails the whole batch with a single 500 rather than degrading per file, and `documents_processed` counts files received (`len(files)`) rather than files parsed (analyst observation).
- The constants table labels the editing fallback "ELA weight 0.5" yet its formula uses a 1.5 multiplier; the relationship is unexplained, and the document does not say whether fallback scores surface in the API response or the UI.
- The AI-vs-human detector is documented as running "over every rendered page", so blank pages still incur model inference even though they are skipped for hashing (analyst observation).
- The Streamlit app appears to run its own batch loop (spinner "wraps the whole batch loop"; metadata read at `claimapp.py:46`) rather than being documented as a client of `POST /analyze`, so the detection logic may exist in two places (analyst observation; not stated either way).

**6. Auditability without a database (business).** Solution: every run writes file names, page counts, blank detections, duplicate matches, and totals to `api.log` and stdout. Alternative (analyst suggestion): structured JSON logs shipped to a central store, or per-request results persisted by claim ID.

## 8. Impact Analysis

- **Business impact:** Automated page-level pre-screening of claim batches: an explicit verdict per page, duplicates matched across files, content-based matching that catches re-scans despite differing metadata, and a REST/OpenAPI surface intake portals can call without bespoke tooling. The document lists exactly four strategic benefits — standard integration surface (REST + auto-generated OpenAPI), content-based rather than metadata-based matching, a horizontal scalability path (stateless per request, documented under Uvicorn `--workers 4`), and deployment simplicity ("Seven declared Python dependencies, one module, no database or external service dependency") — and three operational ones: automated pre-screening with an explicit page-level verdict for every page received, batch handling in a single call with cross-file matching, and an auditable run record.
- **Productivity improvements:** Qualitative only: reviewers get a per-page table instead of reading every page; no throughput or time-per-claim figures are given.
- **Accuracy improvements:** Not stated. No precision/recall, false-positive rate, or evaluation set is documented for any of the three checks.
- **Cost savings:** Not stated in documentation.
- **Time savings:** Not stated as a metric. The only timing evidence is the single logged run: 4 files, 9 pages, 1 blank, 3 duplicates, approximately 7 seconds.
- **User benefits:** Medical claim department: batch upload in one call, a per-page Streamlit table (`Status`, `Similar To`, `File_modified`, `AI label`, `AI confidence`), and an `api.log` audit trail. Operations: a stateless one-module service with seven declared dependencies and no database.

## 9. Interview Discussion Points

1. **Why pHash and why Hamming distance 5?** pHash works on the DCT of a downscaled grayscale image, so re-scans hash near-identically; `< 5` on the default 64-bit imagehash pHash (analyst note) tolerates minor noise. Say plainly the threshold was never validated against a labeled set.
2. **Why render at 2x?** The document gives no rationale; argue it stabilizes the blank ratio and the hash for low-content pages, at a memory/time cost per page.
3. **Where does threshold 240 / ratio 0.99 fail?** Grey scanner backgrounds, faint stamps, near-empty pages; discuss entropy/variance alternatives.
4. **Blank check before hashing.** Cost benefit, and the side effect that blank duplicates are reported as blank only.
5. **First-match-wins and O(n^2) comparison.** The document states match selection is "First match wins (break)", so only one `Similar To` reference is reported per page; the quadratic cost of comparing each page against every prior hash is an analyst observation. What changes for a 500-page batch; BK-tree or persistent index.
6. **Request-scoped `global_hashes`.** Why it enables `--workers 4`, and why historical duplicates are invisible.
7. **CPU-bound work in an `async def` handler.** Rendering, OpenCV, and PyTorch inference block the event loop; threadpool or task-queue options (analyst observation).
8. **Classifier plus ELA/spectral fallbacks.** Why a pretrained Hugging Face model, what 0.62, 0.5 and 1.5 mean, and — since one logged run is the only evidence — how you would build a labeled set and report precision/recall on real claim scans.
9. **API vs UI divergence.** AI and metadata fields exist only in `claimapp.py`; how to unify the contract.
10. **Security and robustness.** No authentication is documented, raw exception text in 500s, uploader accepts any file type; hardening plan.
11. **Model loading.** The document does not say whether the transformers model loads in the API or only in `claimapp.py`; if in the API, four Uvicorn workers each load it — memory and cold-start implications.
12. **AI detector on every rendered page.** The document says the detector runs "over every rendered page", so blank pages are skipped for hashing but not for inference; discuss whether to extend the blank gate to the model and what that saves per batch (analyst observation).
13. **`File_modified` from `creationDate` vs `modDate`.** Explain why this is a weak signal (many PDF producers write differing or identical timestamps regardless of edits, and metadata is trivially rewritable) and why it is surfaced only as a flag alongside the classifier rather than as a verdict (analyst note).
14. **Streamlit app vs API.** The UI runs its own batch loop and adds columns the API lacks; discuss whether it should be a thin client of `POST /analyze` to keep one implementation of the detection logic (analyst suggestion; the document does not state how the UI obtains results).
15. **Interpreting the example response.** Walk through the README example (2 documents, 4 pages, 1 blank, 1 duplicate: `claim_invoice_2.pdf` page 1 matched to `claim_invoice_1.pdf (Page 1)`) to show how `Similar To` cross-references pages across files.
16. **Batch failure semantics and input validation.** Any exception during analysis returns one 500 for the whole request ("Error analyzing documents: <exception text>"), the 400 guard trips only when *every* filename is empty, and `documents_processed` is `len(files)` — files received, not parsed. Discuss per-file error isolation, a partial-success response shape, and file-size/page-count limits (analyst observations).
17. **Why no OCR, and where image-only matching stops.** No OCR tool appears anywhere in the document; matching is purely pixel-level. Explain when that is sufficient (the same render re-scanned or re-exported) and when text extraction would be required (the same invoice re-typed, re-formatted, or re-issued with a new layout — semantically duplicate but perceptually different) (analyst note).
18. **Deployment story and what is missing from it.** Dev `uvicorn main:app --host 127.0.0.1 --port 8000 --reload` vs prod `--host 0.0.0.0 --port 8000 --workers 4`, with "seven declared Python dependencies, one module, no database or external service dependency" as the stated simplicity argument; then name what is absent — container image, CI, lockfile, virtual environment, health endpoint, TLS termination (analyst observation).
19. **Reading the constants table as an engineering artifact.** The document publishes eight verified constants (2x render matrix, threshold 240, ratio 0.99, distance < 5, non-blank-only hashing scope, first-match-wins, frequency threshold 0.62, ELA weight 0.5 with `min(ela_score * 1.5, 1.0)`). Being able to recite where each one lives, what it trades off, and that none was tuned against a labeled set is the strongest honest position available on this project.

## 10. Architecture Explanation Points

Start at the left, following the document's Figure 1 (Client → FastAPI Gateway → PyMuPDF Rendering → OpenCV + Metadata + ImageHash + AI detector → JSON Recommendations): a client (Swagger UI or Postman as documented tested clients, or curl) POSTs a multipart batch of PDFs to `/analyze`; the gateway validates (400 if no files or all filenames empty) and routes to the async `analyze_documents` handler. Draw one box for `main.py`; everything happens inside one request. Stages inside: PyMuPDF opens each PDF from bytes and renders every page at 2x to PNG in a temp directory; OpenCV thresholds at 240 and, if over 99% of pixels are white, the page is `Blank Page` and skips hashing; otherwise ImageHash computes a pHash and compares it against a request-scoped dictionary of every non-blank hash seen across all files in the batch, and Hamming distance under 5 marks `Duplicate` with `Similar To` pointing at the first match. The Streamlit app (own batch loop; relationship to the API not stated) adds the Hugging Face AI-vs-human label and confidence — run over every rendered page — and a metadata `File_modified` flag; ELA/spectral fallback constants are documented without a stated host. Output is a JSON summary plus one row per page, mirrored to `api.log`; the temp directory is removed when the request completes.

Design decisions: content-based matching so re-scans are caught even when metadata differs; cheap check first so blanks never reach the hash index (the AI detector is not gated); stateless per request so it scales with `--workers 4`; zero external dependencies (no database, message queue, external API, or persistent storage); REST plus auto-generated OpenAPI schema as the integration surface.

Trade-offs: batch-bounded duplicate detection (no persistent storage means a page duplicated against last month's submission is invisible); hard-coded thresholds (240, 0.99, < 5, 0.62, 0.5/1.5) chosen without any documented evaluation set; first-match-wins, so a page duplicated three times reports only one `Similar To`; blank-wins-over-duplicate, so a blank page that is also a repeat is reported as blank only; linear hash comparison against every prior page; CPU-bound rendering, OpenCV and PyTorch inference inside an `async def` handler; an API contract that lags the UI (no AI label, confidence or `File_modified`); model inference on blank pages; a `File_modified` signal that depends on easily altered PDF metadata; and all-or-nothing batch error handling — one exception returns a single 500 for the whole request with raw exception text, and no authentication or file-type/size validation beyond `filetype="pdf"` is documented.

Next improvements (analyst suggestions — none of these is proposed by the document): persist hashes keyed by claim/member ID with a BK-tree for sub-linear historical matching; move rendering and inference to a worker pool or task queue so the event loop stays free; promote AI label, confidence, and `File_modified` into the API response so the UI becomes a thin client of `POST /analyze` and the detection logic lives in one place; add PDF type/size validation, authentication and sanitized 500 responses; isolate per-file failures so one bad PDF does not sink a batch; build a labeled evaluation set of real claim scans and report precision/recall for blank, duplicate and AI-vs-human checks; pin dependencies in a lockfile (including Streamlit, torch and transformers, which the seven-dependency count appears to omit) and containerize.
