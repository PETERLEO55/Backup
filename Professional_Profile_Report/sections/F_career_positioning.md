## F. Career Positioning

All judgements below are derived solely from the 15 documented Qatar Insurance Group projects (2026). The documents never state the candidate's job title, team size, reporting line, or years of experience; role and level are inferred from documented scope, ownership, and technical decisions.

### Most Suitable Job Titles

**1. Document AI / Intelligent Document Processing (IDP) Engineer — strongest fit**

Eleven of the fifteen projects are document-intelligence systems: *DOHA OCR AI*, *UAE OCR AI*, *UAE OCR API — Query-Based Extraction*, *SLM AI OCR*, *Multi-doc All-in-one OCR*, *Document Detection API (TextMatch Classification)*, *TextMatch Label Trainer*, *Trade License Extractor*, *Document Classification Model (Deep Learning)*, *VL_version_OCR*, and *KYC Document Validation System*. The portfolio covers essentially the full IDP surface: segmentation (YOLOv8 croppers in *SLM AI OCR*, *Multi-doc All-in-one OCR*, *Document Detection API*, *VL_version_OCR*), OCR engine selection and abstraction (Azure Document Intelligence, RapidOCR, Tesseract, Google Cloud Vision, PyMuPDF text-layer; PaddleOCR appears only in the superseded intermediate pipeline of *Trade License Extractor*), classification (fuzzy keyword n-grams, DINOv2 linear probe, anchor validation), field extraction (custom DI models, regex/label anchors, VLM strict-schema decoding), validation (expiry, image quality, anchor hit-rate gates), and bilingual Arabic/English normalisation. Meets: hands-on OCR/VLM pipelines, multi-vendor integration, production API delivery, RTL/bilingual handling. Typically missing: documented field-level accuracy benchmarks — most documents explicitly state accuracy figures are not recorded.

**2. Senior AI Engineer (applied) — strong fit**

Eight of the fifteen documents independently infer a Senior AI Engineer level (*DOHA OCR AI*, *SLM AI OCR*, *Multi-doc All-in-one OCR*, *P&C Clause Recommendation Engine*, *Trade License Extractor*, *UAE OCR AI*, *VL_version_OCR*, and *LLM Fine-tuning Platform* as "Senior AI / Machine Learning Engineer") — all explicitly inferred, since no document names a title. Evidence of senior behaviour: architecture-level trade-offs (dual cloud/local path in *DOHA OCR AI* for cost, quota and regional deployment flexibility; hybrid cloud-OCR / on-prem-LLM in *SLM AI OCR* for PII sovereignty), evidence-driven rejection of approaches (vision-LLM dropped in *Trade License Extractor* after documented DCCI/Fax field confusions; Azure Document Intelligence dropped in *VL_version_OCR*), and forensic root-cause work (Run 1 catastrophic memorization analysis in *LLM Fine-tuning Platform*). Misses: no documented evidence of leading engineers, owning cross-system enterprise architecture, or running CI/CD and production monitoring.

**3. GenAI / LLM Engineer — strong fit, narrower depth**

*LLM Fine-tuning Platform* is a complete VLM lifecycle: teacher distillation from Qwen3.5-122B on vLLM with Pydantic-guided decoding, a Streamlit HITL annotation tool with reviewer provenance, dataset versioning on Hugging Face Hub, 4-bit QLoRA fine-tuning of Gemma 4 E2B, GGUF Q4_K_S export and on-prem llama.cpp serving. Supporting evidence: *VL_version_OCR* (multi-provider VLM engine, six execution modes, strict-schema decoding), *SLM AI OCR* (self-hosted Qwen via Ollama, temperature 0, strict json_schema, /no_think), *Trade License Extractor* (guard-railed local LLM fallback), *P&C Clause Recommendation Engine* (Gemini structured output), *Multilanguage Speech-to-Text* (multimodal Gemini audio→JSON). Meets: fine-tuning, quantisation, structured output, self-hosted serving, RAG components (hybrid semantic/BM25 + RRF in *Trade License Extractor*). Misses: no agent frameworks, no eval harness for LLM quality beyond `values_match()`, no LLM observability/cost tracking, no vector database in production.

**4. Applied ML Engineer / ML Engineer — good fit**

*Document Classification Model (Deep Learning)* is textbook applied ML: frozen DINOv2-base CLS embeddings, LogisticRegression linear probe, stratified 75/25 split with fixed seed, shared `get_embedding` for train/serve parity, versionable 19,442-byte artefact. *LLM Fine-tuning Platform* adds LoRA hyperparameter design and failure diagnosis. *DOHA OCR AI* adds Azure DI custom model training plus a synthetic training-data pipeline (300+ balanced samples) that fixed a diagnosed training-distribution gap. Misses: no deep-learning training from scratch, limited hyperparameter search, and — the recurring theme — almost no recorded evaluation metrics.

**5. Computer Vision Engineer — moderate fit**

Real CV work exists: YOLOv8 detection and cropping across three services, PDF rasterisation strategy (150 DPI, 2x scale, 300 DPI for OCR), OpenCV quality metrics (Laplacian-variance blur, brightness, contrast, resolution gates) in *Multi-doc All-in-one OCR* and *Document Detection API*, perceptual hashing plus ELA and spectral forensics in *Claim Duplication Detector API*, ArcFace face verification in *KYC Document Validation System*, and ViT embeddings in *Document Classification Model*. Misses: the documents never describe training the YOLO weights (dataset, labelling, evaluation are absent), and there is no video, tracking, 3D, or GPU-optimisation work.

**6. ML / AI Solutions Engineer (insurance domain) — good fit**

Every project is anchored in a named insurance workflow: FNOL and motor claims intake, KYC onboarding, medical claims pre-screening, P&C underwriting clause gap analysis, subrogation via insurer-code resolution. The candidate consistently designs contracts for downstream consumers (uniform `messageType 'S'/'E'` envelopes, role projections `finance_view()` / `gi_view()`, an .env routing table of 14 extractor mappings). Meets: domain fluency, integration-contract design, stakeholder-facing tooling (Streamlit review/admin UIs in *TextMatch Label Trainer*, *LLM Fine-tuning Platform*, *KYC Document Validation System*, *Claim Duplication Detector API* and *Multilanguage Speech-to-Text*). Misses: no documented data-governance, model-risk or regulatory-approval process ownership.

**7. Backend / API Engineer (AI services) — solid secondary fit**

FastAPI appears in twelve of the fifteen projects with genuine engineering: APIRouter organisation and `root_path` for reverse-proxy mounting, custom OpenAPI overrides for binary array uploads, structured error contracts mapped to 400/404/422/500/502, `ThreadPoolExecutor` concurrency, thread-local ONNX sessions to avoid mutex contention, and sync-`def` handlers deliberately dispatched to Starlette's AnyIO worker pool. Misses: authentication, rate limiting, and automated tests are absent almost everywhere the documents look.

**8. Prompt / Structured-Output Engineer — narrow but genuine**

Consistent, disciplined constrained decoding: `format='json'` with explicit schemas (Ollama), `strict=True` / `additionalProperties=False` / temperature 0 (*VL_version_OCR*), Pydantic `response_format` guided decoding (*LLM Fine-tuning Platform*), forced JSON MIME type (*Multilanguage Speech-to-Text*), and schema-constrained Gemini output (*P&C Clause Recommendation Engine*). Too narrow to be a target title on its own, but a strong differentiator inside the titles above.

### Current Experience Level

**Years of experience are not stated in any of the 15 documents, and cannot be inferred from them.** All fifteen projects are dated 2026 and all are for a single client (Qatar Insurance Group), so the corpus is a snapshot of output, not a career timeline.

**Assessed level from scope, complexity and ownership: mid-level to senior AI Engineer, with the weight of evidence sitting at the upper end of that range for document-AI work specifically.**

Justification for senior-leaning:
- **Breadth of end-to-end ownership.** Across the corpus the candidate owns architecture, model integration or training, backend, data pipeline, UI where needed, deployment configuration, and the technical documentation. Fourteen of the fifteen inferred ownership assessments land at "AI Engineer" or "Senior AI Engineer" (the exception is *Document Classification Model (Deep Learning)*, assessed as ML Engineer).
- **Architecture-level trade-offs with articulated rationale.** Dual-path cloud/local routing for cost, quota and regional deployment flexibility (*DOHA OCR AI*); hybrid Azure-OCR + self-hosted Qwen for PII sovereignty and token cost (*SLM AI OCR*); deterministic-first extraction with a guard-railed optional LLM fallback (*Trade License Extractor*); triage-only service decoupled from extraction so it scales independently (*Document Detection API*).
- **Evidence-driven reversal of decisions.** The vision-LLM path was rejected in *Trade License Extractor* only after documenting concrete DCCI-vs-Register-No and Fax-vs-Mobile confusions; Azure Document Intelligence was abandoned in *VL_version_OCR* after it failed on multi-document scans; the synthetic generator in *DOHA OCR AI* was rewritten v1/v2 → v3 after Arabic encoding failures; the Run 1 fine-tune failure in *LLM Fine-tuning Platform* produced a written forensic report and a redesigned recipe.
- **Production concerns appear unprompted**: health endpoints, rotating audit logs, uniform error envelopes, thread-local engines, per-crop error isolation, non-root hardened Docker image (*UAE OCR AI*), Vault-backed credentials (*SLM AI OCR*).

Justification for the ceiling (not yet Lead / Solution Architect / Staff):
- **No documented evidence of leading other engineers.** Every ownership rationale in the corpus notes the absence of team, roster, or role declarations. The only named collaborators anywhere are the annotation reviewers in *LLM Fine-tuning Platform* — the candidate ("peter") alongside "Afsal" and "waris" — and no reporting relationship is stated.
- **Production-engineering maturity is uneven.** Recurring absences: automated tests (explicitly "not implemented" in *VL_version_OCR*), CI/CD anywhere, unpinned dependencies in several projects, no monitoring beyond `/health`, and no authentication on services ingesting passports and civil IDs.
- **Measurement discipline is the weakest dimension.** Most projects state outright that accuracy, throughput, ROI and cost figures are not recorded; several thresholds (65%/90% classification, RapidFuzz 80/85, quality gates) were shipped without a documented calibration set. That is the single clearest gap between this portfolio and a staff-level one.

### Strongest Expertise Areas

**1. Document AI / OCR pipeline architecture (deepest and most repeated)**
Eleven document-processing systems spanning cloud custom models, local text-layer parsing, VLM extraction, and classical CV. Recurring signature: choose the cheapest sufficient path first, escalate only when it fails — PyMuPDF text-layer before Azure DI (*DOHA OCR AI*, *UAE OCR AI*); character-count and Arabic-sanity gates before OCR (*Trade License Extractor*); blank-page gate before perceptual hashing (*Claim Duplication Detector API*); local Tesseract as a *router* that picks the Azure model or filters pages before any billed call (*UAE OCR AI*).

**2. Multi-vendor AI service integration and vendor-neutral abstraction**
Azure Document Intelligence (custom, prebuilt-read, prebuilt-layout with QUERY_FIELDS), Azure Translator, Google Cloud Vision, Google Gemini, Ollama, vLLM, llama.cpp, Hugging Face Hub, DeepFace, RapidOCR, Tesseract, PaddleOCR. Abstractions are deliberate: a runtime `ocr_type` toggle in *Multi-doc All-in-one OCR* and *Document Detection API*; a six-mode `model_name` enum across three VLM providers and two SDK paradigms in *VL_version_OCR*; `azure_client.py` isolated in *UAE OCR API — Query-Based Extraction* explicitly to allow future local-LLM substitution.

**3. Production FastAPI service design with hard contracts**
Uniform response envelopes (`message` / `messageType` / `response`), typed Pydantic v2 schemas, per-item error isolation so one bad crop never fails a batch, explicit 400/404/415/422/500/502 semantics, custom OpenAPI overrides, `root_path` reverse-proxy mounting, `ThreadPoolExecutor` and thread-local ONNX sessions, and sync handlers deliberately offloaded to the AnyIO pool. Present in *DOHA OCR AI*, *UAE OCR AI*, *Multi-doc All-in-one OCR*, *Document Detection API*, *SLM AI OCR*, *VL_version_OCR*, *UAE OCR API*.

**4. Arabic / bilingual NLP for regulated GCC documents**
Genuinely specialist and hard to hire for: RTL bidirectional wrap resolution in PDF text layers via `transfer_window()` / `fix_number_wrap()` / `is_label()` (*DOHA OCR AI*); handling logical vs visual vs presentation forms with arabic-reshaper and python-bidi while preserving unshaped strings for translation and catalogue lookup (*UAE OCR AI*, *VL_version_OCR*); right-to-left polygon sorting to rebuild vehicle strings; dual-language fuzzy insurer resolution with RapidFuzz token-sort/token-set (threshold 85) against 42 records to emit canonical subrogation codes; Arabic-sanity lexicon used as an OCR routing gate (*Trade License Extractor*).

**5. Applied GenAI: constrained decoding, fine-tuning and on-prem serving**
The full loop in *LLM Fine-tuning Platform* — 122B teacher distillation → 664 UAE / 296 Doha human-audited records → versioned 564/100 split on the Hub → 4-bit QLoRA on Gemma 4 E2B → GGUF Q4_K_S → llama.cpp on-prem, verified live via `GET /v1/models` — plus disciplined structured output everywhere else. Notable judgement: the forensic diagnosis that training vision layers while serving a stock `mmproj` contributed to catastrophic memorization, and the resulting recipe redesign.

**6. Cost-, sovereignty- and constraint-aware engineering**
A consistent thread rather than a one-off: local PyMuPDF parsing at ~10 ms and zero cloud cost for MOI reports; page pre-filtering (>2 of 7 keywords) before billed Azure calls; local RapidOCR as the default engine to minimise Azure transactions; an LRU cache (maxsize=1024) to cut translator round-trips; on-prem Qwen and GGUF serving driven by UAE PDPL / Qatar data-residency requirements; template-only Azure build mode worked around with a synthetic-data pipeline generated inside the tenant boundary; a CPU-only, no-paid-API constraint honoured end-to-end in *Trade License Extractor*.

### Skills Gaps

Each gap below states what evidence does exist, so nothing is asserted as a blanket absence.

**1. Databases and persistence — the largest structural gap.**
Across all 15 projects, no relational or NoSQL database appears in production. Persistence is: local filesystem (`uploads/`, `cropped_images/`), append-only `output.json` audit files (*Multi-doc All-in-one OCR*, *Document Detection API*), per-region JSON config stores (*TextMatch Label Trainer*), a flat `insurers.json` of 42 records (*UAE OCR AI*), a CSV (*P&C Clause Recommendation Engine*), and Parquet on Hugging Face Hub (*LLM Fine-tuning Platform*). Consequences are documented, not hidden: *Claim Duplication Detector API* cannot detect duplicates across historical submissions because the hash index is request-scoped. No evidence of schema design, migrations, transactions, indexing, or concurrency control on shared state (the documents note the `output.json` writes may not be guarded).

**2. Automated testing.**
Near-total absence. *UAE OCR API — Query-Based Extraction* lists pytest 9.1.1 and httpx 0.28.1 as test tooling — the only such evidence in the corpus. *VL_version_OCR* states automated testing is "Not implemented in the project". Manual verification is strong (live-service response capture in *Document Classification Model*, notebook cell outputs, source-cited constants tables), but there are no unit, integration, contract, or regression suites. The *VL_version_OCR* UAE anchor/output key mismatch — a regression that disables `/extract/uae` entirely — is precisely what a config-consistency test would have caught.

**3. CI/CD and containerisation.**
One project ships a real container: *UAE OCR AI* (Python 3.11-slim, bundled Tesseract, OpenCV/PyMuPDF runtime deps, non-root uvicorn, `--env-file`, targeting on-prem / Azure Container Apps / Kubernetes). Everywhere else deployment is a Uvicorn or Streamlit CLI command, sometimes behind Nginx (*Multi-doc All-in-one OCR*, *Document Detection API*, *TextMatch Label Trainer*). No CI pipeline, build automation, image scanning, or deployment automation is documented in any of the 15 projects. Version control itself is "not stated in documentation" almost everywhere.

**4. Monitoring, observability and alerting.**
Evidence exists but is shallow: `/health` liveness endpoints (several projects), rotating file logs (`OCR_API.log` with 24-day retention in *SLM AI OCR*; `logs/app.log` 10 MB × 5 in *UAE OCR AI*), per-request latency middleware, and append-only audit JSON. Absent: metrics export, tracing, dashboards, alerting, per-model call/cost metrics, or any record of Azure spend actually avoided despite cost being a stated design driver.

**5. Evaluation frameworks and benchmarking — the most consequential gap.**
The one real harness is `eval/run_eval.py` with `values_match()` over a 16-document ground-truth corpus in *Trade License Extractor* — and even there the POC notebook stored no outputs, so no accuracy figure was recorded. Elsewhere: *TextMatch Label Trainer* calibrated 50/65/75% thresholds in a notebook but published no false-acceptance rates; *Multi-doc All-in-one OCR* and *Document Detection API* carried a 50% POC cutoff into production as 65%/90% and the documents label this "drifted parameters" with no recalibration; *Claim Duplication Detector API*'s eight constants were never tuned against a labelled set; *Document Classification Model* prints accuracy during training but the value is not recorded. Almost every project states explicitly that accuracy, throughput, ROI and cost figures are not available.

**6. Security, authentication and authorisation.**
Two genuine credential-management wins: HashiCorp Vault (hvac KV v2) for Basic-Auth credentials in *SLM AI OCR*, and the removal of a hardcoded Azure endpoint and key from *KYC Document Validation System* when the Azure path was dropped. Against that: no API authentication is documented on services ingesting passports, Emirates IDs and civil IDs (*VL_version_OCR*, *Document Detection API*, *Multi-doc All-in-one OCR*, *UAE OCR AI*); `allow_origins=['*']` with `allow_credentials=True` in *SLM AI OCR*; `Bearer none` and hard-coded IPs on the llama.cpp host; a GCP service-account key stored in the project directory; a Streamlit tool that writes live production classification config bound to `0.0.0.0` with no auth (*TextMatch Label Trainer*); and no documented upload size/count limits, rate limiting, TLS, or data-retention policy for PII-bearing audit logs.

**7. Cloud infrastructure beyond managed AI APIs.**
Deep, repeated use of Azure Document Intelligence and Translator, Google Vision and GenAI, Azure AI Services, and Hugging Face Hub — all as consumed SDK endpoints. No evidence of IaC (Terraform/Bicep/ARM), cloud networking, IAM design, managed databases, queues, storage services, autoscaling, or cost-management tooling. Kubernetes and Azure Container Apps are named as deployment *targets* in *UAE OCR AI* but no manifests or orchestration work are described.

**8. MLOps pipelines and model governance.**
Real ingredients exist: model IDs externalised to `.env` for swap without code change and versioned model naming (*DOHA OCR AI*), a YAML training recipe versioned across two iterations (*LLM Fine-tuning Platform*), dataset versioning on the Hub, provenance stored in the joblib bundle (*Document Classification Model*), and per-cell POC-to-production traceability tables in several projects. Missing: a model registry, automated retraining, drift detection, experiment tracking (MLflow/W&B), model cards or approval workflow, and — noted in *Document Classification Model* — provenance recorded as a path *string* rather than a content hash.

**9. Distributed and asynchronous processing at scale.**
In-process concurrency is well handled and deliberately reasoned: `ThreadPoolExecutor` (10 workers; 5/6 in police parsers), thread-local ONNX sessions to avoid mutex serialisation, `asyncio.gather` fan-out, `AsyncOpenAI`, sync handlers offloaded to the AnyIO pool, and multi-worker Uvicorn. Not present: message queues, task workers (Celery/RQ), async job endpoints for long-running work, event streaming, backpressure, retries with exponential backoff, or circuit breakers — the documents raise several of these as unaddressed (e.g. no retry/circuit-breaker around the extractor dispatch in *Multi-doc All-in-one OCR*).

**10. Front-end engineering.**
Streamlit apps across five projects (*TextMatch Label Trainer*, *LLM Fine-tuning Platform*, *KYC Document Validation System*, *Claim Duplication Detector API*, *Multilanguage Speech-to-Text*), one PyQt5 desktop app, and one HTML/JS + Jinja2 canvas UI with coordinate scaling (*KYC Document Validation System*) show competence at internal-tool level. No modern JS framework, component architecture, accessibility, or design-system work appears anywhere.

**11. Dependency and reproducibility hygiene.**
Mixed and worth naming: *SLM AI OCR* pins everything; *Trade License Extractor* pins minimum versions. But *DOHA OCR AI*, *Multi-doc All-in-one OCR*, *Document Detection API* and *VL_version_OCR* ship unpinned requirements, with documented drift (ultralytics 8.4.14 declared vs 8.4.113 installed; NumPy 2.2.6 vs 2.5.2; Transformers 5.5.0 loading a checkpoint written by 5.14.1). No lockfiles or reproducible builds anywhere.

### Recommended Next Skills to Learn

Prioritised for a document-AI + GenAI engineer in insurance; each is tied to a documented gap and a concrete first step.

**1. Evaluation harnesses and benchmarking for extraction systems** *(gap 5 — the highest-leverage item by a wide margin)*
Nearly every project's weakest sentence is "metrics are not recorded", and several production thresholds were shipped uncalibrated. This is what separates the portfolio from a staff-level one.
*First step:* build a labelled ground-truth set of 100–200 crops per document type and a field-level exact-match / normalised-match harness modelled on the existing `values_match()` in *Trade License Extractor*; re-derive the 65%/90% classification thresholds and the OpenCV quality cut-offs from it, and publish precision/recall per document type.

**2. Application security and API authentication for PII workloads** *(gap 6)*
Services ingesting passports, Emirates IDs, civil IDs and medical claims currently have no documented auth. In a GCC insurer under UAE PDPL and Qatar data-handling rules this is the fastest-escalating risk in the portfolio.
*First step:* put OAuth2/JWT or API-key auth plus rate limiting and upload size/count caps in front of one existing FastAPI service (start with *Document Detection API*), tighten the wildcard CORS in *SLM AI OCR*, and move `OCR_key.json` and the llama.cpp `Bearer none` host behind a secrets manager.

**3. Automated testing: pytest, contract tests, and config-consistency tests** *(gap 2)*
The *VL_version_OCR* UAE key mismatch that disables an entire endpoint is a regression a five-line test would have prevented.
*First step:* extend the pytest/httpx setup already present in *UAE OCR API — Query-Based Extraction* into a shared pattern — endpoint smoke tests, response-schema contract tests, and an assertion that every `DOCUMENT_ANCHORS` key exists in `OUTPUT_DICT` per country.

**4. Containerisation, CI/CD and reproducible builds** *(gaps 3 + 11)*
*UAE OCR AI* proves the container skill exists; it just isn't standard practice across the portfolio, and nothing is automated. Documented dependency drift (ultralytics 8.4.14 declared vs 8.4.113 installed, NumPy 2.2.6 vs 2.5.2, Transformers 5.5.0 loading a 5.14.1 checkpoint, Azure SDKs on open-ended `>=1.0.0b1` beta ranges) is a live source of silent breakage.
*First step:* generalise the hardened *UAE OCR AI* Dockerfile into a template for the other FastAPI services, adopt `uv` or `pip-tools` lockfiles with pinned Azure SDK versions, and wire one GitHub Actions (or Azure DevOps) pipeline that lints, runs the new tests, builds and scans the image.

**5. Persistence and data modelling — relational (PostgreSQL) and vector stores** *(gap 1, and an extension of the existing FAISS/RRF retrieval work)*
Several documented limitations dissolve with a database: cross-submission duplicate detection in *Claim Duplication Detector API*, versioned/rollback-able label config in *TextMatch Label Trainer*, and a queryable, retention-managed audit trail instead of unrotated `output.json` files holding identity-document results.
FAISS `IndexFlatL2` (*P&C Clause Recommendation Engine*) and hybrid semantic/BM25 with RRF fusion (*Trade License Extractor*) are real retrieval components, but both are in-process with no persistence, refresh strategy, or index lifecycle.
*First step:* replace `output.json` in *Document Detection API* with a Postgres audit table keyed on `source_id`, add a persistent pHash index (BK-tree or pgvector-style search) so duplicates are matched across historical claims, and move the P&C clause index into a persistent vector store with incremental refresh — then fuse the two P&C pipelines so the historical shortlist grounds the Gemini call and constrains it to real clause names, an issue that document itself raises.

**6. Observability: structured logging, metrics and tracing (OpenTelemetry + Prometheus/Grafana)** *(gap 4)*
Cost avoidance is a stated design driver across the corpus but never measured — there is no record of Azure calls actually avoided by the local bypass paths.
*First step:* emit structured JSON logs plus counters for per-model call volume, cloud-vs-local routing decisions, p95 stage latency and rejection reasons in *UAE OCR AI*, and put them on a dashboard that quantifies the hybrid engine's savings.

**7. LLM evaluation, tracing and cost governance (LLM-as-judge, prompt/version tracking, token accounting)** *(gaps 4 + 5, GenAI-specific)*
With six projects calling LLMs or VLMs (*SLM AI OCR*, *Trade License Extractor*, *P&C Clause Recommendation Engine*, *Multilanguage Speech-to-Text*, *VL_version_OCR*, *LLM Fine-tuning Platform*), several of them inside service request paths, the missing layer is per-request tracing, prompt/schema versioning, and token-cost attribution — plus the never-run evaluation of the fine-tuned Gemma student on the 100-row validation split.
*First step:* run and publish that exact-match evaluation for *LLM Fine-tuning Platform*, then instrument *VL_version_OCR*'s six model modes with per-mode accuracy, latency and token-cost comparison so model selection becomes evidence-based.

**8. Asynchronous job processing and resilience patterns (queues, retries, circuit breakers)** *(gap 9)*
Synchronous request-response is already strained: 5–19 s end-to-end latencies in *SLM AI OCR* against a 10 s design intent, 12–13.5 s worst cases in *Trade License Extractor*, and fan-out to downstream extractors through 14 document-type mappings (13 distinct URLs, all on one host) with no retry or circuit breaker.
*First step:* add an async job endpoint (submit → poll/callback) backed by a task queue for the slowest path in *Multi-doc All-in-one OCR*, with exponential-backoff retries and a circuit breaker per extractor URL.

**9. MLOps: model registry, experiment tracking and reproducible training** *(gap 8)*
The instincts are already there — `.env` model IDs, versioned model names, YAML recipes, provenance in the artefact bundle — they just need real tooling, and provenance should be a content hash rather than a path string.
*First step:* put the *LLM Fine-tuning Platform* runs and the *Document Classification Model* linear probe into MLflow or Weights & Biases, register artefacts with hashes, and move the dataset off a personal Hugging Face namespace into an organisational one.

**10. Cloud infrastructure and IaC (Terraform/Bicep, Kubernetes fundamentals)** *(gap 7)*
*UAE OCR AI* already names Azure Container Apps and Kubernetes as targets with nothing behind them; this converts "deployable" into "deployed and operated".
*First step:* write Bicep or Terraform for one service's Azure footprint (Container App, Key Vault for the Azure DI/Translator keys, log sink) and deploy the containerised *UAE OCR AI* through it.

### LinkedIn Headline Options

1. `AI Engineer — Document Intelligence & Applied GenAI | OCR, VLMs, FastAPI | Arabic/English extraction for insurance claims, KYC & underwriting` *(141 chars)*

2. `Document AI Engineer | Azure Document Intelligence, YOLOv8, Qwen/Gemma VLMs, FastAPI | Building bilingual GCC document extraction at scale` *(138 chars)*

3. `AI Engineer specialising in Intelligent Document Processing | OCR + Vision-Language Models + on-prem LLM serving | Insurance claims, KYC, underwriting` *(150 chars)*

4. `Applied AI Engineer | 15 documented AI systems for a GCC insurer | Azure DI, YOLOv8, QLoRA fine-tuning, llama.cpp, FastAPI | Arabic/English NLP` *(143 chars)*

5. `GenAI & Document AI Engineer | VLM distillation and QLoRA fine-tuning to on-prem GGUF serving | Cost- and data-sovereignty-aware AI for regulated insurance` *(155 chars)*

### Positioning Statement

> I'm an AI engineer specialising in intelligent document processing and applied generative AI, and almost all of my recent work has been building document-AI systems for an insurance group across the UAE and Qatar. I've delivered around fifteen systems end to end, from deployed services to evaluated prototypes — OCR and extraction APIs for driving licences, Emirates and residency IDs, vehicle registrations, trade licences, passports and multi-jurisdiction traffic police reports; document classification and triage services; a claim-duplication detector; a clause-recommendation engine for property underwriting; and a full vision-language-model lifecycle that distilled labels from a 122B teacher, ran them through a human-in-the-loop review tool I built, QLoRA fine-tuned a small Gemma student, and deployed it quantised on-premise so customer PII never leaves internal infrastructure. Technically I work across Azure Document Intelligence, YOLOv8 segmentation, RapidOCR and PyMuPDF, Qwen and Gemini VLMs with strict schema-constrained decoding, and FastAPI services with hard response contracts, per-item error isolation and thread-safe concurrency. What I care most about is engineering judgement rather than reaching for the biggest model: routing digital PDFs to a local parser that runs in milliseconds at zero cloud cost instead of paying per page, using a cheap local OCR pass purely as a router to pick the right cloud model before anything is billed, keeping deterministic extraction first with a guard-railed LLM fallback only where it earns its place, and solving Arabic right-to-left and bilingual normalisation properly because in this domain that's where most pipelines break. I'm equally comfortable owning the messy parts — synthetic training-data generation when a regional tenant only allows template-mode training, or a forensic root-cause write-up when a fine-tuning run memorises its training set — and the part I'm actively pushing hardest on now is measurement: building real evaluation harnesses and observability so the thresholds and routing decisions in these systems are backed by numbers rather than judgement alone.
