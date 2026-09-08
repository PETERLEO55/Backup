## B. Skills Inventory

Depth ratings are judged from breadth (number of projects) and complexity of use (custom training, multi-model orchestration, production hardening) as evidenced in the 15 project documents. Where the documentation is thin, that is stated rather than padded.

### Programming Languages

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Python (3.8–3.12 across projects; 3.11.15 and 3.12.7 verified) | all 15: doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, label_trainer, multidoc_ocr, pnc_recommendation, speech_to_text, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr | Expert |
| Python regular expressions (anchored regex, lookarounds, date/ID parsing) | doha_ocr, kyc, textmatch_classification, multidoc_ocr, trade_license, uae_ocr | Proficient |
| YAML (training recipes, `unsloth_config_v2.yaml`) | llm_finetune | Working |
| JavaScript / HTML (canvas drag-to-select ROI UI, Jinja2 templates) | kyc | Working |

Note: the only non-Python application code documented anywhere is the KYC canvas UI. There is no evidence of Java, Go, C++, TypeScript frameworks, or SQL as authored languages.

### AI/ML

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| PyTorch | dl_doc_classification, claim_duplication, slm_ocr, multidoc_ocr, textmatch_classification, llm_finetune | Proficient |
| Hugging Face Transformers | dl_doc_classification, claim_duplication, llm_finetune | Proficient |
| QLoRA / LoRA fine-tuning (Unsloth, 4-bit BitsAndBytes, r=16, adamw_8bit) | llm_finetune | Proficient |
| Teacher–student distillation (122B teacher → E2B student) | llm_finetune | Proficient |
| Linear-probe classification on frozen embeddings (DINOv2 + LogisticRegression) | dl_doc_classification | Proficient |
| scikit-learn (LogisticRegression, stratified split, classification_report) | dl_doc_classification, llm_finetune | Working |
| Azure Document Intelligence custom model training (doha_ocr: training/iteration in DI Studio under template-only build mode; uae_ocr: ten custom-trained models applied, but the training procedure, dataset and evaluation are not described in that document) | doha_ocr, uae_ocr | Proficient |
| Custom YOLOv8 detector application and threshold tuning | slm_ocr, multidoc_ocr, textmatch_classification, vl_ocr | Proficient |
| Embedding models / vector similarity (all-MiniLM-L6-v2, paraphrase-multilingual-MiniLM-L12-v2, DINOv2, InsightFace, all-mpnet-base-v2) | pnc_recommendation, trade_license, dl_doc_classification, kyc | Proficient |
| FAISS vector search (IndexFlatL2) | pnc_recommendation | Working |
| Failure diagnosis and recipe redesign (catastrophic memorization root-cause, training-distribution gap) | llm_finetune, doha_ocr | Proficient |
| Threshold calibration / decision-boundary tuning (calibrated in label_trainer at 50/65/75%; drift 50% → 65%/90% documented in textmatch_classification and multidoc_ocr; in kyc and claim_duplication the thresholds are set but the documents state they were never tuned against a labelled set) | label_trainer, textmatch_classification, multidoc_ocr, kyc, claim_duplication | Working |
| Model evaluation harnesses (values_match(), stratified hold-out, ground-truth corpora) | trade_license, dl_doc_classification, llm_finetune | Working |
| Perceptual hashing (pHash, Hamming distance) | claim_duplication, dl_doc_classification | Working |
| Face verification (DeepFace / ArcFace) | kyc | Working |

Honest limit: only llm_finetune (QLoRA fine-tuning of Gemma 4 E2B) and dl_doc_classification (fitting a LogisticRegression head on frozen embeddings) involve training or fitting weights in project code; doha_ocr additionally trained Azure Document Intelligence custom extraction models in DI Studio, and uae_ocr/vl_ocr/multidoc_ocr apply custom-trained models whose training is not described. Several documents state that no accuracy, precision/recall, or throughput metrics were recorded — model quality is largely unmeasured across the portfolio.

### NLP

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Fuzzy string matching (RapidFuzz ratio / partial_ratio / token_sort_ratio / token_set) | kyc, label_trainer, multidoc_ocr, textmatch_classification, trade_license, uae_ocr | Expert |
| Arabic NLP: RTL/bidi handling, reshaping, logical vs visual order (arabic-reshaper, python-bidi) | doha_ocr, trade_license, uae_ocr, vl_ocr | Proficient |
| Bilingual Arabic↔English normalization and machine translation (Azure Translator, deep_translator/GoogleTranslator) | doha_ocr, query_ocr, slm_ocr, uae_ocr, vl_ocr | Proficient |
| Structured key-information extraction / slot filling from noisy text | slm_ocr, speech_to_text, trade_license, llm_finetune, vl_ocr, query_ocr | Proficient |
| Keyword-anchor / n-gram document classification | label_trainer, multidoc_ocr, textmatch_classification, slm_ocr, kyc, vl_ocr | Proficient |
| N-gram frequency extraction and keyword discovery | label_trainer, multidoc_ocr | Working |
| Hybrid retrieval (dense semantic + BM25 with RRF fusion) | trade_license | Working |
| Date normalization and parsing (ISO / DD-MM-YYYY, dateparser) | trade_license, uae_ocr, kyc, textmatch_classification, llm_finetune | Working |
| Multi-turn dialogue state management with memory-aware prompting | speech_to_text | Working |
| Entity resolution to canonical codes (insurer alias matching, 42 records) | uae_ocr | Working |

### Computer Vision

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| OpenCV (thresholding, grayscale metrics, crop, decode, Laplacian variance, contour work) | kyc, claim_duplication, multidoc_ocr, textmatch_classification, slm_ocr, label_trainer, trade_license, vl_ocr (in uae_ocr OpenCV is listed only as a Docker runtime dependency) | Expert |
| YOLOv8 object detection, cropping, and class-based routing | slm_ocr, multidoc_ocr, textmatch_classification, vl_ocr | Proficient |
| PDF rasterization (PyMuPDF 150 DPI / 2x, pypdfium2 scale=2, Wand 300 DPI) | doha_ocr, claim_duplication, dl_doc_classification, label_trainer, multidoc_ocr, textmatch_classification, trade_license, uae_ocr, vl_ocr, slm_ocr | Expert |
| Vision transformer embeddings (DINOv2 CLS token, L2-normalised) | dl_doc_classification | Proficient |
| Image-quality assessment (blur, brightness, contrast, resolution gating) | multidoc_ocr, textmatch_classification, kyc | Proficient |
| Vision-language models on document images | vl_ocr, llm_finetune (slm_ocr's Qwen model consumes OCR text, not the image, so it is not counted here) | Proficient |
| Spatial/polygon reasoning: line clustering, RTL word reconstruction, coordinate sorting | slm_ocr, vl_ocr, uae_ocr | Proficient |
| Pillow imaging (EXIF transpose, RGB conversion, downscaling, JPEG encoding) | dl_doc_classification, llm_finetune, vl_ocr, claim_duplication, kyc, label_trainer | Proficient |
| Forensic image analysis (Error Level Analysis, spectral peak concentration) | claim_duplication | Working |
| Synthetic image/PDF data generation (digit-rectangle redaction, raster twins) | doha_ocr | Proficient |

### OCR

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Azure AI Document Intelligence (custom models, prebuilt-read, prebuilt-layout, QUERY_FIELDS) | doha_ocr, query_ocr, slm_ocr, label_trainer, multidoc_ocr, textmatch_classification, uae_ocr, kyc (prototype), vl_ocr (experimental) | Expert |
| RapidOCR (ONNX Runtime, CPU, thread-local sessions) | kyc, label_trainer, multidoc_ocr, textmatch_classification, trade_license | Proficient |
| PyMuPDF native text-layer extraction (no OCR engine) | doha_ocr, trade_license, uae_ocr, vl_ocr | Expert |
| Selective/gated OCR strategy (text-layer first, OCR only on failure) | trade_license, uae_ocr, doha_ocr | Proficient |
| Google Cloud Vision OCR | vl_ocr | Working |
| Tesseract / pytesseract (layout classification, page filtering, zero-char fallback) | uae_ocr, trade_license | Working |
| VLM-as-OCR (single-pass image-grounded extraction) | vl_ocr, llm_finetune (trade_license trialled qwen2.5vl:3b but did not carry it) | Proficient |
| EasyOCR, PaddleOCR, Chrome "searchify" (trialled/abandoned) | kyc, trade_license, doha_ocr | Exposure |

### LLMs

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Google Gemini (gemini-2.0-flash, gemini-3.1-flash-lite-preview, Gemini-3-Flash/Pro-Preview) | speech_to_text, pnc_recommendation, vl_ocr | Proficient |
| Qwen family (Qwen3-VL-4B, Qwen3.5-122B teacher, qwen2.5:3b via Ollama, Qwen via Ollama) | vl_ocr, llm_finetune, trade_license, slm_ocr | Proficient |
| Self-hosted / on-prem LLM serving (Ollama, vLLM 0.27.1, llama.cpp with mmproj) | slm_ocr, trade_license, llm_finetune, vl_ocr | Proficient |
| Constrained/structured decoding (strict JSON schema, `format='json'`, `response_format`, temperature 0) | slm_ocr, trade_license, llm_finetune, vl_ocr, speech_to_text, pnc_recommendation | Expert |
| Prompt engineering (schema-as-prompt, memory-embedded system prompts, `/no_think`, catalog prompts, query-field naming) | query_ocr, slm_ocr, pnc_recommendation, speech_to_text, trade_license, llm_finetune, vl_ocr (textmatch_classification uses no LLM and its document states prompt engineering is not applicable) | Proficient |
| OpenAI SDK / AsyncOpenAI against OpenAI-compatible endpoints | slm_ocr, llm_finetune, vl_ocr | Proficient |
| LangChain Core / LangChain OpenAI / LangChain Google GenAI | llm_finetune, vl_ocr | Working |
| Model quantization and GGUF export (llm_finetune: 4-bit BitsAndBytes QLoRA and Q4_K_S GGUF export, consuming a GPTQ-Int4 teacher; trade_license: running a q4_K_M-quantised model) | llm_finetune, trade_license | Working |
| Multi-provider LLM abstraction (vl_ocr: six selectable modes across on-prem vLLM, Google GenAI and Azure) | vl_ocr | Proficient |
| Guard-railed LLM fallback confined to unknown templates and low-confidence fields | trade_license | Proficient |
| Multimodal VLM fine-tuning (Gemma 4 E2B, frozen vision layers, completion-only loss) | llm_finetune | Proficient |
| RAG (chunking, hybrid semantic/BM25 retrieval with RRF fusion, grounding check) | trade_license (pnc_recommendation runs FAISS retrieval and a Gemini leg as two independent pipelines; its document records retrieval-augmented prompting only as a suggestion, not as built) | Working |
| Azure AI Services hosted Llama-4-Scout | vl_ocr | Exposure |

### Frameworks

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| FastAPI (routers, prefixes/tags, UploadFile multipart, root_path, custom OpenAPI overrides, exception handlers) | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, multidoc_ocr, pnc_recommendation, textmatch_classification, trade_license, uae_ocr, vl_ocr | Expert |
| Pydantic / Pydantic v2 (domain models, BaseSettings, strict schema synthesis) | query_ocr, slm_ocr, pnc_recommendation, trade_license, llm_finetune, vl_ocr | Proficient |
| Streamlit (session state, dialogs, data_editor, chat, audio_input, multi-column layouts) | kyc, label_trainer, pnc_recommendation, claim_duplication, speech_to_text, llm_finetune | Proficient |
| Starlette / AnyIO worker-pool concurrency model (deliberate sync-handler offloading in doha_ocr and uae_ocr; in dl_doc_classification Starlette appears only as the default 405/500 error surface) | doha_ocr, uae_ocr | Proficient |
| Ultralytics (YOLOv8) | slm_ocr, multidoc_ocr, textmatch_classification, vl_ocr | Proficient |
| Unsloth / Unsloth Zoo / TRL | llm_finetune | Working |
| PyQt5 desktop UI | kyc | Working |
| Jinja2 templating | kyc | Working |

### Databases

Evidence here is genuinely thin — this is the weakest area in the portfolio. **No project in the 15 uses a relational or NoSQL database.** Persistence is filesystem- or Hub-based throughout.

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| FAISS vector index (IndexFlatL2) | pnc_recommendation | Working |
| JSON / JSONL file stores used as configuration and audit stores (`labels_DB/<region>_labels.json`, `output.json`, `insurers.json`, `api.log`) | label_trainer, multidoc_ocr, textmatch_classification, uae_ocr, claim_duplication, llm_finetune | Proficient |
| Hugging Face Hub as a versioned Parquet/Arrow dataset store | llm_finetune | Working |
| CSV as a data contract (cp1252 legacy encoding, mandatory columns) | pnc_recommendation | Working |
| Relational databases (PostgreSQL/MySQL/SQL Server), SQL, NoSQL, caching stores (Redis) | none | No evidence |

Several documents explicitly state "no database, no queue and no cache" (dl_doc_classification, claim_duplication, speech_to_text) as a deliberate design choice.

### Cloud

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Microsoft Azure — AI Document Intelligence | doha_ocr, query_ocr, slm_ocr, label_trainer, multidoc_ocr, textmatch_classification, uae_ocr, kyc (prototype), vl_ocr (experimental) | Expert |
| Microsoft Azure — Translator / Cognitive Services Translator | doha_ocr, query_ocr, uae_ocr | Proficient |
| Azure regional/tenant constraint handling (template-only build mode, data residency) | doha_ocr | Proficient |
| Azure AI Services (hosted Llama-4-Scout) | vl_ocr | Exposure |
| Azure Container Apps (named deployment target) | uae_ocr | Exposure |
| Google Cloud (Vision API, GenAI/Gemini, service-account auth) | vl_ocr, speech_to_text, pnc_recommendation | Working |
| Hugging Face Hub (token-authenticated dataset versioning) | llm_finetune | Working |
| On-premise / private-host inference (vLLM host, Ollama endpoint, llama.cpp server) | slm_ocr, llm_finetune, vl_ocr, trade_license | Proficient |
| Hybrid cloud/on-prem architecture for PII sovereignty and cost | slm_ocr, llm_finetune, doha_ocr, textmatch_classification, trade_license | Proficient |
| AWS / GCP compute, managed Kubernetes, serverless, IaC (Terraform) | none | No evidence |

Azure is the only hyperscaler used in depth, and only its cognitive services — not compute, storage, identity, or networking.

### APIs

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| REST API design (multipart/form-data upload contracts, versioned namespaces, enum query params) | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, multidoc_ocr, pnc_recommendation, textmatch_classification, trade_license, uae_ocr, vl_ocr | Expert |
| OpenAPI / Swagger UI / ReDoc (auto-generated plus custom schema overrides for binary arrays) | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, multidoc_ocr, pnc_recommendation, textmatch_classification, uae_ocr, vl_ocr | Proficient |
| Uniform response envelopes (`message` / `messageType` 'S'/'E' / `response`) | doha_ocr, query_ocr, slm_ocr, multidoc_ocr, textmatch_classification, uae_ocr | Expert |
| Structured error contracts with correct HTTP semantics (400/404/415/422/500/502) | query_ocr, uae_ocr, textmatch_classification, multidoc_ocr, dl_doc_classification, claim_duplication | Proficient |
| Third-party API consumption (Azure DI/Translator, Gemini, Google Vision, Ollama, vLLM, HF Hub, Vault KV v2) | doha_ocr, query_ocr, slm_ocr, uae_ocr, vl_ocr, llm_finetune, trade_license, speech_to_text, pnc_recommendation | Expert |
| Service-to-service dispatch / hub-and-spoke routing (14 extractor mappings, timeouts, error isolation) | multidoc_ocr | Proficient |
| HTTP Basic Auth | slm_ocr | Working |
| Health/liveness probes for monitoring and orchestrators (CORS middleware documented in slm_ocr and query_ocr only) | slm_ocr, query_ocr, doha_ocr, multidoc_ocr, textmatch_classification, uae_ocr, pnc_recommendation, dl_doc_classification | Proficient |
| API authentication beyond Basic Auth (OAuth2/JWT/API keys, rate limiting) | none | No evidence — repeatedly listed as a documented gap |

### DevOps

Evidence is thin and honest reporting matters here: **no project documents CI/CD, and only one documents containerization.** Multiple documents state that tests, CI/CD, monitoring, and version control are "not stated in documentation."

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Uvicorn ASGI serving (worker counts, dev `--reload` vs production binds) | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, multidoc_ocr, pnc_recommendation, textmatch_classification, uae_ocr, vl_ocr | Proficient |
| Nginx reverse proxy (path-prefix routing, WebSocket upgrade, root_path alignment) | label_trainer, multidoc_ocr, textmatch_classification | Proficient |
| Docker (hardened image: Python 3.11-slim, bundled Tesseract, non-root uvicorn, `--env-file`) | uae_ocr | Working |
| `.env` / environment-driven configuration | doha_ocr, slm_ocr, multidoc_ocr, uae_ocr, vl_ocr, query_ocr | Proficient |
| Secrets management (HashiCorp Vault KV v2 via hvac) | slm_ocr | Working |
| Rotating file logging and latency audit middleware (rotating handlers plus per-request latency middleware in slm_ocr and uae_ocr; claim_duplication logs to stdout and a plain `api.log` audit file with no rotation) | slm_ocr, uae_ocr, claim_duplication | Working |
| Health endpoints for orchestrators/monitoring | doha_ocr, query_ocr, slm_ocr, multidoc_ocr, textmatch_classification, uae_ocr, pnc_recommendation, dl_doc_classification | Proficient |
| Dependency management (requirements.txt; often unpinned — noted as a defect in several projects) | doha_ocr, slm_ocr, multidoc_ocr, textmatch_classification, trade_license, vl_ocr | Working |
| Kubernetes | uae_ocr (named as a deployment target only) | Exposure |
| Git / version control | claim_duplication (inferred only from a .gitignore) | Exposure — "not stated in documentation" in 13 of 15 projects |
| CI/CD pipelines, container registries, IaC, observability stacks | none | No evidence |
| Automated testing (pytest + httpx) | query_ocr | Working — the only project with any test tooling |

### Data Processing

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Multi-format document ingestion (JPEG/PNG/TIFF/BMP/PDF, multi-page) into uniform buffers | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, multidoc_ocr, textmatch_classification, trade_license, uae_ocr, vl_ocr, label_trainer | Expert |
| In-memory / stateless byte-stream processing (no disk writes, UploadFile streaming) | doha_ocr, query_ocr, slm_ocr, claim_duplication, speech_to_text, dl_doc_classification | Proficient |
| Schema normalization across heterogeneous sources into one JSON contract | uae_ocr, multidoc_ocr, vl_ocr, doha_ocr, textmatch_classification | Expert |
| Synthetic training-data generation (300+ balanced samples, layout variants, raster twins) | doha_ocr | Proficient |
| Dataset construction, stratified splitting, Parquet/Arrow sharding | llm_finetune, dl_doc_classification | Proficient |
| Data auditing and label QA (forensic audit, duplicate/date-swap/unit-contamination detection) | llm_finetune, doha_ocr | Proficient |
| pandas tabular processing and aggregation | pnc_recommendation, trade_license, speech_to_text | Working |
| NumPy array handling | kyc, pnc_recommendation, dl_doc_classification, slm_ocr, uae_ocr | Working |
| PDF table and block-span extraction (vector tables, coordinate-indexed cells) | uae_ocr, vl_ocr, doha_ocr, trade_license | Proficient |
| Provenance and audit-trail capture (per-field confidence, page/method, `edited_by`/`edited_fields`, append-only logs) | trade_license, llm_finetune, multidoc_ocr, textmatch_classification, claim_duplication | Proficient |
| Legacy encoding handling (cp1252, mixed Arabic encodings, mojibake) | pnc_recommendation, doha_ocr, trade_license | Working |

### Automation

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Concurrency: ThreadPoolExecutor, asyncio/asyncio.gather, thread-local engine singletons | multidoc_ocr, textmatch_classification, vl_ocr, slm_ocr, uae_ocr, kyc (asyncio in the superseded Azure prototype only) | Proficient |
| Jupyter notebook experimentation programmes with cell-level traceability to production code | doha_ocr, query_ocr, kyc, label_trainer, multidoc_ocr, textmatch_classification, trade_license, dl_doc_classification, llm_finetune, vl_ocr | Expert |
| Batch generation / batch folder processing scripts | doha_ocr, trade_license | Working |
| Hot-reload configuration pipelines (write-is-the-deployment JSON stores) | label_trainer, multidoc_ocr, textmatch_classification | Proficient |
| Human-in-the-loop annotation tooling with auto-save and provenance | llm_finetune, label_trainer, kyc | Proficient |
| Document generation scripts (`build_all_artifacts.py`, `generate_doc.py`, python-docx, openpyxl) | llm_finetune, claim_duplication | Working |
| Windows GUI automation (pyautogui/pyperclip/pygetwindow) | doha_ocr (POC, abandoned) | Exposure |
| Workflow schedulers (Airflow, Prefect, cron pipelines), CI-driven retraining | none | No evidence |

### Analytics

Thin area. There is no BI tooling, no SQL analytics, and — critically — most projects explicitly record that accuracy, throughput, cost, and ROI figures were never measured.

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Actuarial frequency/recency scoring and ranking over historical portfolio data | pnc_recommendation | Working |
| Latency benchmarking and per-endpoint performance profiling | slm_ocr, trade_license, doha_ocr, multidoc_ocr, textmatch_classification | Working |
| Comparative engine/model benchmarking (RapidOCR vs Azure, EasyOCR vs RapidOCR, ratio vs token_sort_ratio, pHash vs kNN) | kyc, label_trainer, multidoc_ocr, dl_doc_classification, trade_license | Proficient |
| Evaluation harness design and ground-truth corpora | trade_license, dl_doc_classification, llm_finetune | Working |
| Matplotlib visualization | kyc (notebook only), llm_finetune | Exposure |
| BI tools (Power BI/Tableau/Looker), SQL analytics, statistical A/B testing | none | No evidence |

Explicit statements of missing metrics appear in query_ocr, kyc, label_trainer, pnc_recommendation, speech_to_text, textmatch_classification, uae_ocr, vl_ocr, and partially in doha_ocr, multidoc_ocr, dl_doc_classification and llm_finetune.

### Architecture Design

| Skill/Technology | Evidence (project keys) | Depth |
| --- | --- | --- |
| Multi-stage AI pipeline design (segment → OCR → classify → validate → extract → route) | slm_ocr, multidoc_ocr, textmatch_classification, vl_ocr, uae_ocr, trade_license | Expert |
| Cost/quota-driven hybrid routing (local parse vs cloud OCR; on-prem LLM vs cloud) | doha_ocr, uae_ocr, trade_license, textmatch_classification, multidoc_ocr, slm_ocr | Expert |
| Provider-agnostic / pluggable engine abstractions (dual OCR, six VLM modes, swappable model IDs) | multidoc_ocr, textmatch_classification, label_trainer, vl_ocr, query_ocr, trade_license | Expert |
| Configuration-as-data extensibility (regions, document types, anchors, schemas as JSON/env, not code) | label_trainer, multidoc_ocr, textmatch_classification, slm_ocr, vl_ocr, doha_ocr | Expert |
| Stateless, horizontally scalable service design | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, textmatch_classification | Proficient |
| Service decoupling (triage vs extraction; hub-and-spoke to extractor microservices) | textmatch_classification, multidoc_ocr, vl_ocr | Proficient |
| Blast-radius-limited modular design (star-shaped imports, per-jurisdiction handlers) | uae_ocr, vl_ocr, trade_license | Proficient |
| Fail-fast validation gates ahead of expensive calls (anchor checks, quality gates, field-count gates) | slm_ocr, uae_ocr, multidoc_ocr, textmatch_classification, doha_ocr, label_trainer | Expert |
| Deterministic-first design with guard-railed AI fallback | trade_license, doha_ocr, uae_ocr | Proficient |
| Human-in-the-loop quality gates in the pipeline | llm_finetune, label_trainer, kyc, claim_duplication | Proficient |
| Concurrency architecture (sync handlers on worker pools, bounded thread pools, timeout isolation) | doha_ocr, uae_ocr, slm_ocr, multidoc_ocr, textmatch_classification, vl_ocr | Proficient |
| Data-sovereignty-driven architecture (PII on-prem, in-tenant training assets) | slm_ocr, llm_finetune, doha_ocr, trade_license | Proficient |
| Documented candour about architectural gaps (auth, retries, observability, retention) | uae_ocr, textmatch_classification, vl_ocr, dl_doc_classification, label_trainer, kyc | Proficient |

---

## C. Technology Frequency Analysis

Counts below merge synonyms and version variants into a single canonical name (e.g. every "Azure Document Intelligence" / "Azure AI Document Intelligence" / "azure-ai-documentintelligence" / named custom model collapses to one entry; all YOLOv8/Ultralytics variants to one; all Uvicorn pins to one). A project is counted once per canonical technology. Maximum possible count is 15.

### 1. Ranked Technology Usage

Technologies appearing in two or more projects:

| Rank | Technology | # projects | Projects (keys) |
| --- | --- | --- | --- |
| 1 | Python | 15 | all 15 |
| 2 | FastAPI | 12 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, multidoc_ocr, pnc_recommendation, textmatch_classification, trade_license, uae_ocr, vl_ocr |
| 3 | Uvicorn (ASGI) | 11 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, multidoc_ocr, pnc_recommendation, textmatch_classification, uae_ocr, vl_ocr |
| 4 | Swagger UI / OpenAPI auto-docs | 11 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, multidoc_ocr, pnc_recommendation, textmatch_classification, uae_ocr, vl_ocr |
| 5 | Jupyter Notebook (experimentation) | 10 | doha_ocr, query_ocr, kyc, label_trainer, multidoc_ocr, textmatch_classification, trade_license, dl_doc_classification, llm_finetune, vl_ocr |
| 6 | PyMuPDF (fitz) | 9 | doha_ocr, claim_duplication, dl_doc_classification, label_trainer, multidoc_ocr, textmatch_classification, trade_license, uae_ocr, vl_ocr |
| 7 | Azure AI Document Intelligence (all model types) | 9 | doha_ocr, query_ocr, slm_ocr, label_trainer, multidoc_ocr, textmatch_classification, uae_ocr, kyc* (Azure Form Recognizer prototype, removed), vl_ocr* (experimental only) |
| 8 | OpenCV | 8 | kyc, claim_duplication, slm_ocr, label_trainer, multidoc_ocr, textmatch_classification, trade_license, vl_ocr |
| 9 | Pillow (PIL) | 6 | dl_doc_classification, llm_finetune, vl_ocr, claim_duplication, kyc, label_trainer |
| 10 | Pydantic (incl. Pydantic-Settings) | 6 | query_ocr, slm_ocr, pnc_recommendation, trade_license, llm_finetune, vl_ocr |
| 11 | RapidFuzz | 6 | kyc, label_trainer, multidoc_ocr, textmatch_classification, trade_license, uae_ocr |
| 12 | Streamlit | 6 | kyc, label_trainer, pnc_recommendation, claim_duplication, speech_to_text, llm_finetune |
| 13 | PyTorch (torch/torchvision) | 6 | claim_duplication, dl_doc_classification, slm_ocr, multidoc_ocr, textmatch_classification, llm_finetune |
| 14 | `.env` / environment-driven configuration | 6 | doha_ocr, query_ocr, slm_ocr, multidoc_ocr, uae_ocr, vl_ocr |
| 15 | Python `re` / regex extraction | 6 | doha_ocr, kyc, multidoc_ocr, textmatch_classification, trade_license, uae_ocr |
| 16 | Concurrency primitives (ThreadPoolExecutor / asyncio) | 6 | multidoc_ocr, textmatch_classification, vl_ocr, slm_ocr, uae_ocr, kyc* (prototype only) |
| 17 | requirements.txt dependency manifests | 6 | doha_ocr, slm_ocr, multidoc_ocr, textmatch_classification, speech_to_text, vl_ocr |
| 18 | NumPy | 5 | kyc, pnc_recommendation, dl_doc_classification, slm_ocr, uae_ocr |
| 19 | RapidOCR (rapidocr_onnxruntime / ONNX models) | 5 | kyc, label_trainer, multidoc_ocr, textmatch_classification, trade_license* (named as an OCR option, PaddleOCR/Tesseract also cited) |
| 20 | Ultralytics YOLOv8 (custom weights `yolo-multipage-OCR-cls.pt`) | 4 | slm_ocr, multidoc_ocr, textmatch_classification, vl_ocr |
| 21 | arabic-reshaper | 4 | doha_ocr* (abandoned v1/v2 synthetic generator), trade_license, uae_ocr, vl_ocr |
| 22 | python-multipart | 4 | query_ocr, dl_doc_classification, slm_ocr, claim_duplication |
| 23 | Qwen models (Qwen3-VL-4B, Qwen3.5-122B teacher, qwen2.5:3b, Ollama 'qwen') | 4 | slm_ocr, trade_license, llm_finetune, vl_ocr |
| 24 | Hugging Face Transformers / Hub | 3 | claim_duplication, dl_doc_classification, llm_finetune |
| 25 | Google Gemini (all variants) | 3 | speech_to_text, pnc_recommendation, vl_ocr |
| 26 | Azure Translator / Cognitive Services Translator | 3 | doha_ocr, query_ocr, uae_ocr |
| 27 | Nginx | 3 | label_trainer, multidoc_ocr, textmatch_classification |
| 28 | python-bidi | 3 | trade_license, uae_ocr, vl_ocr |
| 29 | Ollama (self-hosted LLM serving) | 3 | slm_ocr, trade_license, llm_finetune |
| 30 | OpenAI Python SDK (AsyncOpenAI) | 3 | slm_ocr, llm_finetune, vl_ocr |
| 31 | pandas | 3 | pnc_recommendation, trade_license, speech_to_text |
| 32 | Machine translation via deep-translator / GoogleTranslator | 2 | slm_ocr, vl_ocr |
| 33 | pypdfium2 | 2 | slm_ocr, vl_ocr |
| 34 | LangChain (Core / OpenAI / Google GenAI) | 2 | llm_finetune, vl_ocr |
| 35 | scikit-learn | 2 | dl_doc_classification, llm_finetune |
| 36 | ImageHash (pHash) | 2 | claim_duplication, dl_doc_classification* (experimental baseline) |
| 37 | Tesseract / pytesseract | 2 | uae_ocr, trade_license |
| 38 | llama.cpp | 2 | llm_finetune, trade_license* (experimental remote server, timed out) |
| 39 | Sentence/multilingual embedding models (MiniLM family) | 2 | pnc_recommendation, trade_license |

\* marks a project where the technology is present in a prototype, notebook, experimental or abandoned role rather than the production path; it is still counted once.

**Single-project technologies, grouped by category**

- **AI/ML models & training:** DINOv2-base frozen backbone (dl_doc_classification); Unsloth / Unsloth Zoo / TRL, BitsAndBytes 4-bit, Gemma 4 E2B, Qwen3.5-122B GPTQ-Int4 teacher, vLLM 0.27.1, GGUF Q4_K_S + mmproj, Hugging Face Datasets (llm_finetune); FAISS IndexFlatL2, all-MiniLM-L6-v2 (pnc_recommendation); Ateeqq/ai-vs-human-image-detector, Error Level Analysis, spectral analysis (claim_duplication); ArcFace/DeepFace, InsightFace buffalo_l, all-mpnet-base-v2, ONNX Runtime (kyc); paraphrase-multilingual-MiniLM-L12-v2, fastembed/ONNX, BM25 + RRF fusion (trade_license); Dev-Llama-4-Scout-17B-16E-Instruct (vl_ocr).
- **OCR & document processing:** Azure DI QUERY_FIELDS zero-shot (query_ocr); Google Cloud Vision, Wand/ImageMagick, custom coordinates_sort.py (vl_ocr); PaddleOCR, dateparser, phonenumbers (trade_license); EasyOCR, Azure Form Recognizer (kyc); Chrome "searchify" OCR, Azure DI Studio, synthetic generator scripts (doha_ocr); python-dateutil (uae_ocr).
- **Frameworks & UI:** PyQt5, Jinja2, HTML/JS canvas (kyc); Streamlit CLI runner behind Nginx (label_trainer); streamlit-mic-recorder, google-genai SDK, st.audio_input (speech_to_text). (Starlette is not single-project: it backs FastAPI in doha_ocr and uae_ocr as the AnyIO worker pool and in dl_doc_classification as the default error surface; Matplotlib appears in kyc and llm_finetune notebooks.)
- **Infra & DevOps:** Docker hardened image, Kubernetes/Azure Container Apps targets, RotatingFileHandler (uae_ocr); HashiCorp Vault + hvac, TimedRotatingFileHandler, gc.collect() memory hygiene (slm_ocr); Conda environment (vl_ocr); pytest + httpx (query_ocr); Git (claim_duplication, inferred).
- **Utilities:** functools.lru_cache (query_ocr); joblib (dl_doc_classification); python-docx, openpyxl, YAML recipes (llm_finetune); Requests, hvac (slm_ocr); pyautogui/pyperclip/pygetwindow (doha_ocr, POC); UUID generation (textmatch_classification, kyc).

### 2. Skill-Category Coverage

| Category | # projects | Projects |
| --- | --- | --- |
| Software engineering | 15 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, label_trainer, multidoc_ocr, pnc_recommendation, speech_to_text, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| AI engineering | 15 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, label_trainer, multidoc_ocr, pnc_recommendation, speech_to_text, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| Data engineering | 15 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, label_trainer, multidoc_ocr, pnc_recommendation, speech_to_text, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| System design | 15 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, label_trainer, multidoc_ocr, pnc_recommendation, speech_to_text, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| Machine learning | 14 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, label_trainer, multidoc_ocr, pnc_recommendation, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| API development | 14 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, label_trainer, multidoc_ocr, pnc_recommendation, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| NLP | 13 | doha_ocr, query_ocr, slm_ocr, kyc, label_trainer, multidoc_ocr, pnc_recommendation, speech_to_text, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| Computer vision | 13 | doha_ocr, query_ocr, slm_ocr, claim_duplication, dl_doc_classification, kyc, label_trainer, multidoc_ocr, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| Cloud | 11 | doha_ocr, query_ocr, slm_ocr, kyc, label_trainer, multidoc_ocr, speech_to_text, textmatch_classification, uae_ocr, llm_finetune, vl_ocr |
| MLOps | 11 | doha_ocr, slm_ocr, dl_doc_classification, label_trainer, multidoc_ocr, pnc_recommendation, textmatch_classification, trade_license, uae_ocr, llm_finetune, vl_ocr |
| Prompt engineering | 8 | query_ocr, slm_ocr, pnc_recommendation, speech_to_text, textmatch_classification, trade_license, llm_finetune, vl_ocr |

Two caveats on this table, which reproduces the per-project category tagging: the textmatch_classification document states explicitly that prompt engineering is *not applicable* (no LLM is used), and several "machine learning" entries are model selection or applied inference rather than training — query_ocr records "model selection only", slm_ocr "deployment of a trained YOLOv8 detector; training methodology not documented", claim_duplication "inference-only use of a transformers vision model".

Note on MLOps: the count reflects lightweight practices actually documented — externalized model IDs, versioned model naming, health endpoints, experiment-to-production traceability, local weight management. None of the 11 projects documents a model registry, CI/CD, or production monitoring stack.

### 3. Strongest Technical Areas

**1. Document AI / OCR pipeline engineering (13+ of 15 projects).** This is the spine of the portfolio. The work spans every strategy in the space: Azure DI custom-model training under a template-only regional tenant (doha_ocr, uae_ocr), zero-shot query fields on prebuilt-layout (query_ocr), local CPU OCR with thread-local ONNX sessions (multidoc_ocr, textmatch_classification), pure text-layer parsing with no OCR at all (trade_license, doha_ocr police reports, uae_ocr digital PDFs), and VLM-as-OCR (vl_ocr, slm_ocr, llm_finetune). The routing decisions are consistently cost-and-latency reasoned rather than defaulted.

**2. Production Python API engineering with FastAPI (12 of 15 projects).** FastAPI appears in twelve services with genuinely varied surface design: APIRouter prefixes and tags, `root_path` for reverse-proxy mounting, custom `get_openapi` overrides so binary array uploads render correctly (multidoc_ocr, textmatch_classification), a custom `OCRError`-to-HTTP mapper across 400/404/422/500/502 (uae_ocr), and uniform success/error envelopes that keep downstream claims pipelines from crashing (doha_ocr, slm_ocr, query_ocr). Concurrency is handled deliberately — synchronous handlers on Starlette's AnyIO pool for blocking I/O, bounded ThreadPoolExecutors for CPU-bound parsing.

**3. LLM/VLM application engineering and constrained decoding (8+ projects).** Across slm_ocr, trade_license, vl_ocr, llm_finetune, speech_to_text, pnc_recommendation and query_ocr, the consistent pattern is treating an LLM as an unreliable component to be fenced: strict JSON schemas with `additionalProperties=False`, `temperature=0`, `format='json'`, `/no_think` reasoning suppression, anchor gates that reject documents *before* paying for inference, and guard-railed fallbacks confined to unknown templates and low-confidence fields. vl_ocr goes further with a six-mode provider abstraction spanning on-prem vLLM, Google GenAI and Azure.

**4. Architecture design under real constraints (15 of 15 projects).** Every project shows an explicit constraint being designed around: per-page Azure OCR cost and transaction quota (doha_ocr, uae_ocr, textmatch_classification), template-only build mode in a regional tenant (doha_ocr), PII data sovereignty (slm_ocr, llm_finetune, trade_license), CPU-only hardware with no GPU (trade_license, dl_doc_classification), and jurisdiction variance across the GCC (multidoc_ocr, textmatch_classification, vl_ocr). The recurring answer — configuration-as-data, pluggable engines, deterministic-first with AI fallback — is a coherent architectural point of view, not an accident.

**5. Computer vision for document segmentation and quality triage (13 projects).** A custom-trained YOLOv8 detector (`yolo-multipage-OCR-cls.pt`; the documents say the weights are custom-trained but never describe the dataset, labelling or training run, and never name who produced them) is reused across four services to split multi-document captures and multi-page PDFs, with threshold tuning documented (conf 0.5/IoU 0.4 in vl_ocr; imgsz=320/conf=0.25 in multidoc_ocr). Alongside it sits a full OpenCV quality-gate stack (Laplacian-variance blur, brightness bounds, contrast std-dev, resolution) and a DINOv2 frozen-backbone linear probe for OCR-free document classification.

**6. Arabic / bilingual NLP (5+ projects).** This is a genuinely differentiated skill: RTL bidirectional wrap resolution in PDF text layers with bespoke `transfer_window()` / `fix_number_wrap()` helpers (doha_ocr), the logical-vs-visual-vs-presentation-form encoding failure that killed two synthetic-generator versions before v3 shipped, right-to-left word reconstruction from OCR polygons (uae_ocr), and Arabic-sanity lexicon checks used as an OCR routing gate (trade_license). Bidirectional Arabic↔English normalization appears in doha_ocr, query_ocr, slm_ocr, uae_ocr and vl_ocr.

**7. Fuzzy matching and rule-based classification systems (6 projects).** RapidFuzz is applied with real discrimination: `ratio` for single tokens versus `token_sort_ratio` for phrases at cutoff 80 (multidoc_ocr, textmatch_classification), `partial_ratio` at 70 for OCR error correction and at 80 for suggestion clustering (kyc, label_trainer), and token-sort/token-set entity resolution against a 42-record insurer catalogue at threshold 85 (uae_ocr). label_trainer wraps this in a no-code admin tool with an empirical acceptance gate before rules reach production.

**8. Model fine-tuning and the full ML lifecycle (2 projects, high depth).** Narrow in breadth but the deepest single piece of work: llm_finetune runs teacher-VLM distillation, a Streamlit HITL annotation tool used by three named reviewers over 664 UAE and 296 Doha records, a forensic label audit in which 12 of 30 flagged field labels were wrong (a 40% error rate on flagged fields), versioned Hugging Face dataset splits, a documented catastrophic-memorization failure with root-cause analysis, a redesigned recipe, and GGUF quantization to an on-prem llama.cpp deployment verified live. dl_doc_classification adds a disciplined frozen-backbone linear probe with train/serve preprocessing parity.

### 4. Notable Absences

These are technologies a reader might expect from this profile that appear in zero or one project. Stated as evidence, not speculation.

- **Relational databases and SQL — zero projects.** No PostgreSQL, MySQL, SQL Server, SQLite, or ORM appears anywhere. Persistence is JSON files (`labels_DB/*.json`, `output.json`, `insurers.json`), local disk, or Hugging Face Hub. Three documents explicitly state "no database, no queue and no cache" as a deliberate choice (dl_doc_classification, claim_duplication, speech_to_text).
- **NoSQL and caching infrastructure — zero projects.** No MongoDB, DynamoDB, Elasticsearch, or Redis. The only caching documented is an in-process `functools.lru_cache(maxsize=1024)` in query_ocr, which the document itself notes is per-process and lost on restart.
- **CI/CD — zero projects.** No GitHub Actions, GitLab CI, Jenkins, or any build pipeline is documented in any of the 15. Multiple documents list this as an explicit gap.
- **Version control — effectively zero.** Fourteen of the fifteen documents name no version-control system (recorded either as "not stated in documentation" or as an empty entry). claim_duplication infers Git only from `api.log` being git-ignored; llm_finetune versions datasets via Hugging Face Hub commits rather than a code VCS.
- **Automated testing — one project.** Only query_ocr documents test tooling (pytest 9.1.1 + httpx 0.28.1). dl_doc_classification, slm_ocr, vl_ocr, kyc and others state that automated tests do not exist.
- **Containerization — one project.** Only uae_ocr documents a Dockerfile (Python 3.11-slim, bundled Tesseract, non-root uvicorn). Kubernetes and Azure Container Apps appear in that same project as named deployment targets only, with no manifests described.
- **Message queues and event streaming — zero projects.** No Kafka, RabbitMQ, Celery, SQS, or task queue. Concurrency is in-process (ThreadPoolExecutor, asyncio); several documents name a queue as a suggested future improvement rather than something built.
- **API authentication and authorization — near zero.** Only slm_ocr documents HTTP Basic Auth (credentials from HashiCorp Vault). No OAuth2, JWT, API keys, rate limiting, or TLS termination is documented in any service, and uae_ocr, textmatch_classification, vl_ocr, dl_doc_classification, label_trainer and kyc all name this as an open gap — notable given these services ingest passports, Emirates IDs and civil IDs.
- **Observability and monitoring — near zero.** Health endpoints and rotating file logs are the ceiling (uae_ocr, slm_ocr, claim_duplication). No Prometheus, Grafana, OpenTelemetry, APM, alerting, or distributed tracing appears in any project.
- **Infrastructure as code — zero projects.** No Terraform, CloudFormation, Pulumi, Helm, or Ansible.
- **AWS and GCP compute — near zero.** Azure is used in depth but only its cognitive services. GCP appears solely as Vision API and GenAI/Gemini endpoints (vl_ocr, speech_to_text, pnc_recommendation). AWS appears in zero projects.
- **BI and analytics tooling — zero projects.** No Power BI, Tableau, Looker, dbt, Spark, or warehouse work. Matplotlib appears in two projects, notebook-only.
- **Frontend frameworks — zero projects.** No React, Vue, or Angular. UI work is Streamlit (six projects), one PyQt5 desktop app, and one hand-written HTML/JS canvas (both kyc).
- **Measured model quality — largely absent.** This is the most consequential gap. query_ocr, kyc, label_trainer, pnc_recommendation, speech_to_text, textmatch_classification, uae_ocr and vl_ocr all state explicitly that no accuracy, throughput, cost or ROI figures were recorded; doha_ocr, multidoc_ocr, dl_doc_classification and llm_finetune record only architectural benchmarks (latencies, artifact sizes, corpus counts) rather than field-level accuracy. Only trade_license (`eval/run_eval.py`, `values_match()` over a 16-document ground-truth corpus), dl_doc_classification (a stratified hold-out with a printed accuracy and classification report whose values are not recorded) and llm_finetune (an 85/15 split) document any evaluation apparatus at all, and llm_finetune's v2 evaluation result is itself unreported.
