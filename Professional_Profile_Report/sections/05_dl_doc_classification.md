# Project 5: Document Classification Model (Deep Learning)

Source: `DL_doc_detection_Documentation.md` — Document no. AT/classification/V1, 01-Sep-2026, client Qatar Insurance Group. Service name in code: "Motor Doc Check AI" (FastAPI title, app.py:16, version "0.1"). The document never names the candidate's role; role statements are inferred from the artefacts it describes.

## 1. Project Overview

- **Project name:** Motor Doc Check AI — Document Classification Model (Deep Learning), Document no. AT/classification/V1.
- **Business domain:** Insurance — UAE motor claims document intake for Qatar Insurance Group. Stated target users: "All UAE Motor claim projects fro doc classification" (sic).
- **Problem statement:** Motor claim submissions arrive as images or PDFs of identity and vehicle documents (driving licence, resident, vehicle licence — front and back) that must be typed before downstream processing (inferred from the six classes; the document does not describe the downstream flow). The incumbent path depends on OCR extraction, recorded as taking "around 7 – 10 seconds depends on the ocr extraction". The stated aim is an OCR-free classifier that "helps us with a second"; offline and CPU-only operation are described under operational benefits rather than as stated requirements.
- **Project objective:** Build an offline, single-process HTTP service that accepts one uploaded image or PDF and returns the document class plus the classifier's probability, using a frozen DINOv2-base vision transformer for 768-d embeddings and a scikit-learn LogisticRegression head; make retraining and adding a class a directory-level operation with a tiny, version-controllable artefact.
- **Expected business outcome:** Fast document-type identification for motor claims intake — the stated aim is "this helps us with a second" against a traditional path of "around 7 – 10 seconds depends on the ocr extraction", while the document also states "No per-request latency figure is recorded anywhere in the project." Downstream routing and consumers of the label are not described (any "routing" use is inferred). Further stated outcomes: zero outbound network calls at request time (only the optional `/docs` Swagger UI page reaches a CDN), no GPU requirement (automatic CPU fallback, train.py:13), and a 19,442-byte retrainable head — "the 346 MB transformer never changes" — so a new document type is "a directory operation" plus a retrain with no code change.

## 2. Role and Responsibilities

**Core responsibilities (inferred)**

- Designed the two-stage architecture: frozen DINOv2-base CLS embedding (768-d, L2-normalised) feeding a LogisticRegression head, avoiding both OCR and fine-tuning ("no fine-tuning code exists in the project").
- Implemented the shared `get_embedding` (train.py:19–26): EXIF transpose, RGB conversion, DINOv2 `AutoImageProcessor`, `last_hidden_state[:, 0]` CLS extraction, L2 normalisation at `dim=-1`, under `@torch.inference_mode()`, with automatic `cuda`/`cpu` selection (train.py:13).
- Implemented `train.train(data_dir, test_size, model_file)` (train.py:40–54): class list from `sorted(...)` sub-directories of `dataset/`, stratified split with `test_size=0.25` and `random_state=42`, `LogisticRegression(max_iter=2000)`, printed accuracy and per-class `classification_report`, joblib bundle with keys `clf`, `class_names`, `model_name`.
- Built the FastAPI service (app.py): `GET /` health/discovery (app.py:39–41), `POST /classify` multipart handler (app.py:44–54), `pdf_to_images` via PyMuPDF (app.py:30–36), `classify_pil` with `predict_proba` argmax and 4-dp rounding (app.py:21–24), fail-fast `FileNotFoundError` at import if `classifier.joblib` is absent (app.py:11).
- Localised the checkpoint to `Models/dinov2-base` (hub id `facebook/dinov2-base` commented out at train.py:11–12) so inference makes no network call.
- Curated the six-class dataset under `dataset/` (db, df, rb, rf, vb, vf — the class list is literally the sorted sub-directory names, train.py:31) and the test fixtures the document classifies live (`test_images/easy/db_1.png`, `vf_2.jpeg`, plus `one_page.pdf` and `two_page.pdf` built from them). Dataset size, per-class counts and provenance of the images are not stated.

**Supporting tasks (inferred)**

- Baseline evaluation in `openclip_test.ipynb`: ImageHash pHash baseline (cell ccd6451e) and a top-3-mean-cosine kNN over the same embeddings as a second opinion.
- PoC orchestration in `run_poc.ipynb`: executed training run with recorded throughput, Uvicorn launch on `0.0.0.0:8000` (cell 9e00bf09), example client `requests.post("http://127.0.0.1:8000/classify", files=...)`.
- Live API verification against the running service — "Every response body quoted below was captured by calling the live application; nothing is reconstructed from the source": 200 responses for `test_images/easy/db_1.png`, for `one_page.pdf` built from that same image, and for `two_page.pdf` (db_1.png + vf_2.jpeg); a captured 422 for a missing `file` field and a captured 500 for a text file sent as `notes.txt`. (The 405 row is documented as the Starlette default "Not declared by the project", not stated as captured; the zero-page PDF branch is stated as "not exercised".)
- Artefact inspection (`n_features_in_`, `coef_` shape, `model_name` in `classifier.joblib` and `doc_classifier.joblib`) and authoring of the documentation with a "Verified Implementation Constants" table.

**Estimated ownership level:** ML Engineer (sole end-to-end developer — inferred).

Rationale: the project comprises two source modules, two notebooks, one local checkpoint and two persisted bundles (`classifier.joblib`, which the service loads, and `doc_classifier.joblib`, whose status is not explained), with no other contributor mentioned, so end-to-end ownership of model selection, embedding pipeline, training, serving and evaluation is the reasonable inference. The work is ML-engineering in character — representation choice, linear probe, reproducible split, artefact provenance, train/serve parity via a shared function. It does not reach Senior AI Engineer or Solution Architect: a single process with two routes, no middleware, no error handling, no integrations beyond local disk, launched from a notebook named `run_poc.ipynb`. PoC-level production readiness caps the label at ML Engineer.

## 3. End-to-End Workflow

**Processing stages**

1. **Upload.** Client POSTs `multipart/form-data` with one required `file` field (`UploadFile = File(...)`) to `POST /classify`; FastAPI returns 422 if the field is missing.
2. **Format branch.** If `filename` ends in `.pdf` (case-insensitive), `pdf_to_images` opens the bytes with PyMuPDF, calls `page.get_pixmap()` with library defaults per page and converts via `Image.frombytes("RGB", ...)`; otherwise the bytes go to `Image.open` (app.py:50).
3. **Per-image normalisation** (inside `get_embedding`): EXIF transpose, RGB conversion, then the `BitImageProcessor` — resize shortest edge 256, centre-crop 224×224, rescale 1/255, normalise with mean [0.485, 0.456, 0.406] / std [0.229, 0.224, 0.225].
4. **Embedding.** Frozen DINOv2-base (geometry in Section 4), loaded once at import in `.eval()` mode, runs under `@torch.inference_mode()`; CLS token `last_hidden_state[:, 0]` is L2-normalised to 768-d.
5. **Classification.** `classify_pil` calls `predict_proba` on the `LogisticRegression` (`C=1.0`, `solver='lbfgs'`, `max_iter=2000`, `coef_` (6, 768)); `prediction` is the argmax class, `confidence` is `round(float(probs.max()), 4)`.
6. **Response shaping** (app.py:51–54). One image (any single image, or a single-page PDF) returns flat `{file, prediction, confidence}`. More than one returns `{file, results: [{page, prediction, confidence}]}` with 1-based pages. Zero images (a PDF rendering no pages) yields `{"file": ..., "results": []}` — noted as unexercised.

**Training workflow** (train.py:40–54): walk `dataset/<class>/*`, embed with the same `get_embedding`, stratified split (`test_size=0.25`, `random_state=42`), fit, print accuracy and `classification_report`, `joblib.dump` the bundle.

**Data flow**

- Inbound: multipart bytes (image or PDF) -> PIL `Image` objects, one per page.
- Internal: PIL image -> processor tensor (224×224) -> DINOv2 `last_hidden_state` -> 768-d unit vector -> `predict_proba` (1×6) -> label + float.
- Outbound: JSON, flat or with `results[]`; `file` echoes `UploadFile.filename` unsanitised.
- On disk: `Models/dinov2-base/` (config.json, preprocessor_config.json, `model.safetensors` 346,345,912 bytes) and `classifier.joblib` (19,442 bytes).

**Integrations and dependencies:** no database, queue, cache or external service at request time; "Everything it needs — the transformer weights and the trained head — is loaded from local disk when the module is imported, and each request is handled synchronously in that same process." Libraries: FastAPI/Starlette, Uvicorn, python-multipart, PyTorch, Transformers (`AutoImageProcessor`, `AutoModel`), scikit-learn, joblib, Pillow, PyMuPDF, NumPy. Only the optional `/docs` Swagger UI page reaches a CDN.

**Cross-cutting concerns as documented:** "No middleware, dependency, response model or exception handler is registered" on the FastAPI app (app.py:16), and no authentication, authorisation, rate limiting, TLS, logging or monitoring appears anywhere in the document; the only documented launch binds `0.0.0.0:8000` from run_poc.ipynb. Treating that combination as an exposure risk is an analyst inference, not a documented finding.

**Flow line**

`Client (multipart file) -> FastAPI POST /classify -> .pdf? PyMuPDF get_pixmap per page : Pillow Image.open -> EXIF transpose + RGB -> AutoImageProcessor (256 -> 224 crop, ImageNet norm) -> DINOv2-base CLS 768-d L2 -> LogisticRegression predict_proba -> argmax + round(4) -> JSON {file, prediction, confidence} | {file, results[]}`

**Architecture Summary**

Motor Doc Check AI is a deliberately minimal single-process inference service. A 346 MB frozen ViT and a 19 KB linear head are loaded from local disk at import; each request is handled synchronously in that process with no network, storage or messaging dependency. The learned component is a 6×768 coefficient matrix, so retraining is one function call and the shipped artefact is small enough to diff. Because `get_embedding` is shared by train.py and app.py, training-time and serving-time preprocessing are identical by construction. The design trades robustness (error handling, input limits, rejection thresholding, concurrency) for simplicity, consistent with its PoC status.

## 4. Technologies and Tools Used

| Category | Technology (as stated in the document) |
|---|---|
| Programming languages | Python (CPython) 3.11.15 |
| Frameworks | FastAPI 0.141.1 (on Starlette); PyTorch 2.11.0; Hugging Face Transformers 5.5.0 (`AutoImageProcessor`, `AutoModel`; the checkpoint's config.json was written by transformers_version 5.14.1) |
| Libraries | scikit-learn 1.9.0 (`LogisticRegression`, `train_test_split`, `classification_report`); joblib 1.5.3; Pillow 12.3.0; PyMuPDF 1.28.0 (bundles MuPDF 1.29.0); NumPy 1.26.4; python-multipart 0.0.32; ImageHash 4.3.2 (experimental pHash baseline); `requests` (example client, version not stated) |
| AI/ML models | DINOv2-base — local checkpoint `Models/dinov2-base` (model_type dinov2, 12 layers, 12 heads, hidden 768, patch 14, image_size 518, layer_norm_eps 1e-06; `model.safetensors` 346,345,912 bytes; hub origin `facebook/dinov2-base`, commented out at train.py:11–12; loaded in `.eval()` mode, frozen — "no fine-tuning code exists in the project"); scikit-learn LogisticRegression head (`C=1.0`, `solver='lbfgs'`, `max_iter=2000`, `n_features_in_=768`, `coef_` (6, 768)) |
| OCR tools | None — the design replaces the OCR-based path; no OCR library is used |
| Databases | None ("no database, no queue, no cache") |
| Cloud platforms | Not stated in documentation (offline, local-disk service) |
| APIs | Own REST API: `GET /` and `POST /classify` (multipart/form-data); auto-generated OpenAPI body schema `Body_classify_classify_post` (`"required": ["file"]`) and Swagger UI at `/docs` (the only component that reaches the internet). No external API is consumed. No authentication/authorisation scheme is stated in documentation |
| DevOps tools | Not stated in documentation |
| Version control | Not stated in documentation (the document only notes `classifier.joblib` is the artefact to be "shipped and version-controlled") |
| Deployment tools | Uvicorn 0.52.0 on `0.0.0.0:8000`, launched from run_poc.ipynb cell 9e00bf09 ("notebook launch only") |
| Document processing tools | PyMuPDF 1.28.0 (`page.get_pixmap()`, `Image.frombytes("RGB", ...)`); Pillow 12.3.0 (decode, EXIF transpose, RGB conversion) |
| Automation tools | Not stated in documentation (retraining is a one-call function; no scheduler, CI or pipeline tool described) |

## 5. Technical Skills Demonstrated

- **Software Engineering:** separation of concerns (train.py = embedding + training, app.py = HTTP + request logic); shared `get_embedding` for train/serve parity; fail-fast import-time check; module-level constants (`MODEL_FILE`, `REJECT_THRESHOLD`, `DINO_NAME`); source-cited documentation.
- **AI Engineering:** selection of a self-supervised ViT (DINOv2-base) as a frozen feature extractor; checkpoint localisation for offline inference; `model_name` provenance in the bundle; cosine-similarity kNN second opinion over L2-normalised embeddings.
- **Machine Learning:** linear-probe classification (`LogisticRegression`, lbfgs, `max_iter=2000`) on frozen embeddings; stratified 75/25 hold-out with `random_state=42`; accuracy and per-class `classification_report`; inspection of the fitted estimator; pHash baseline comparison.
- **NLP:** Not demonstrated in this project (image-only pipeline).
- **Computer Vision:** ViT CLS-token embeddings; `BitImageProcessor` preprocessing (256 resize, 224 centre-crop, ImageNet normalisation); EXIF orientation handling; RGB conversion; PDF rasterisation; perceptual hashing baseline.
- **Data Engineering:** directory-as-label dataset layout with class list from `sorted(...)`; batch embedding pass with recorded throughput; multi-page PDF splitting into per-page records.
- **Cloud:** Not demonstrated in this project.
- **MLOps:** versionable 19,442-byte bundle (`clf`, `class_names`, `model_name`); provenance traceability across two bundles (`classifier.joblib` -> 'Models/dinov2-base', `doc_classifier.joblib` -> 'facebook/dinov2-base'); reproducible split; automatic device selection; import-time model validation. No CI, monitoring or registry described.
- **API Development:** FastAPI `UploadFile = File(...)` multipart handling; conditional response shaping (flat vs `results[]`); 4-dp rounding; 1-based page numbering; auto-generated schema `Body_classify_classify_post`; captured 200/422/500/405 behaviours.
- **Prompt Engineering:** Not demonstrated in this project.
- **System Design:** single-process, dependency-free inference topology; frozen-backbone / tiny-head split isolating the retrainable surface; extensibility by directory; explicit robustness-for-simplicity trade-off at PoC stage.

## 6. Detailed Technical Contributions

**Features implemented**

- `POST /classify` (app.py:44–54): one multipart `file`; returns `{"file", "prediction", "confidence"}` for a single image or single-page PDF, `{"file", "results": [{"page", "prediction", "confidence"}]}` for multi-page PDFs.
- `GET /` (app.py:39–41): "Health check and service discovery", handler `health()`. The document does not quote its response body; that it surfaces the class list is inferred from "The classifier recognises six classes … read back from the persisted bundle and confirmed by calling GET /". Note the document's own endpoint summary table lists only `POST /classify` (numbered "S.No 2"), so `GET /` appears only in the request/response section — a documentation gap, not a code gap.
- PDF handling: case-insensitive `.pdf` suffix check; `pdf_to_images` renders every page with `page.get_pixmap()` and `Image.frombytes("RGB", ...)`.
- CPU/GPU transparency: `"cuda" if torch.cuda.is_available() else "cpu"`.
- Directory-driven retraining via `train.train(data_dir, test_size, model_file)`; fail-fast `FileNotFoundError` at import when `classifier.joblib` is missing.

**Models used**

- DINOv2-base, frozen (`.eval()`, `@torch.inference_mode()`), local path `Models/dinov2-base`; CLS token, L2-normalised, 768-d.
- scikit-learn `LogisticRegression(max_iter=2000)`; persisted estimator: `C=1.0`, `solver='lbfgs'`, `n_features_in_=768`, `coef_` (6, 768). Classes `['db', 'df', 'rb', 'rf', 'vb', 'vf']`; the document separately names the six classes as driving licence back, driving licence front, resident back, resident front, vehicle licence back, vehicle licence front — the code-to-label mapping is inferred from that ordered list matching the sorted codes, and the document notes the codes "are nothing more than the sub-directory names under `dataset/`".
- Experimental: ImageHash pHash baseline and top-3-mean-cosine kNN (openclip_test.ipynb).

**Pipelines built**

- Inference: bytes -> (PyMuPDF | Pillow) -> EXIF transpose -> RGB -> `AutoImageProcessor` -> DINOv2 CLS -> L2 -> `predict_proba` -> argmax/round(4) -> JSON.
- Training: `dataset/<class>/*` -> `get_embedding` -> stratified split -> fit -> accuracy + report -> `joblib.dump` to `classifier.joblib`.

**APIs integrated**

- None external. The only documented client is `requests.post("http://127.0.0.1:8000/classify", files=...)` in run_poc.ipynb Step 3.

**Data extraction methods**

- Visual only: DINOv2 CLS embeddings. No text, OCR or metadata extraction; PDF content is obtained solely by rasterising pages with PyMuPDF defaults.

**Validation logic**

- Request-level: FastAPI schema validation of the required `file` field (422 with `{"type": "missing", "loc": ["body", "file"], "msg": "Field required"}`).
- Model-level: `REJECT_THRESHOLD = 0.0` (app.py:9) is declared; at that value no prediction is rejected (inferred from the value — the document does not describe rejection behaviour).
- Hold-out evaluation during training: stratified 25% split (`test_size` default 0.25, `stratify=y`, `random_state=42`), accuracy and per-class `classification_report` printed (values not recorded).
- Format detection: PDF branch is chosen purely on a case-insensitive `.pdf` filename suffix — no magic-byte or MIME sniffing is declared.
- Fail-fast startup validation: `app.py:11` raises `FileNotFoundError` at import time if `classifier.joblib` is absent, so a mis-deployed artefact fails at load rather than per request.
- Not declared: content-type restriction, file-size limit, page-count limit, extension allow-list ("No content-type restriction, file-size limit, page-count limit or extension allow-list is declared").
- Not declared (security): no authentication, authorisation, rate limiting or TLS; "No middleware, dependency, response model or exception handler is registered." `file` echoes `UploadFile.filename` "exactly as the client supplied it, unsanitised". The only documented bind is `0.0.0.0:8000` from run_poc.ipynb cell 9e00bf09.
- Error behaviour is entirely framework default: 422 from FastAPI request validation, 405 `{"detail": "Method Not Allowed"}` from Starlette, and 500 for anything raised inside the handler ("no error handling is declared anywhere in app.py, so any exception from PyMuPDF, Pillow, PyTorch or scikit-learn surfaces as an unhandled 500").

**Automation workflows**

- Retraining is a single `train.train(...)` call; adding a class is adding a folder under `dataset/` and retraining. No scheduled or CI-driven automation described.

**Optimization techniques**

- Frozen backbone with a linear head: everything learned is a 6×768 matrix, so retraining is one embedding pass plus an lbfgs fit and the shipped artefact is 19,442 bytes.
- `@torch.inference_mode()` plus one-time import-time loading avoid autograd and per-request weight loading; L2 normalisation makes dot products equal cosine similarity, so the kNN second opinion reuses embeddings without re-embedding.

**Performance improvements**

- Against the OCR path recorded at "around 7 – 10 seconds", the stated aim is classification in about a second. The only measured figure is CPU embedding throughput of 1.38–2.35 images per second per class folder during the training pass; the document states "No per-request latency figure is recorded anywhere in the project."

## 7. Challenges and Solutions

1. **OCR-bound latency for document typing (business + technical).** The existing route took "around 7 – 10 seconds depends on the ocr extraction". Solution: bypass OCR and classify from a visual embedding with a linear head. Alternatives: OCR keyword rules (the incumbent); end-to-end CNN/ViT fine-tuning (analyst suggestion); zero-shot CLIP/OpenCLIP prompting (analyst suggestion — the notebook is named openclip_test.ipynb, but the document records only a pHash baseline and a kNN in it); pHash template matching (documented experimental baseline).

2. **Keeping the retrainable surface small (technical).** Solution: freeze DINOv2-base under `@torch.inference_mode()` and learn only a `LogisticRegression` head — a 19 KB artefact "small enough to review and diff". Alternatives: full or parameter-efficient fine-tuning such as LoRA (analyst suggestion), raising the accuracy ceiling at the cost of training infrastructure and a large artefact.

3. **Offline, no-GPU operation (business constraint).** Solution: `DINO_NAME` points at local `Models/dinov2-base` (hub id commented out); automatic CPU fallback. Only `/docs` reaches a CDN. Alternative: hub download with a local cache (analyst suggestion), implicitly rejected for on-prem deployment.

4. **One endpoint for images and multi-page PDFs (technical).** Solution: filename-suffix branch to PyMuPDF, per-page classification, response shape driven by image count. Documented limitations: detection by filename only, `get_pixmap()` at library defaults (resolution uncontrolled), zero-page PDF yields `{"results": []}` via an untested branch. Alternative: magic-byte sniffing and content-type validation (analyst suggestion).

5. **Train/serve preprocessing parity (technical).** Solution: one `get_embedding` used by both training and app.py, so EXIF/RGB/processor steps cannot drift. Alternative: duplicated preprocessing in the service (analyst suggestion) — the failure mode this avoids.

6. **Model provenance (MLOps).** Solution: store `model_name` with `clf` and `class_names`; the document shows `classifier.joblib` -> 'Models/dinov2-base' versus `doc_classifier.joblib` -> 'facebook/dinov2-base'. Alternative: a model registry (analyst suggestion).

7. **Robustness gaps (documented limitations).** No `try/except` in app.py: an undecodable upload raises `PIL.UnidentifiedImageError` at app.py:50 and surfaces as a bare 500 (captured live); any PyMuPDF/Pillow/PyTorch/scikit-learn failure also returns 500. No size, page-count, content-type or extension limits. `file` echoes the client filename unsanitised. `REJECT_THRESHOLD = 0.0` means low-confidence predictions (e.g. the captured 0.7935 for `vf`) return without an "unknown" path. Requests are synchronous in a single process. Transformers 5.5.0 loads a checkpoint written by 5.14.1. Fixes (analyst suggestions): exception handlers returning 400/415, upload limits, a reject threshold calibrated on the hold-out set, Uvicorn workers or a queue for concurrency, pinning Transformers to the checkpoint's version.

8. **Security and operability are undocumented (gap, not a solved challenge).** The document registers "no middleware, dependency, response model or exception handler" and describes no authentication, authorisation, rate limiting, TLS, logging, monitoring, container or CI; the only documented launch binds `0.0.0.0:8000` from a notebook cell. Nothing in the source addresses this — for an on-prem insurance intake service handling licence and residency images, an auth layer, request logging and a non-notebook deployment path would be the first hardening steps (analyst suggestion).

## 8. Impact Analysis

- **Business impact:** An OCR-free document-type classifier for UAE motor claim intake covering six document faces, aimed at ~1-second classification versus the recorded "7 – 10 seconds" OCR path. No measured per-request latency, accuracy or volume figures are recorded.
- **Productivity improvements:** Adding a document type is "a directory operation" plus one retrain call; retraining "is one function call and one small artefact".
- **Accuracy improvements:** Not quantified. Training prints accuracy and a per-class report, but the document does not record the values. Captured confidences: 0.9458 for `db_1.png` and 0.9458 again for `one_page.pdf` built from that same image — evidence that the PyMuPDF rasterisation path reproduced the direct-image result exactly on that sample (a de facto parity check) — and 0.7935 for the `vf` page of `two_page.pdf`, i.e. a ~0.15 spread between an easy and a harder page with no rejection path in place.
- **Cost savings:** Not quantified. Qualitatively: no GPU, no cloud or external API calls, a 19,442-byte artefact while the 346 MB transformer never changes.
- **Time savings:** Documented OCR baseline of 7–10 seconds; the only recorded throughput is 1.38–2.35 images/second (CPU, training pass). Per-request latency not measured.
- **User benefits:** One multipart endpoint handling images and multi-page PDFs uniformly; fully offline; JSON with per-page results and confidences.

## 9. Interview Discussion Points

- Why frozen DINOv2 + logistic regression rather than fine-tuning: six classes, cheap retraining (one embedding pass plus an lbfgs fit) and a diffable 19,442-byte artefact against a 346 MB backbone that never changes; the documented position is that "no fine-tuning code exists in the project" and everything learned lives in a 6×768 coefficient matrix. (Dataset size and class balance are *not* stated in the document — do not claim "small dataset" as fact; the honest line is that a linear probe is the low-risk default when the labelled set is directory-curated and the backbone is self-supervised — inferred.)
- Why the CLS token with L2 normalisation: a fixed 768-d representation whose dot product is cosine similarity, enabling the kNN second opinion without re-embedding.
- How train/serve skew is prevented: one `get_embedding` shared by train.py and app.py, including EXIF transpose and RGB conversion before the processor.
- What `REJECT_THRESHOLD = 0.0` means in practice and how to calibrate a real threshold from the stratified hold-out set (the 0.7935 `vf` example).
- Why one image returns a flat object but multiple pages return `results[]`, and why the zero-page branch returning `[]` is a latent inconsistency.
- Error handling: own that an undecodable upload returns a bare 500 and no size/page/content-type limits exist; describe the fix.
- Metrics honesty: accuracy is printed but not recorded; the only measured number is 1.38–2.35 images/s during the CPU embedding pass; "one second" is a target, not a measurement.
- Offline/on-prem: how localising `Models/dinov2-base` and recording `model_name` support governance, including the `classifier.joblib` vs `doc_classifier.joblib` provenance difference.
- Version skew: Transformers 5.5.0 runtime loading a checkpoint saved by 5.14.1 — how you would pin and verify.
- Scaling path (analyst suggestions on top of the documented single-process design): Uvicorn workers, batching page embeddings per PDF, controlling `get_pixmap` DPI instead of library defaults, GPU via the existing `"cuda" if torch.cuda.is_available()` switch.
- Extensibility: "new class = new folder + retrain" works because classes are sorted directory names — and the risk that renaming a folder reorders classes.
- Verification discipline: every response body in the API reference "was captured by calling the live application; nothing is reconstructed from the source", and the constants table cites the source line or reads the value from the artefact itself (`n_features_in_`, `coef_` shape, `model_name`). Be ready to explain why documenting an unexercised branch (zero-page PDF -> `{"results": []}`) as unexercised matters more than claiming coverage.
- A useful captured detail: `db_1.png` and the single-page PDF built from it both returned 0.9458, showing the PyMuPDF path did not perturb the embedding on that sample — the seed of a regression test comparing image-vs-rasterised-PDF confidences.
- Security posture: no authentication, authorisation, rate limiting, TLS or logging is described; "no middleware, dependency, response model or exception handler is registered"; `file` echoes the client filename unsanitised; and the only documented bind is `0.0.0.0:8000` from a notebook. For licence and residency images this is the first thing to harden before any production claim (analyst position).
- Provenance limits: `model_name` records a *string* — `'Models/dinov2-base'` in `classifier.joblib` versus `'facebook/dinov2-base'` in `doc_classifier.joblib` — so it identifies the intended checkpoint but not its bytes; a hash or a registry entry would (analyst suggestion).
- Documentation gaps worth owning: the endpoint summary table lists only `POST /classify` (starting at "S.No 2") so `GET /` is missing from it, and the status/purpose of the second bundle `doc_classifier.joblib` is never explained.

## 10. Architecture Explanation Points

Draw the document's box diagram: Client -> FastAPI gateway -> Preprocessing (PDF -> RGB pages) -> DINOv2-base (768-d embedding) -> LogReg head (label + confidence).

- **Components (30 s):** One process. app.py owns `GET /` for health and `POST /classify` for inference; train.py owns `get_embedding` and `train`. On disk: `Models/dinov2-base` (346 MB, frozen) and `classifier.joblib` (19 KB, the only thing that changes).
- **Data flow (45 s):** Multipart bytes arrive. `.pdf` filenames go to PyMuPDF for per-page rasterisation; anything else to Pillow. Each image is EXIF-transposed, converted to RGB, resized to 256, centre-cropped to 224 with ImageNet normalisation, run through DINOv2 under inference mode, and the CLS token is L2-normalised to 768-d. Logistic regression yields six probabilities; argmax is the label, max rounded to four places is the confidence. One image returns a flat object; several return `results[]` with 1-based pages.
- **Key design decisions (30 s):** Freeze the expensive model and learn a 6×768 matrix so retraining is one call and the artefact is diffable (19,442 bytes vs 346,345,912 bytes). Derive classes from sorted directory names so a new document type is a folder plus a retrain, no code change. Share `get_embedding` between training and serving so preprocessing cannot drift. Load both artefacts once at import and fail fast (`FileNotFoundError`) if the head is missing. Keep the checkpoint local (`Models/dinov2-base`, hub id commented out) so nothing is downloaded at run time. Normalise embeddings to unit length so the same vectors serve a cosine kNN second opinion without re-embedding. Pick device automatically so the same code runs on CPU or GPU.
- **Trade-offs (15 s):** Simplicity over robustness — no error handling, no input limits, no auth or middleware of any kind, no effective reject threshold (`REJECT_THRESHOLD = 0.0`), synchronous single-process serving, PDF detection by filename suffix, `get_pixmap()` at library defaults so page resolution is uncontrolled. Linear probe over fine-tuning — cheaper, smaller and reviewable, but it caps accuracy on hard cases and the document records no accuracy figure either way. Notebook launch over a packaged deployment — fine for a PoC, not for the on-prem service it is aimed at.
- **What I would improve next (15 s, analyst suggestions):** Exception handlers returning 400/415 and upload/page-count limits; a calibrated non-zero `REJECT_THRESHOLD` with an "unknown" outcome (the captured 0.7935 page is the motivating case); magic-byte format detection instead of the filename suffix; batched page embeddings and an explicit `get_pixmap` DPI; recorded hold-out accuracy and per-request latency so the "about a second" aim becomes a measurement; pinned Transformers version matching the checkpoint's 5.14.1; an auth layer and request logging before exposure; multiple Uvicorn workers or a queue; and resolving the `classifier.joblib` / `doc_classifier.joblib` ambiguity with a hash-based provenance record.
