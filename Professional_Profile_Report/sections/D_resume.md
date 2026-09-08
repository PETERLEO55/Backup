## D. Resume Content

### Professional Summary

> *Note on titles:* the project documents never state a job title. Ownership level in the source analyses is **inferred** from documented scope (AI Engineer to Senior AI Engineer, depending on the project). The summaries below use "AI Engineer" as a self-description, not as a documented title.

**3-line variant**

> AI Engineer specialising in document intelligence for insurance — OCR, key information extraction, document classification and vision-language models across UAE, Qatar, Oman and Kuwait identity, vehicle, trade-licence and police-report documents.
> Built 15 production and pilot systems for Qatar Insurance Group spanning FastAPI microservices, Azure Document Intelligence and self-hosted VLM/LLM pipelines, YOLOv8 document segmentation, and Arabic/English bilingual normalisation.
> Comfortable end-to-end: model selection and fine-tuning (QLoRA on Gemma 4 E2B), synthetic training data generation, strict-schema LLM decoding, containerised on-premise deployment, and honest documentation of limits and unmeasured claims.

**5-line variant**

> AI Engineer focused on document intelligence and applied LLM/VLM systems for the insurance domain, delivering 15 projects for Qatar Insurance Group covering motor claims intake, KYC onboarding, underwriting support and claims pre-screening.
> Designed hybrid extraction architectures that route digital PDFs to local parsers (PyMuPDF, regex, rule-based) and only raster scans to cloud OCR, cutting per-page cloud OCR spend and enabling operation in restricted networks.
> Trained, integrated and deployed custom models across the stack: Azure Document Intelligence custom extraction models, a frozen DINOv2 + logistic-regression classifier, a custom-weights YOLOv8 multi-document detector (integration and threshold tuning; training authorship is not stated in the source documents), and a QLoRA fine-tune of Gemma 4 E2B distilled from a Qwen3.5-122B teacher and served on-premise via llama.cpp.
> Engineered Arabic/English bilingual pipelines — RTL bidirectional wrap correction in PDF text layers, arabic-reshaper/python-bidi handling, Azure Translator and GoogleTranslator enrichment, and RapidFuzz entity resolution against insurer master data.
> Delivered production concerns alongside models: strict JSON contracts, uniform error envelopes, health probes, thread-local inference sessions, HashiCorp Vault secret loading, rotating audit logs and Docker/Nginx/Uvicorn deployment.

**LinkedIn "About" (first person, ~180 words)** — *names the client; if your NDA does not permit that, substitute "a GCC insurance group", as the consolidated report's LinkedIn draft does.*

> I build document intelligence systems for insurance. Across fifteen projects for Qatar Insurance Group I have taken identity cards, vehicle registrations, trade licences, medical claim batches and traffic-police accident reports from a scanned page to validated, bilingual JSON that underwriting and claims platforms can consume directly.
>
> Most of my work sits where machine learning meets real engineering constraints. I have trained Azure Document Intelligence custom models under a regional tenant that only allowed template build mode, and built a synthetic data pipeline that regenerated digit-bearing rectangles inside real reports to produce 300+ balanced training samples. I have distilled a Qwen3.5-122B teacher into a Gemma 4 E2B student with 4-bit QLoRA, diagnosed a catastrophic memorisation failure through forensic review, redesigned the recipe, and shipped the quantised model on-premise via llama.cpp so customer PII never leaves internal infrastructure.
>
> I care about Arabic/English correctness, strict output schemas that downstream systems can trust, and documenting what a system genuinely does — including what has not been measured yet. Python, FastAPI, PyTorch, Azure AI, YOLO, vLLM, Unsloth.

### Technical Skills Section

**Languages:** Python, JavaScript, HTML, YAML (no SQL/database work appears in these projects; persistence is JSON/JSONL/Parquet and the local filesystem)

**AI/ML & Deep Learning:** PyTorch, Hugging Face Transformers, scikit-learn, Ultralytics YOLOv8, DINOv2 (frozen ViT feature extraction), LogisticRegression linear probes, ArcFace/DeepFace face verification, InsightFace (evaluated), SentenceTransformers, FAISS (IndexFlatL2), ImageHash (pHash), ONNX Runtime, BitsAndBytes 4-bit quantisation

**LLMs & GenAI:** Google Gemini (gemini-2.0-flash, gemini-2.0-flash-lite, gemini-3.1-flash-lite-preview, gemini-3-flash-preview), Qwen (Qwen3-VL-4B-Instruct, Qwen3.5-122B-A10B, qwen2.5:3b-instruct-q4_K_M, Qwen via Ollama), Google Gemma 4 E2B, Llama-4-Scout-17B (Azure AI Services), Unsloth, TRL, QLoRA/LoRA, vLLM, Ollama, llama.cpp, GGUF (Q4_K_S), OpenAI Python SDK (AsyncOpenAI), LangChain Core / LangChain OpenAI / LangChain Google GenAI, structured outputs (JSON schema, `strict=True`, Pydantic guided decoding)

**NLP:** RapidFuzz (ratio, partial_ratio, token_sort_ratio, token_set_ratio), difflib SequenceMatcher, n-gram keyword extraction and fuzzy classification, regex entity/date extraction, Arabic RTL bidirectional handling (arabic-reshaper, python-bidi), Azure AI Translator Text, deep-translator / GoogleTranslator, multilingual embeddings (paraphrase-multilingual-MiniLM-L12-v2, all-MiniLM-L6-v2), BM25 + semantic hybrid retrieval with RRF fusion, dateparser

**Computer Vision:** OpenCV, Pillow, YOLOv8 object detection and cropping, image quality metrics (Laplacian-variance blur, brightness, contrast, resolution gating), perceptual hashing, Error Level Analysis, spectral/frequency analysis, EXIF-aware preprocessing, ROI cropping and coordinate scaling, PDF rasterisation (150 DPI / 300 DPI / 2x scale)

**OCR & Document AI:** Azure AI Document Intelligence (custom extraction models, prebuilt-read, prebuilt-layout, QUERY_FIELDS), Google Cloud Vision, RapidOCR (rapidocr_onnxruntime), Tesseract / pytesseract, EasyOCR (evaluated), PaddleOCR, PyMuPDF (fitz) native text-layer extraction, pypdfium2, Wand/ImageMagick, Azure Document Intelligence Studio labelling

**Frameworks & APIs:** FastAPI, Uvicorn, Starlette/AnyIO worker pools, Pydantic v2 / Pydantic-Settings, APIRouter prefixes and tags, OpenAPI 3.0/3.1, Swagger UI, ReDoc, python-multipart, httpx, requests, CORS middleware, REST multipart/form-data contracts

**Data & Document Processing:** pandas, NumPy, PyMuPDF table and block-span extraction, JSON/JSONL/Parquet/Arrow, Hugging Face Datasets and Hub, stratified train/validation splitting, synthetic PDF training-data generation, coordinate-aware line reconstruction, append-only JSON audit stores, openpyxl, python-docx

**Cloud & Model Serving:** Microsoft Azure (Document Intelligence, Translator, AI Services, Container Apps), Google Cloud (Vision API, GenAI), Hugging Face Hub, vLLM, Ollama, llama.cpp, on-premise GPU/CPU inference hosts

**Tooling & UI:** Streamlit, PyQt5, Jinja2 templates, Jupyter Notebook, HashiCorp Vault (hvac KV v2), joblib, Matplotlib, pytest, httpx test client

**MLOps/Deployment:** Docker (Python 3.11-slim hardened images, non-root uvicorn), Nginx reverse proxy with WebSocket upgrade, Uvicorn multi-worker deployment, `.env`/BaseSettings configuration, model IDs externalised for hot-swap, hot-reloadable JSON rule stores, rotating file logging (TimedRotatingFileHandler, RotatingFileHandler), health/liveness endpoints, ThreadPoolExecutor and thread-local inference sessions, YAML-versioned training recipes, evaluation harnesses over ground-truth corpora

### Core Competencies

- Document intelligence architecture (OCR, key information extraction, classification, validation)
- Hybrid cloud/local extraction routing for cost, quota and data-residency constraints
- Vision-language and multimodal model integration with strict schema enforcement
- LLM fine-tuning, distillation and quantised on-premise serving
- Custom model training and evaluation (Azure DI custom models, YOLOv8, linear probes)
- Arabic/English bilingual NLP: RTL handling, reshaping, translation, entity resolution
- Synthetic training-data engineering for constrained cloud tenants
- FastAPI microservice design with typed contracts and uniform error envelopes
- Fuzzy-matching and rule-based classification with configurable, hot-reloadable rule sets
- Human-in-the-loop annotation tooling and data-quality auditing
- Concurrency engineering (thread pools, thread-local engines, async fan-out)
- Production hardening: secrets management, audit logging, health probes, containerisation
- Experiment-to-production traceability with source-cited verification
- Honest technical documentation of limitations, defects and unmeasured claims

### Key Achievements

- Delivered **15 document-intelligence and applied-AI systems** for a single insurance client spanning motor claims intake, KYC onboarding, underwriting decision support, claims pre-screening and voice data capture, covering UAE, Qatar, Oman and Kuwait document sets.
- Built a **local Arabic-aware police-report parser that extracts vehicle-party, driver, injury and property-damage tables in ~10 milliseconds at zero cloud OCR cost**, replacing per-page Azure Document Intelligence billing for high-volume MOI accident reports. [doha_ocr]
- Engineered a **synthetic training-data pipeline producing 300+ balanced samples** (1–7 vehicle layouts, cross-page injury-table splits, scan-style raster twins) to overcome a regional Azure tenant limited to template-only build mode, after rewriting the generator from v1/v2 to v3 to survive mixed Arabic encodings. [doha_ocr]
- Distilled a **Qwen3.5-122B-A10B teacher into a Gemma 4 E2B student**, processing **553 images in 18 min 41 s (2.03 s/image)** with Pydantic-guided decoding and zero JSON decoding failures, then deployed the merged model as a **4-bit GGUF (4,647,450,147 params, ~3.03 GB, n_ctx 65,536)** on an on-premise llama.cpp host. [llm_finetune]
- Diagnosed and fixed a **catastrophic memorisation failure** in an initial fine-tuning run through forensic review, redesigning the recipe (frozen vision layers, completion-only loss, constant schema prompt, 5 → 2 epochs, added validation split). [llm_finetune]
- Ran a **forensic label audit that found 12 of 30 flagged fields wrong (40% error rate on flagged fields)** plus a day/month swap, unit contamination and duplicate captures — instituting a mandatory human-in-the-loop quality gate before any training. [llm_finetune]
- Built a **Streamlit annotation dashboard used by three reviewers to verify 664 UAE and 296 Doha records with zero data loss**, with real-time diffs, `edited_by`/`edited_fields` provenance and save-before-navigate. [llm_finetune]
- Achieved **born-digital trade-licence extraction in 65–130 ms and worst-case 14-page raster bundles in 12–13.5 s**, with all **16 corpus documents inside a 20-second operational budget**, on CPU-only hardware with no paid APIs. [trade_license]
- Shipped an **OCR-free document classifier** using a frozen DINOv2-base 768-d embedding and a **19,442-byte LogisticRegression artefact** over six document classes, against a documented **7–10 second OCR baseline**. [dl_doc_classification]
- Designed a **YOLO-segmented, multi-engine OCR hub** routing **14 document-type mappings** (13 distinct extractor URLs) to downstream microservices, with YOLO cropping timed at **0.125 s per image and 1.074 s per PDF page** in stored PoC notebook cells (PoC measurements, not production benchmarks). [multidoc_ocr, textmatch_classification]
- Delivered a **six-endpoint bilingual UAE identity extraction service** chaining YOLOv8 cropping, Azure prebuilt-read OCR and a self-hosted Qwen SLM, with observed **end-to-end latencies of 5.07–19.09 s** captured in an integration test log. [slm_ocr]
- Architected a **ten-route UAE claims extraction API integrating ten custom-trained Azure Document Intelligence models** across three police jurisdictions and six card layouts, with insurer entity resolution against **42 master records** and structured 400/404/422/502 error semantics. [uae_ocr]

### Project Highlights

**DOHA OCR AI (FastAPI / Azure Document Intelligence / PyMuPDF)**
Headless microservice converting Qatar driving licences, residency permits, vehicle registrations, doctor licences and MOI traffic accident reports into structured JSON across six REST endpoints. Dual-path design sends card documents to four Azure DI custom models while MOI PDFs are parsed locally in ~10 ms by an Arabic-aware PyMuPDF/regex parser with RTL wrap correction, backed by a synthetic training-data pipeline of 300+ samples.
*Stack:* Python, FastAPI, Uvicorn, PyMuPDF, Azure Document Intelligence, Azure AI Translator, arabic-reshaper, regex, Jupyter.

**UAE OCR API — Query-Based Extraction (FastAPI / Azure Document Intelligence)**
Stateless service extracting up to 11 fields per document side from UAE driving licences, Emirates IDs, vehicle Mulkiya and DET trade licences using zero-shot QUERY_FIELDS on Azure prebuilt-layout — no custom model training or annotation. Ten declared routes (root plus health, and eight `POST /api/v1/extract/*` endpoints) with typed Pydantic v2 contracts, English→Arabic enrichment behind an LRU cache, and a dynamic query endpoint for arbitrary documents.
*Stack:* Python, FastAPI, Pydantic v2, Pydantic-Settings, Azure Document Intelligence, Azure AI Translation, Uvicorn, pytest, httpx.

**SLM AI OCR (FastAPI / YOLOv8 / Azure Document Intelligence / Qwen LLM)**
Seven-phase asynchronous pipeline turning UAE identity and vehicle document images or multi-page PDFs into bilingual JSON across six endpoints. Combines pypdfium2 rendering, a custom YOLOv8 card detector, Azure prebuilt-read OCR, anchor validation, agglomerative line clustering and schema-constrained extraction on a self-hosted Qwen model, with Vault-backed Basic Auth and rotating latency logs.
*Stack:* Python, FastAPI, YOLOv8/Ultralytics, OpenCV, pypdfium2, Azure Document Intelligence, Ollama + Qwen, AsyncOpenAI, deep-translator, HashiCorp Vault (hvac).

**Claim Duplication Detector API**
Stateless FastAPI service that ingests multi-PDF medical claim batches, renders every page at 2x with PyMuPDF, and returns per-page Unique / Blank / Duplicate verdicts with an aggregated summary and audit log. Cross-document perceptual hashing catches re-scans and re-exports that byte comparison misses; a Streamlit reviewer UI adds AI-vs-human classification and forensic fallbacks.
*Stack:* Python, FastAPI, PyMuPDF, OpenCV, ImageHash (pHash), Pillow, PyTorch, Hugging Face Transformers, Streamlit, Uvicorn.

**Document Classification Model (Deep Learning)**
Offline, OCR-free classifier for UAE motor-claims intake: a frozen DINOv2-base vision transformer produces 768-d L2-normalised CLS embeddings feeding a LogisticRegression head over six document classes. Single `POST /classify` endpoint handles images and multi-page PDFs page by page; retraining is one call and ships a 19,442-byte artefact while the 346 MB backbone never changes.
*Stack:* Python, FastAPI, PyTorch, Hugging Face Transformers, DINOv2, scikit-learn, joblib, PyMuPDF, Pillow, Uvicorn.

**KYC Document Validation System (FastAPI / RapidOCR / DeepFace / OpenCV / Streamlit)**
Identity-document verification for GCC onboarding, exposing one polymorphic `/process` endpoint that dispatches to five checks: OCR keyword compliance, expiry detection, resolution gating, ArcFace facial comparison and operator-drawn ROI OCR. Three interchangeable front-ends (Jinja2 canvas UI, Streamlit, PyQt5) sit over one CPU-only, zero-external-dependency processing layer.
*Stack:* Python, FastAPI, RapidOCR (ONNX Runtime), DeepFace/ArcFace, OpenCV, RapidFuzz, Streamlit, PyQt5, Jinja2, Uvicorn.

**TextMatch Label Trainer (Streamlit UI)**
No-code admin console letting operations staff author, test and hot-deploy document-classification keyword sets per jurisdiction (UAE, DOHA, OMAN, KUWAIT). Automated n-gram keyword discovery with fuzzy clustering, a dual OCR toggle (RapidOCR vs Azure prebuilt-read), and an empirical acceptance gate that persists rules only when the average match score clears an operator-set threshold.
*Stack:* Python, Streamlit, RapidOCR, Azure Document Intelligence, RapidFuzz, PyMuPDF, Pillow, OpenCV, Nginx.

**Multi-doc All-in-one OCR: Classifier + Extractor**
Hub-and-spoke FastAPI service that segments, OCRs, classifies, validates and routes identity, vehicle and police-report documents to downstream extractor microservices in one REST call. YOLO cropping over 150-DPI page renders, thread-local RapidOCR sessions across a 10-worker pool, fuzzy region-scoped classification, OpenCV quality guardrails and an `.env` routing table of 14 document-type mappings.
*Stack:* Python, FastAPI, Ultralytics YOLO, PyMuPDF, RapidOCR, Azure Document Intelligence, RapidFuzz, OpenCV, ThreadPoolExecutor, Nginx, Uvicorn.

**P&C Clause Recommendation Engine**
Underwriting decision-support service that detects missing risk clauses in commercial property submissions via two parallel pipelines behind one JSON contract: FAISS semantic retrieval over MiniLM-embedded business activities with a hybrid recency × frequency score, and a Gemini pipeline constrained to a JSON schema. Results are tiered Highly Recommended / Review Required / Optional with per-clause confidence and rationale.
*Stack:* Python, FastAPI, SentenceTransformers (all-MiniLM-L6-v2), FAISS, Google Gemini, Pydantic, pandas, NumPy, Streamlit, Uvicorn.

**Multilanguage Speech-to-Text Voice Data Capture**
Streamlit prototype suite that captures browser microphone audio and sends the WAV bytes plus a JSON-schema prompt directly to Gemini in a single multimodal call — collapsing transcription and extraction, so multilanguage support needs no separate ASR stage. One app fills 13 vehicle-insurance fields into an editable table; another runs a memory-aware conversational assistant that asks only for missing fields.
*Stack:* Python, Streamlit, google-genai SDK, Gemini (3.1-flash-lite-preview, 2.0-flash-lite), pandas.

**Document Detection API (TextMatch Classification)**
Detection-only triage microservice that classifies, expiry-checks and quality-scores uploaded insurance documents before they reach underwriters or the heavier extraction suite. PyMuPDF 150-DPI rasterisation, YOLO segmentation, a pluggable OCR layer (local RapidOCR default, Azure prebuilt-read optional), region-scoped fuzzy keyword classification and an append-only audit trail, deployed behind Nginx.
*Stack:* Python, FastAPI, Ultralytics YOLO, PyMuPDF, RapidOCR, Azure Document Intelligence, RapidFuzz, OpenCV, Uvicorn, Nginx.

**Trade License Extractor**
CPU-only, open-source pipeline reading UAE trade-licence PDFs of any provenance — born-digital, rasterised, flatbed scan, vector-outline with zero fonts, multi-document bundles — across 9 authority templates and five emirates. Deterministic-first design with gated OCR, negative-label guards, and a guard-railed optional local LLM fallback; output carries per-field confidence, provenance and review flags with Finance and GI role views.
*Stack:* Python, PyMuPDF, Pydantic, RapidFuzz, Tesseract/RapidOCR, python-bidi, arabic-reshaper, dateparser, OpenCV, Ollama (qwen2.5:3b), FastAPI, fastembed/ONNX, BM25 + RRF.

**UAE OCR AI (FastAPI / Azure Document Intelligence)**
Ten-route claims extraction microservice covering police accident reports from three jurisdictions and six identity/vehicle card layouts, backed by ten custom-trained Azure DI models. Cost-optimised hybrid engine parses digital PDFs locally with PyMuPDF and uses a local Tesseract header pass to select the right Azure model and pre-filter pages before any billed call; ships as a hardened non-root Docker image.
*Stack:* Python 3.11, FastAPI, PyMuPDF, pytesseract/Tesseract, Azure Document Intelligence, Azure Translator, RapidFuzz, arabic-reshaper, python-bidi, Docker, Uvicorn.

**LLM Fine-tuning Platform (Unsloth / Gemma / Qwen / vLLM / llama.cpp / Streamlit)**
Complete VLM lifecycle for KYC OCR/KIE: a Qwen3.5-122B teacher on vLLM performs Pydantic-guided zero-shot extraction, a Streamlit HITL dashboard audits it with reviewer provenance, the corpus is versioned to Hugging Face Hub, Gemma 4 E2B is fine-tuned with 4-bit QLoRA via Unsloth, and the merged model ships as GGUF on an on-premise llama.cpp host with the same OpenAI-compatible contract as the teacher.
*Stack:* Python 3.11, Unsloth, TRL, PyTorch, Transformers, BitsAndBytes QLoRA, vLLM, llama.cpp, GGUF, Pydantic, Streamlit, Hugging Face Hub/Datasets, LangChain, OpenAI SDK.

**VL_version_OCR — Vision-Language OCR (FastAPI / YOLOv8 / PyPDFium2 / LangChain / VLMs)**
End-to-end multimodal document intelligence service exposing five country extraction endpoints and four police-report endpoints. YOLOv8 splits composite scans, then a pluggable async VLM engine with six execution modes (Qwen3-VL on local vLLM, Gemini, Azure Llama-4-Scout × AsyncOpenAI/LangChain) performs two-stage anchor classification and strict-schema extraction over 18+ document types; police reports use four bespoke parsers instead.
*Stack:* Python 3.11, FastAPI, Ultralytics YOLOv8, pypdfium2, PyMuPDF, LangChain, AsyncOpenAI, vLLM, Google GenAI, Azure AI Services, Google Cloud Vision, Wand/ImageMagick, arabic-reshaper, python-bidi, deep_translator.

### Resume Bullet Points

**Document AI & OCR**

1. Architected a dual-path document extraction service that routes identity and vehicle cards to four Azure Document Intelligence custom models while parsing MOI traffic accident PDFs locally with PyMuPDF, reaching **~10 ms per report at zero cloud OCR cost** across six REST endpoints and five document types. [doha_ocr]
2. Engineered an Arabic-aware police-report parser resolving RTL bidirectional wrap artefacts through purpose-built `transfer_window()`, `fix_number_wrap()` and `is_label()` helpers plus lookaround-guarded regex anchors, extracting incident metadata and vehicle-party, driver, injury and property-damage tables into nested JSON. [doha_ocr]
3. Built a zero-shot extraction API using Azure Document Intelligence `QUERY_FIELDS` on prebuilt-layout, covering seven UAE document sides plus arbitrary documents with **up to 11 typed fields per side** and eliminating custom model training, annotation datasets and weight redeployment. [query_ocr]
4. Delivered a dynamic ad-hoc extraction endpoint accepting caller-supplied field lists (invoices, contracts, customs documents) and returning `query_count`/`matched_count` so integrating systems can detect partial extraction. [query_ocr]
5. Architected a ten-route UAE motor-claims extraction API integrating **ten custom-trained Azure Document Intelligence models** across Sharjah, Abu Dhabi and Dubai police reports and six identity/vehicle card layouts, normalising three jurisdictions into a single JSON schema. [uae_ocr]
6. Optimised cloud OCR spend by detecting native PDF text layers and extracting locally with PyMuPDF, and by using a local Tesseract pass on the page-1 header to select between two Azure layout models and pre-filter scanned pages before any billed call. [uae_ocr]
7. Built a CPU-only trade-licence extraction pipeline covering **9 UAE authority templates across five emirates**, handling born-digital, rasterised, flatbed-scanned, vector-outline and multi-document-bundle PDFs with selective OCR gating and coordinate-aware line reconstruction. [trade_license]
8. Designed negative-label guards that prevent Chamber/DCCI numbers being captured as the Commercial Register number and Fax being captured as Mobile, derived from documented vision-LLM failures on the project corpus. [trade_license]
9. Unified segmentation, OCR, classification, validation and downstream extractor routing into a single REST call, dispatching crops to **14 document-type extractor mappings** with per-crop error isolation and 3.0 s / 30.0 s timeouts. [multidoc_ocr]

**Computer Vision**

10. Integrated a custom-weights YOLO document detector (`yolo-multipage-OCR-cls.pt`, imgsz 320, conf 0.25) over 150-DPI page renders to split multi-document sheets and multi-page PDFs into individual crops, with confidence-based full-frame fallback — timed in PoC notebook cells at **0.125 s per image and 1.074 s per PDF page** (PoC, not a production benchmark). [multidoc_ocr, textmatch_classification]
11. Engineered an OCR-free document classifier from a frozen DINOv2-base ViT producing 768-d L2-normalised CLS embeddings into a LogisticRegression head over six classes, shipping a **19,442-byte retrainable artefact** against a documented **7–10 second OCR baseline**. [dl_doc_classification]
12. Eliminated train/serve preprocessing skew by sharing one embedding function (EXIF transpose, RGB conversion, 256-resize/224-centre-crop, ImageNet normalisation) between the training script and the inference service, with automatic CPU/GPU selection and import-time fail-fast loading. [dl_doc_classification]
13. Implemented cross-document duplicate page detection with perceptual hashing over 2x-rendered pages against a request-scoped hash index, catching re-scanned and re-exported claim pages that byte-level or metadata comparison would miss. [claim_duplication]
14. Built a cheap-to-expensive detection cascade placing OpenCV blank-page gating ahead of hashing so blank pages never consume hash computation or pollute the match index. [claim_duplication]
15. Implemented pre-extraction image-quality guardrails in OpenCV — blur (Laplacian variance), darkness, over-exposure, contrast and resolution — returning actionable per-document feedback so poor uploads are rejected at intake rather than failing downstream. [textmatch_classification, multidoc_ocr]
16. Built an operator-driven ROI extraction path scaling canvas drag-selections to original image coordinates, cropping with OpenCV under zero-dimension and out-of-bounds guards, and OCRing the isolated field for KYC field-level validation. [kyc]

**LLM & Prompt Engineering**

17. Designed a two-stage vision-language pipeline that first classifies document type against per-country anchor rules, then synthesises a strict Pydantic/JSON schema (`strict=True`, `temperature=0`, `additionalProperties=False`) covering **18+ document types across four jurisdictions and six passport nationalities**. [vl_ocr]
18. Engineered a provider-agnostic async VLM engine with **six selectable execution modes** spanning on-premise Qwen3-VL-4B-Instruct on vLLM, Google Gemini and Azure Llama-4-Scout across both AsyncOpenAI and LangChain paradigms, keeping business logic decoupled from vendors. [vl_ocr]
19. Enforced deterministic small-language-model extraction with `temperature=0`, strict `json_schema` response formatting, `/no_think` reasoning suppression and pipe-delimited per-document field schemas, producing normalised snake_case JSON for underwriting and claims systems. [slm_ocr]
20. Built a fail-fast multi-token anchor validation gate that rejects mismatched document types before any LLM inference, preventing plausible-but-wrong records from entering enterprise databases and avoiding wasted inference. [slm_ocr]
21. Integrated Google Gemini behind a JSON-schema-constrained `AIRecommendation` contract for underwriting clause gap analysis, returning the same three-tier `{clause, confidence, reason}` shape as the empirical retrieval pipeline. [pnc_recommendation]
22. Collapsed transcription and extraction into a single multimodal Gemini call over browser-captured WAV bytes, delivering multilanguage voice-to-form capture for **13 vehicle-insurance fields** without a separate ASR or translation stage. [speech_to_text]
23. Designed a stateful conversational prompt that embeds the current memory dictionary as JSON each turn so the assistant asks only for missing fields, returning `{response, field values, completed}` with safe defaults and truthy-only incremental merge. [speech_to_text]
24. Built a guard-railed local LLM fallback (Ollama `qwen2.5:3b-instruct-q4_K_M` over httpx `/api/chat` with `format='json'`) restricted to unknown templates and low-confidence fields, with every LLM-sourced value surfaced in review flags. [trade_license]

**LLM Fine-tuning & Serving**

25. Fine-tuned Google Gemma 4 E2B with Unsloth 4-bit BitsAndBytes QLoRA (r=16, alpha=16, adamw_8bit, batch 2 × accumulation 4) on a versioned **564-train / 100-validation** stratified 85/15 Hugging Face dataset drawn from the **664 human-verified UAE records** (a further **296 Doha records** were audited in the same tool). [llm_finetune]
26. Built a teacher-distillation pipeline against Qwen3.5-122B-A10B on vLLM with Pydantic-guided decoding, processing **553 images in 18 min 41 s (2.03 s/image)** and eliminating the JSON syntax failures caused by the earlier regex fence-stripping approach. [llm_finetune]
27. Diagnosed a catastrophic memorisation failure through forensic root-cause analysis and redesigned the training recipe — freezing vision layers, switching to completion-only loss, fixing a constant schema prompt, adding an evaluation split and cutting epochs from 5 to 2. [llm_finetune]
28. Quantised the merged model to GGUF Q4_K_S (**4,647,450,147 parameters, ~3.03 GB, n_ctx 65,536, multimodal capability**) and deployed it on an on-premise llama.cpp server exposing the same OpenAI-compatible `/v1/chat/completions` contract as the teacher, verified live via `GET /v1/models`. [llm_finetune]
29. Delivered on-premise inference as a compliance decision, keeping customer identity PII on internal infrastructure in line with UAE PDPL and Qatar financial data handling standards while removing per-call OCR API costs. [llm_finetune]
30. Served a self-hosted Qwen model through Ollama's OpenAI-compatible endpoint as the entity-extraction stage of a document pipeline, keeping PII-bearing extraction on-premise while using Azure only for OCR — a hybrid the documentation credits with reduced cloud token consumption and improved data sovereignty. [slm_ocr]

**NLP & Classical ML**

31. Built dual-language insurer entity resolution using RapidFuzz token-sort and token-set scoring against **42 Arabic/English alias records**, emitting canonical internal insurer codes to enable inter-company subrogation recovery. [uae_ocr]
32. Implemented semantic clause retrieval with SentenceTransformers `all-MiniLM-L6-v2` embeddings in a FAISS `IndexFlatL2` index, using exact-match-first logic with a configurable Top-N similar-activity fallback for sparse or paraphrased risk classes. [pnc_recommendation]
33. Designed a hybrid recency × frequency scoring model over underwriting-year and clause-occurrence data on a 0–1 confidence scale, tiering clauses as Highly Recommended / Review Required / Optional with a rationale citing year and occurrence count for audit and peer review. [pnc_recommendation]
34. Engineered a configurable fuzzy n-gram keyword classifier (1–3 word n-grams, RapidFuzz `ratio`/`token_sort_ratio`) over hot-reloadable per-region JSON dictionaries, enabling new jurisdictions to be added without code changes or redeployment. [textmatch_classification, multidoc_ocr]
35. Built automated keyword discovery combining n-gram frequency extraction with length and cross-document filters and fuzzy clustering, replacing manual document inspection for classification rule authoring. [label_trainer]
36. Integrated Azure AI Translator and GoogleTranslator for bidirectional Arabic↔English normalisation of names, nationalities, occupations, employers, vehicle attributes and accident narratives, with reshaping and bidi handling preserving Arabic presentation forms for legal audit. [doha_ocr, uae_ocr, slm_ocr]

**API & System Design**

37. Designed uniform JSON success/error envelopes and validation gates so malformed files, empty payloads, unreadable scans and cloud timeouts never surface as unhandled HTTP 500s to downstream claims pipelines. [doha_ocr, slm_ocr]
38. Built typed API contracts in Pydantic v2 with a standard response envelope, strict multipart validation (extension whitelist, empty-file detection) and explicit 400/415/422/500 semantics across **ten declared routes**, eight of them extraction endpoints under a versioned `/api/v1` namespace. [query_ocr]
39. Engineered concurrency for blocking I/O in ASGI services using synchronous handlers on Starlette's AnyIO worker pool, `ThreadPoolExecutor` fan-out and `asyncio.gather`, keeping Azure round-trips and PDF parsing off the event loop. [doha_ocr, uae_ocr, vl_ocr]
40. Designed a detection-only triage microservice deliberately decoupled from the extraction suite so document classification, expiry checking and quality scoring scale independently under burst traffic. [textmatch_classification]
41. Built a single polymorphic validation endpoint dispatching on a `type` form field to five in-process checks (keyword compliance, expiry, resolution, face verification, ROI OCR), each with a defined JSON response schema. [kyc]

**MLOps, Deployment & Tooling**

42. Containerised a document-intelligence service as a hardened Python 3.11-slim image with bundled Tesseract, OpenCV/PyMuPDF runtime dependencies and a non-root uvicorn process, targeting on-premise servers, Azure Container Apps and Kubernetes. [uae_ocr]
43. Deployed FastAPI services behind Nginx reverse proxies with correct `root_path` mounting and health/liveness endpoints for orchestrator health checks and monitoring, and a Streamlit admin console behind the same gateway with WebSocket upgrade support. [textmatch_classification, multidoc_ocr, label_trainer]
44. Eliminated plaintext secrets by loading Basic-Auth and API credentials at runtime from HashiCorp Vault (hvac KV v2), and added per-request latency audit logging with daily rotation and 24-day retention. [slm_ocr]
45. Designed file-based hot-reload configuration so accepted classification rules land in the same per-region JSON store the production FastAPI pipelines read, making a rule change live with no restart or deployment cycle. [label_trainer]
46. Built a no-code Streamlit admin console with an empirical acceptance gate that scores candidate keyword sets against real uploaded samples and persists them only above an operator-set threshold, preventing weak rules reaching production. [label_trainer]
47. Built a Streamlit human-in-the-loop annotation dashboard with real-time field diffs, `edited_by`/`edited_fields` audit provenance and save-before-navigate, used by three reviewers across **664 UAE and 296 Doha records with zero data loss**. [llm_finetune]
48. Externalised model IDs, extractor routes and thresholds to `.env` and YAML recipes so models and downstream endpoints swap without code changes, and versioned training recipes across two documented iterations. [doha_ocr, multidoc_ocr, llm_finetune]
49. Built an evaluation harness over a ground-truth corpus reused across two extraction pipelines, and measured per-document-class latency against a **20-second operational budget**, with all **16 corpus documents** inside it. [trade_license]
50. Engineered thread-local ONNX RapidOCR sessions so a 10-worker thread pool runs without mutex serialisation during high-volume ingestion, with Azure Document Intelligence selectable per request as an alternative backend. [multidoc_ocr, textmatch_classification]
51. Produced source-cited technical documentation verifying every constant, endpoint and error behaviour against the live service — including honestly labelling unexercised branches, known defects and unmeasured claims. [dl_doc_classification, and across the portfolio]

### Metric Honesty Note

Only some of the 15 project documents record quantified figures. Before publishing this CV, add your own measured numbers where the "no metrics" column applies — and do not let a reviewer read a scope count (endpoints, fields, document types) as a performance result.

**Projects with quantified figures available (use verbatim):**

| Project | Figures recorded in the documentation |
|---|---|
| doha_ocr | ~10 ms local MOI parse; 3.4–3.8 s vs 7–11 s scanned-PDF POC at 150 DPI; ~15 s Chrome GUI OCR POC; 300+ synthetic samples; 1–7 vehicle layouts; 30-PDF corpus check; 8/14/31/7 fields per card endpoint; 5 document types over 6 endpoints; 2 recommended Uvicorn workers |
| slm_ocr | Per-endpoint end-to-end latency from a 31 Mar 2026 integration test log: license_front 7.00–9.98 s; license_back 5.07–10.54 s; residence_front 8.27–13.66 s; residence_back 6.62–19.09 s; vehicle_front 7.71–8.86 s; vehicle_back 5.41–8.05 s. Design intent "within 10 secs" is a target, not a result |
| claim_duplication | One recorded log run: 4 files, 9 pages, 1 blank, 3 duplicates, ~7 s span; README example (2 documents, 4 pages, 1 blank, 1 duplicate); 8 verified constants; 4 Uvicorn workers |
| dl_doc_classification | 7–10 s OCR baseline being replaced; 1.38–2.35 images/s CPU embedding throughput during the training pass; 19,442-byte classifier artefact; 346,345,912-byte backbone; 768-d embeddings, 6 classes; captured confidences 0.9458 and 0.7935 |
| multidoc_ocr | 0.125 s YOLO cropping per image; 1.074 s per PDF page; POC fuzzy label scores 83.3% / 66.7%; 14 document-type mappings over 13 URLs; 71 POC notebook cells (all POC-only, not production benchmarks) |
| textmatch_classification | *(its document declares `metrics_available: false` — everything below is a PoC output or a configuration constant, never a production result)* Same 0.125 s / 1.074 s YOLO PoC timings; 83.3% / 66.7% PoC fuzzy scores; thresholds (cutoff 80, 65%, 90%); quality thresholds (Laplacian 80, brightness 50/220, std dev 30, 500,000 px); example response confidence 94 |
| trade_license | Under 10 s headline claim (conflicts with its own 12–13.5 s worst case); 65–130 ms born-digital pages; 6–10.7 s single-page OCR; 12–13.5 s for 11- and 14-page raster bundles; all 16 corpus documents inside a 20-second budget; 9 authority templates; 15-document zero-manual-entry claim |
| llm_finetune | 553 images in 18 min 41 s (2.03 s/image); 664 UAE + 296 Doha records reviewed with zero data loss; 564/100 train/validation split; 12 of 30 flagged labels wrong (40% error rate on flagged fields) and 2 duplicate images; GGUF 4,647,450,147 params / 3,028,113,548 bytes / n_ctx 65,536; LoRA r=16, alpha=16, lr 1e-4, effective batch 8, 5 → 2 epochs; 96% null legacy `resident_back` fields |

**Projects with NO quantified metrics in their documentation (`metrics_available: false`) — add your own before publishing:**

- **query_ocr** — no accuracy, throughput, cost or ROI figures; the document's own "high-accuracy" wording and "hours or days to sub-second" claim are unmeasured. Only scope facts are safe (10 routes, up to 11 fields per side, LRU cache size 1024).
- **kyc** — the document states verbatim that no benchmark results, log files or metrics exist. Thresholds (RapidFuzz 70, 65% keyword pass, 640×480 gate) are configuration constants, not results.
- **label_trainer** — no operational hours saved, ROI or throughput; no false-acceptance rates or OCR accuracy deltas published from the four calibration trials.
- **pnc_recommendation** — no accuracy, precision/recall, latency, adoption or cost figures; "hours to seconds" and "sub-millisecond" FAISS lookups are unmeasured claims in the source.
- **speech_to_text** — no throughput, accuracy, cost or processing-time figures; no metrics, benchmarks, logs or test results exist. Scope facts only (13 fields, 3 apps, each under 110 lines).
- **uae_ocr** — no accuracy, throughput, latency or cost-savings figures. Every number in the source is a configuration constant or scope count (10 routes, 10 models, 42 insurer records, 50% anchor gate, 150 DPI, 6 worker threads).
- **vl_ocr** — no accuracy, throughput, latency or cost figures. All numbers are constants or artefact sizes (conf 0.5 / IoU 0.4, 1536×1536, 300 DPI, 54.5 MB weights, 214 notebook cells, 18+ document types, 9 functional endpoints).

**Portfolio-wide cautions:**

- Several documents explicitly state that field-level exact-match accuracy, production throughput (documents/minute), labour-cost reduction and financial ROI are "Not specified in the project." Do not convert scope counts into performance claims.
- POC notebook timings (multidoc_ocr, textmatch_classification, doha_ocr) are stored cell outputs, not production benchmarks — label them as such if used.
- Thresholds such as the 65%/90% classification cutoffs, the RapidFuzz 80/70 cuts, the pHash Hamming distance of 5 and the 75% acceptance default were largely not calibrated against a labelled evaluation set. Presenting them as tuned values would overstate the evidence.
- `multidoc_ocr`, `claim_duplication`, `dl_doc_classification`, `trade_license`, `slm_ocr`, `doha_ocr` and `llm_finetune` are the projects whose documents set `metrics_available: true`; `textmatch_classification` records figures but flags `metrics_available: false`. The remaining seven record none at all.
- No document anywhere states team size, project duration, headcount saved, or a monetary figure. Do not add any.
- The strongest defensible numbers in this portfolio are the fine-tuning pipeline figures (llm_finetune), the trade-licence latency profile (trade_license), the SLM latency log (slm_ocr) and the ~10 ms local parse (doha_ocr).
