## A. Master Experience Summary

*Scope note: all fifteen projects were delivered for Qatar Insurance Group and are dated 2026. The source documents never state a job title, team size or years of experience — every ownership level below is **inferred** from documented scope, and no career length should be read into the project count. Figures are quoted only where a project document records them.*

### Comprehensive Experience Narrative

The candidate is an applied AI engineer who builds production document-intelligence systems for insurance operations. Across fifteen projects delivered for Qatar Insurance Group, the consistent output is the same shape of artefact: a service that takes a messy real-world input — a photographed Emirates ID, a bilingual Arabic/English police accident report, a scanned trade licence, a batch of claim PDFs, a spoken sentence — and returns validated, schema-stable JSON that downstream policy, underwriting and claims platforms can consume without a human retyping anything.

The recurring architectural spine is a FastAPI microservice wrapping a multi-stage extraction pipeline. That pattern appears in twelve of the fifteen projects: DOHA OCR AI, UAE OCR AI, UAE OCR API (Query-Based Extraction), SLM AI OCR, Multi-doc All-in-one OCR, Document Detection API (TextMatch Classification), Claim Duplication Detector, Document Classification Model, KYC Document Validation, P&C Clause Recommendation Engine, Trade License Extractor and VL_version_OCR. The internals vary, but a set of engineering habits recurs across the set: multipart `UploadFile` ingestion with extension and payload validation; a uniform success/error envelope (`messageType` `S`/`E` in DOHA OCR AI, UAE OCR API and SLM AI OCR) so malformed inputs never surface as unhandled 500s; auto-generated OpenAPI/Swagger docs; a `/health` probe; environment-driven configuration; and deployment behind Uvicorn — often with an Nginx reverse proxy and `root_path` mounting (`/detection`, `/ocr`), and in UAE OCR AI a hardened non-root Docker image targeting on-premise, Azure Container Apps or Kubernetes.

Inside those services, several pipeline motifs repeat. **YOLO-based region detection feeding OCR** appears in SLM AI OCR, Multi-doc All-in-one OCR, TextMatch Classification and VL_version_OCR: pages are rasterised with PyMuPDF or pypdfium2 (150 DPI, or 2x scale), a custom YOLOv8 detector crops individual cards from multi-document sheets, non-document crops are diverted, and only the crops reach the expensive recogniser. **Cost-aware hybrid routing** is a signature decision — DOHA OCR AI parses digital MOI accident PDFs locally with PyMuPDF in ~10 ms at zero cloud cost while sending card images to Azure Document Intelligence; UAE OCR AI runs a cheap local Tesseract pass purely as a router (selecting the Sharjah old/new model, filtering Dubai pages) before paying for any cloud call; Trade License Extractor gates OCR behind a character-count sufficiency test and an Arabic-sanity lexicon check so born-digital pages finish in 65–130 ms. **LLM structured-JSON extraction with hard guardrails** runs through SLM AI OCR (self-hosted Qwen via Ollama, `temperature=0`, strict `json_schema`, `/no_think`), Trade License Extractor (guard-railed local `qwen2.5:3b` fallback restricted to unknown templates and low-confidence fields), VL_version_OCR (dynamically synthesised strict Pydantic schemas, `additionalProperties=False`) and the Speech-to-Text suite (forced JSON MIME type on Gemini). **Deterministic-first design** recurs as an explicit stance: fuzzy RapidFuzz keyword classifiers with regional JSON dictionaries in TextMatch and Multi-doc OCR, negative-label anchor guards in Trade License Extractor that were designed after a trialled vision-LLM confused Chamber numbers with Register numbers, and anchor-keyword verification gates returning HTTP 422 in UAE OCR AI.

Bilingual Arabic/English handling threads through nearly every OCR project: RTL bidirectional wrap resolution in PDF text layers, `arabic-reshaper`/`python-bidi` for display forms kept separate from unshaped forms used for translation and lookup, right-to-left polygon sorting to rebuild vehicle strings, and Azure Translator or GoogleTranslator enrichment behind caches and timeouts.

Beyond extraction, the portfolio widens into adjacent ML work. The Document Classification Model uses a frozen DINOv2-base backbone with a scikit-learn logistic-regression probe to type documents without OCR, shipping a 19 KB retrainable artefact against a 346 MB fixed transformer. The Claim Duplication Detector applies perceptual hashing with Hamming-distance matching and OpenCV blank-page gating, plus a Hugging Face AI-vs-human image classifier and ELA/spectral forensics. The P&C Clause Recommendation Engine leaves documents entirely for underwriting decision support: SentenceTransformers embeddings in a FAISS index, a Recency × Frequency hybrid score, and a parallel Gemini pipeline behind one shared three-tier JSON contract. The LLM Fine-tuning project is the most complete lifecycle: a 122B teacher VLM under Pydantic-guided decoding produces labels, a Streamlit human-in-the-loop dashboard with reviewer provenance corrects them, the audited corpus is versioned to Hugging Face Hub, Gemma 4 E2B is fine-tuned with 4-bit QLoRA via Unsloth, and the merged model is quantised to GGUF and served on-premise through llama.cpp on the same OpenAI-compatible contract as the teacher.

Streamlit recurs as the tooling layer for humans: the TextMatch Label Trainer lets non-engineers author and empirically validate classification keyword sets that hot-reload into production FastAPI services; the KYC system ships three interchangeable front-ends (Jinja2 canvas UI, Streamlit, PyQt5) over one processing layer; the Speech-to-Text suite delivers three voice UX prototypes.

The trajectory the portfolio shows is a move from cloud-managed OCR services (Azure Document Intelligence custom and prebuilt models) toward hybrid and then self-hosted open-weight models — local RapidOCR, Ollama-served Qwen, on-premise vLLM, and finally a fine-tuned quantised student model — driven by documented concerns about per-page transaction cost, network latency, regional tenant constraints and PII data sovereignty. A second visible trajectory is engineering discipline: notebook proofs-of-concept traced cell-by-cell into named production functions, verified-constants tables citing file and line, forensic root-cause reports (the Run 1 catastrophic-memorisation investigation that redesigned the QLoRA recipe), and candid documentation of unresolved defects rather than silence. Metrics are the honest gap: most documents explicitly state that accuracy, throughput and ROI figures were not recorded, and the candidate's own write-ups say so rather than inventing them.

### Total Areas of Expertise

**Document AI / OCR extraction**
- Azure Document Intelligence custom & prebuilt models — DOHA OCR AI, UAE OCR AI, UAE OCR API (Query-Based), SLM AI OCR, Multi-doc All-in-one OCR, TextMatch Classification, TextMatch Label Trainer
- Local/open-source OCR — RapidOCR (KYC Document Validation, TextMatch Classification, TextMatch Label Trainer, Multi-doc All-in-one OCR), Tesseract (UAE OCR AI, Trade License Extractor), PaddleOCR (Trade License Extractor, named inconsistently in that document), Google Cloud Vision (VL_version_OCR police parsers)
- Native PDF text-layer parsing (PyMuPDF/fitz) — DOHA OCR AI, UAE OCR AI, Trade License Extractor, VL_version_OCR
- Zero-shot query-field extraction (no custom training) — UAE OCR API (Query-Based Extraction)
- Hybrid cost-aware cloud/local routing — DOHA OCR AI, UAE OCR AI, Trade License Extractor; local-default pluggable OCR to avoid cloud transaction cost — TextMatch Classification, Multi-doc All-in-one OCR

**Computer Vision**
- YOLOv8 document detection, cropping and non-document filtering — SLM AI OCR, Multi-doc All-in-one OCR, TextMatch Classification, VL_version_OCR
- Vision-transformer embeddings (DINOv2 CLS tokens, L2-normalised) — Document Classification Model
- Perceptual hashing, blank-page detection, ELA and spectral forensics — Claim Duplication Detector
- Facial biometric verification (DeepFace/ArcFace), ROI cropping, image-quality gating — KYC Document Validation
- OpenCV quality metrics (Laplacian blur, brightness, contrast, resolution) — Multi-doc All-in-one OCR, TextMatch Classification; resolution gating only — KYC Document Validation
- PDF rasterisation and DPI/scale tuning — DOHA OCR AI, TextMatch Classification, Multi-doc All-in-one OCR, VL_version_OCR, TextMatch Label Trainer, Claim Duplication Detector (2x render)

**LLM engineering (application layer)**
- Schema-constrained structured output (strict JSON schema, temperature 0, `additionalProperties=False`) — SLM AI OCR, VL_version_OCR, Trade License Extractor, Speech-to-Text, LLM Fine-tuning
- Multi-provider / pluggable model engines (Qwen, Gemini, Llama-4-Scout; AsyncOpenAI and LangChain paradigms) — VL_version_OCR
- Guard-railed LLM fallback restricted to low-confidence or unknown cases — Trade License Extractor
- Generative recommendation with schema-bound output — P&C Clause Recommendation Engine
- Multimodal single-call audio+prompt extraction — Multilanguage Speech-to-Text
- Prompt design: anchor/system prompts, memory-embedded stateful prompts, per-document field catalogues — SLM AI OCR, Speech-to-Text, LLM Fine-tuning, VL_version_OCR

**LLM fine-tuning & serving**
- Teacher–student distillation, dataset construction and Hugging Face Hub versioning — LLM Fine-tuning Platform
- 4-bit QLoRA/BitsAndBytes via Unsloth, LoRA hyperparameter design, failure diagnosis and recipe redesign — LLM Fine-tuning Platform
- GGUF quantisation and llama.cpp serving — LLM Fine-tuning Platform; self-hosted model serving — Ollama (SLM AI OCR, Trade License Extractor), vLLM (LLM Fine-tuning Platform, VL_version_OCR)

**NLP, classification & entity resolution**
- Fuzzy matching and n-gram keyword classification (RapidFuzz ratio/token_sort/partial_ratio) — TextMatch Classification, Multi-doc All-in-one OCR, TextMatch Label Trainer, KYC Document Validation, UAE OCR AI
- Arabic RTL/bidi handling, reshaping, encoding pitfalls — DOHA OCR AI, UAE OCR AI, Trade License Extractor, VL_version_OCR
- Machine translation orchestration — Azure Translator (DOHA OCR AI, UAE OCR API, UAE OCR AI), GoogleTranslator/deep-translator (SLM AI OCR, VL_version_OCR), with an LRU cache in UAE OCR API and a 3-second timeout with no retries in SLM AI OCR
- Bilingual entity resolution to canonical codes — UAE OCR AI (insurer master lookup)
- Regex/label-anchor deterministic extraction — DOHA OCR AI (Arabic-aware police-report parser), KYC Document Validation (expiry regex), UAE OCR AI (anchor-keyword verification); with negative-label guards — Trade License Extractor

**Speech**
- Browser audio capture to structured slot-filling, multi-turn voice dialogue, multilanguage adaptation — Multilanguage Speech-to-Text Voice Data Capture

**Recommendation, retrieval & similarity**
- SentenceTransformers embeddings + FAISS `IndexFlatL2`, exact-match-first with semantic fallback — P&C Clause Recommendation Engine
- Hybrid semantic/BM25 retrieval with RRF fusion and grounding checks — Trade License Extractor (RAG sub-package)
- Hybrid Recency × Frequency actuarial ranking with tiered confidence — P&C Clause Recommendation Engine
- Embedding-space kNN second opinion and pHash baselines — Document Classification Model

**Classical ML**
- Linear probe on frozen embeddings, stratified splits, reproducible artefacts — Document Classification Model
- Threshold calibration and empirical acceptance gating — TextMatch Label Trainer, TextMatch Classification

**API & system design**
- FastAPI service design with APIRouter prefixes/tags — DOHA OCR AI, UAE OCR AI, SLM AI OCR, Multi-doc All-in-one OCR, VL_version_OCR; custom OpenAPI schema overrides for binary array uploads — Multi-doc All-in-one OCR, TextMatch Classification; custom OpenAPI schema generation — Claim Duplication Detector
- Uniform response envelopes, explicit 400/404/415/422/500/502 semantics, per-item error isolation — UAE OCR AI, UAE OCR API, Multi-doc All-in-one OCR, DOHA OCR AI
- Concurrency: sync handlers on Starlette/AnyIO worker pools, ThreadPoolExecutor pools, thread-local ONNX sessions, asyncio.gather — DOHA OCR AI, UAE OCR AI, Multi-doc All-in-one OCR, TextMatch Classification, VL_version_OCR
- Hub-and-spoke orchestration and downstream microservice routing tables — Multi-doc All-in-one OCR
- Configuration-as-data and hot-reload without restarts — TextMatch Label Trainer, TextMatch Classification, Multi-doc All-in-one OCR
- Deployment: Uvicorn, Nginx reverse proxy, Docker (non-root, bundled native deps), HashiCorp Vault secrets — UAE OCR AI, SLM AI OCR, TextMatch Classification, Multi-doc All-in-one OCR

**UI tooling & human-in-the-loop**
- Streamlit admin/annotation/review interfaces — TextMatch Label Trainer, LLM Fine-tuning Platform, Claim Duplication Detector, KYC Document Validation, Speech-to-Text
- Multi-front-end delivery over one processing layer (HTML/Jinja2 canvas, Streamlit, PyQt5) — KYC Document Validation
- Reviewer provenance, diffing and auto-save safeguards — LLM Fine-tuning Platform

**Data engineering & MLOps practice**
- Synthetic training-data generation preserving Arabic typography and label alignment — DOHA OCR AI
- Ground-truth corpora and evaluation harnesses — Trade License Extractor (`values_match()` over a 16-document corpus), Document Classification Model (stratified 75/25 hold-out with per-class report), LLM Fine-tuning Platform (100-row validation split); note that in all three the harness exists but no accuracy figure is recorded in the documents
- Notebook-to-production traceability with cell-level and file/line citation — Multi-doc All-in-one OCR, TextMatch Classification, TextMatch Label Trainer, Trade License Extractor, VL_version_OCR
- Audit logging and append-only transaction stores — TextMatch Classification, Multi-doc All-in-one OCR, Claim Duplication Detector

### Industries / Domains Worked In

All fifteen projects were delivered for a single client, **Qatar Insurance Group (insurance)**, with document coverage spanning the GCC — UAE (Abu Dhabi, Dubai including JAFZA, Sharjah, Ajman, Umm Al Quwain), Qatar/Doha, Oman and Kuwait — plus international passports.

**Motor claims — FNOL, claim intimation, liability and subrogation.** DOHA OCR AI parses MOI traffic accident reports into vehicle-party, driver, injury and property-damage structures for FNOL intake. UAE OCR AI covers police accident reports from three emirates (Rafid Sharjah, Saaed Abu Dhabi, Dubai Police) and resolves free-text carrier names to canonical insurer codes for subrogation recovery. VL_version_OCR adds four police-authority parsers including Doha Traffic Police. Multi-doc All-in-one OCR routes police reports plus identity and vehicle documents to per-type extractors. SLM AI OCR and UAE OCR API serve retail motor claims and vehicle Mulkiya registration.

**KYC, identity verification and customer onboarding.** KYC Document Validation performs keyword compliance, expiry detection, image-quality gating and ArcFace facial verification across GCC licences, residency permits, vehicle registrations and six passport nationalities. UAE OCR API covers Emirates ID, driving licence and Mulkiya extraction for onboarding and policy issuance. The LLM Fine-tuning Platform is scoped explicitly to KYC identity and vehicle document verification, with on-premise inference chosen for UAE PDPL and Qatar financial data residency.

**Underwriting and automated policy issuance.** DOHA OCR AI feeds underwriting portals; SLM AI OCR targets automated underwriting platforms and fleet management portals; UAE OCR API supports automated underwriting and policy issuance.

**P&C commercial property underwriting — clause governance.** The P&C Clause Recommendation Engine detects missing risk clauses in commercial property submissions, tiering recommendations for underwriters and underwriting managers, aimed at reducing inter-underwriter variability and uninsured exposure gaps.

**Commercial/corporate onboarding and trade licensing.** Trade License Extractor processes UAE trade licences across five emirates for the Finance Department and General Insurance team, covering nine authority templates plus free-zone layouts.

**Medical claims — intake pre-screening and integrity.** Claim Duplication Detector serves the Medical claim department, classifying every page of a claim batch as Unique, Blank or Duplicate with cross-document perceptual matching, plus AI-generated/edited page flags and PDF metadata modification checks.

**Medical-practitioner credentialing.** DOHA OCR AI includes a doctor-licence extraction model and endpoint.

**Document intake, triage and classification (cross-functional).** Document Detection API (TextMatch Classification) provides detection-only triage ahead of extraction. Document Classification Model provides OCR-free document typing for motor claims intake. TextMatch Label Trainer gives operations staff no-code authoring of the classification rules those services consume, scoped per jurisdiction.

**Customer-facing digital channels and data capture.** DOHA OCR AI serves mobile/web claim-submission channels; UAE OCR AI names external customer-facing intimation portals as a consumer class; Multilanguage Speech-to-Text captures motor-claim data by voice in the customer's own language.

### Project Portfolio Table

| # | Project | Business function | Core technologies | Ownership level (inferred) | One-line description |
|---|---------|-------------------|-------------------|----------------------------|----------------------|
| 1 | DOHA OCR AI | Motor claims FNOL, identity verification, medical-practitioner credentialing | FastAPI, Azure Document Intelligence (4 custom models), Azure Translator, PyMuPDF, regex/RTL parser, synthetic data pipeline | Senior AI Engineer | Headless service converting Qatar licences, QIDs, vehicle registrations, doctor licences and MOI accident reports into JSON via a dual cloud/local path. |
| 2 | UAE OCR API — Query-Based Extraction | Onboarding/KYC, policy issuance, automated underwriting, claims registration | FastAPI, Pydantic v2, Azure DI prebuilt-layout QUERY_FIELDS, Azure Translation, LRU cache | AI Engineer | Zero-shot extraction service pulling typed bilingual fields from UAE IDs, Mulkiya and trade licences without any custom model training. |
| 3 | SLM AI OCR | Underwriting, retail motor claims, digital onboarding, fleet portals | FastAPI, YOLOv8, Azure DI prebuilt-read, self-hosted Qwen via Ollama, HashiCorp Vault, deep-translator | Senior AI Engineer | Seven-phase pipeline cropping UAE cards with YOLO, OCRing in the cloud and extracting entities with an on-prem SLM under strict JSON schema. |
| 4 | Claim Duplication Detector API | Medical claims intake pre-screening and document integrity | FastAPI, PyMuPDF, OpenCV, ImageHash pHash, PyTorch + Hugging Face classifier, Streamlit | AI Engineer | Page-level triage of claim PDF batches into Unique/Blank/Duplicate with cross-document perceptual hashing and AI-generated-image flags. |
| 5 | Document Classification Model (Deep Learning) — service title "Motor Doc Check AI" | Motor claims document intake typing | DINOv2-base (frozen), scikit-learn LogisticRegression, FastAPI, PyMuPDF, PyTorch | ML Engineer | Offline OCR-free classifier typing six document faces from frozen vision-transformer embeddings with a 19 KB retrainable head. |
| 6 | KYC Document Validation System | Customer onboarding / KYC identity verification | FastAPI, RapidOCR (ONNX), DeepFace ArcFace, OpenCV, RapidFuzz, Streamlit, PyQt5, Jinja2 | AI Engineer | Polymorphic validation service running keyword compliance, expiry, resolution, face-match and ROI OCR behind three interchangeable UIs. |
| 7 | TextMatch Label Trainer | Classification rule governance for document intake (ops-facing) | Streamlit, RapidOCR, Azure DI prebuilt-read, RapidFuzz, PyMuPDF, Nginx | AI Engineer | No-code admin tool letting operations staff author, empirically test and hot-deploy per-jurisdiction classification keyword sets. |
| 8 | Multi-doc All-in-one OCR: Classifier + Extractor | Policy, underwriting and claims document intake orchestration | FastAPI, Ultralytics YOLO, RapidOCR (thread-local ONNX), Azure DI, RapidFuzz, PyMuPDF, ThreadPoolExecutor, Nginx | Senior AI Engineer | Hub service that segments, OCRs, classifies, quality-checks and routes each document crop to the right downstream extractor microservice. |
| 9 | P&C Clause Recommendation Engine | Commercial property underwriting — clause gap analysis | FastAPI, SentenceTransformers all-MiniLM-L6-v2, FAISS, Google Gemini (schema-constrained), Streamlit, Pandas | Senior AI Engineer | Underwriting decision-support service pairing FAISS historical retrieval with Gemini reasoning to flag missing policy clauses in three tiers. |
| 10 | Multilanguage Speech-to-Text Voice Data Capture | Motor claim data capture / customer-facing intake | Streamlit, Google Gemini via google-genai (forced JSON), pandas | AI Engineer | Voice-to-form prototypes sending browser audio straight to a multimodal LLM to fill thirteen motor-claim fields in any spoken language. |
| 11 | Document Detection API (TextMatch Classification) | Document intake triage ahead of extraction | FastAPI, Ultralytics YOLO, RapidOCR, Azure DI prebuilt-read, RapidFuzz, OpenCV, PyMuPDF, Nginx | AI Engineer | Detection-only microservice classifying, expiry-checking and quality-scoring uploads so bad documents never reach extractors or underwriters. |
| 12 | Trade License Extractor | Trade-licence intake for Finance and General Insurance | PyMuPDF, Tesseract/RapidOCR, Pydantic, RapidFuzz, Ollama qwen2.5:3b, fastembed/BM25 RAG, FastAPI | Senior AI Engineer | CPU-only deterministic-first extractor covering nine UAE authority templates with a guard-railed local LLM fallback and role-scoped output views. |
| 13 | UAE OCR AI | Motor claims intimation, liability determination, subrogation, underwriting | FastAPI, 10 custom Azure DI models, Azure Translator, Tesseract router, PyMuPDF, RapidFuzz, Docker | Senior AI Engineer | Ten-route microservice normalising three police jurisdictions and six card layouts into one JSON schema with cost-avoiding local parsing. |
| 14 | LLM Fine-tuning Platform (Unsloth / Gemma 4 / Qwen3.5 / vLLM / llama.cpp) | KYC identity and vehicle document verification | Unsloth QLoRA, Gemma 4 E2B, Qwen3.5-122B on vLLM, Streamlit, Hugging Face Hub, llama.cpp GGUF, Pydantic | Senior AI / ML Engineer | Full teacher-to-student VLM lifecycle: guided-decoding distillation, human-in-the-loop audit, QLoRA fine-tune and on-prem quantised serving. |
| 15 | VL_version_OCR — Vision-Language OCR | Motor underwriting, claims intake, policy administration across GCC | FastAPI, YOLOv8, pypdfium2, LangChain, Qwen3-VL/Gemini/Llama-4-Scout, Google Cloud Vision, PyMuPDF, Wand | Senior AI Engineer | Provider-agnostic VLM service segmenting composite scans and extracting strict-schema JSON for 18+ document types plus four police-report parsers. |
