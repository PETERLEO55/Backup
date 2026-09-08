# Professional Profile Report — Peter Leo

**Prepared:** 7 September 2026
**Source:** 15 project documentation files (client: Qatar Insurance Group)
**Method:** each document was analyzed by an independent analyst, then challenged by an adversarial fact-checker and a completeness critic against the source. Cross-project sections were synthesized from the verified analyses, fact-checked again, and given a final consistency pass.
**Labels:** anything marked *(inferred)* is a judgement from project scope, not a statement in the documents. Figures appear only where a document states them.
# Part III — Consolidated Professional Profile Report

## Consolidated Professional Profile Report

*Scope note: everything below is derived from fifteen project documents delivered for Qatar Insurance Group and dated 2026. The documents never state a job title, team size, employment dates or years of experience; role and seniority are **inferred** from documented scope. Where a document states that accuracy, throughput, cost or ROI figures were not recorded, that is said plainly rather than filled in.*

---

### Who I Am

I am an applied AI engineer who builds document intelligence systems for insurance. Across fifteen projects for Qatar Insurance Group the output has consistently been the same shape of artefact: a service that takes a messy real-world input — a photographed Emirates ID, a bilingual Arabic/English traffic police accident report, a scanned trade licence, a batch of medical claim PDFs, a spoken sentence in whatever language the customer speaks — and returns validated, schema-stable JSON that policy, underwriting and claims platforms can consume without anyone retyping anything.

The recurring shape is a FastAPI microservice wrapping a multi-stage extraction pipeline: ingest and validate at the edge, segment, recognise, classify, gate, extract, normalise, and hand back a uniform envelope. Twelve of the fifteen projects are built that way. Inside them, a set of habits repeats because they earn their place — multipart `UploadFile` intake with extension and payload checks, a uniform success/error envelope so malformed inputs never surface as an unhandled 500, `/health` probes, environment-driven configuration, and concurrency handled deliberately rather than by default (synchronous handlers offloaded to Starlette's AnyIO pool for blocking I/O, bounded thread pools for CPU-bound parsing, thread-local ONNX sessions to kill mutex contention).

What distinguishes the work is not model choice, it is routing judgement. I default to the cheapest mechanism that meets the bar and escalate only on evidence. Digital MOI accident PDFs are parsed locally with PyMuPDF in roughly 10 ms at zero cloud OCR cost while card images go to Azure Document Intelligence. A cheap local Tesseract pass on a page header is used purely as a *router* — picking the correct Azure model and filtering pages — so it never reaches the output but avoids calls I do not need to pay for. Trade licences gate OCR behind a character-count sufficiency test and an Arabic-sanity lexicon check, so born-digital pages finish in 65–130 ms. LLMs are fenced in: strict JSON schemas, `temperature=0`, `additionalProperties=False`, anchor gates that reject the wrong document *before* inference, and fallbacks confined to unknown templates and low-confidence fields.

Two other things define me. Arabic and bilingual correctness — RTL bidirectional wrap resolution in PDF text layers, keeping unshaped strings for lookup and reshaped ones for display, right-to-left polygon sorting — because that is where most GCC document pipelines quietly break. And documentation honesty: I record unexercised branches, drifted thresholds, schema typos already baked into contracts, and the fact that most of these systems have no measured accuracy. Measurement is the gap I am pushing hardest on now.

---

### What I Have Built

**Document AI & OCR extraction.** Eleven of the fifteen projects are document-intelligence systems covering essentially the full IDP surface. *DOHA OCR AI* is a six-endpoint headless service with a dual path: four Azure Document Intelligence custom models for Qatar driving licences, residency permits, vehicle registrations and doctor licences, alongside a local PyMuPDF/regex parser that extracts vehicle-party, driver, injury and property-damage tables from MOI accident PDFs in ~10 ms at zero cloud cost. *UAE OCR AI* is a ten-route service over ten custom-trained Azure DI models spanning three police jurisdictions (Rafid Sharjah, Saaed Abu Dhabi, Dubai Police) and six card layouts, normalised into one schema, with an `is_scanned_pdf` local/cloud split and a Tesseract header router. *UAE OCR API — Query-Based Extraction* takes the opposite route: zero-shot `QUERY_FIELDS` on Azure `prebuilt-layout` with no annotation or training, per-document query presets and a dynamic endpoint accepting caller-supplied field lists. *Trade License Extractor* is CPU-only and open-source end to end, covering nine UAE authority templates across five emirates and every PDF provenance (born-digital, print-to-PDF raster, flatbed scan, vector-outline with zero fonts, multi-document bundles), with selective OCR gating, coordinate-aware line reconstruction and Finance/GI role projections. *Multi-doc All-in-one OCR* is the hub — segment, OCR, classify, quality-gate, then dispatch each crop to one of fourteen extractor mappings over a `.env` routing table with 3 s/30 s timeouts and per-crop error isolation. *Document Detection API (TextMatch Classification)* is the deliberately decoupled triage tier: classification, expiry and quality scoring only, scaling independently of extraction.

**Computer vision.** A custom-weights YOLOv8 detector (`yolo-multipage-OCR-cls.pt`) is reused as a pre-OCR splitter across *SLM AI OCR*, *Multi-doc All-in-one OCR*, *Document Detection API* and *VL_version_OCR*: pages rasterised at 150 DPI or 2× scale, cards cropped, non-document classes diverted, full-frame fallback when nothing clears confidence. Thresholds were compared empirically in *VL_version_OCR* (`conf=0.5`/`IoU=0.4` chosen over `iou=0.9`) and run at `imgsz=320`/`conf=0.25` in the CPU services. *Document Classification Model (Deep Learning)* — the service titled "Motor Doc Check AI" — replaces OCR entirely for document typing: a frozen DINOv2-base backbone producing 768-d L2-normalised CLS embeddings into a scikit-learn LogisticRegression head, shipping a 19,442-byte retrainable artefact against a 346 MB backbone that never changes, with one shared embedding function eliminating train/serve skew. *Claim Duplication Detector API* applies perceptual hashing (pHash, Hamming < 5) over 2×-rendered pages against a request-scoped index behind an OpenCV blank-page gate (threshold 240, white ratio > 0.99), plus a Hugging Face AI-vs-human classifier and ELA/spectral forensics in its Streamlit reviewer UI. *KYC Document Validation System* adds ArcFace face verification and an operator-drawn ROI pipeline with canvas-to-image coordinate scaling. OpenCV quality gating — Laplacian-variance blur, brightness bounds, contrast std-dev, resolution — runs ahead of extraction in the triage services.

**LLM engineering & prompt design.** The consistent stance is treating an LLM as an unreliable component to be constrained. *VL_version_OCR* runs anchor classification, then a strict Pydantic/JSON schema synthesised per document type (`strict=True`, `temperature=0`, `additionalProperties=False`) over 18+ document types across four jurisdictions and six passport nationalities, behind a provider-agnostic engine with six execution modes (Qwen3-VL on local vLLM, Gemini, Azure Llama-4-Scout × AsyncOpenAI/LangChain). *SLM AI OCR* enforces determinism with `temperature=0`, strict `json_schema`, `/no_think` suppression and pipe-delimited field schemas behind a multi-token anchor gate that rejects mismatched documents before inference. *Trade License Extractor* keeps the LLM as a guard-railed fallback only. *Multilanguage Speech-to-Text* collapses transcription and extraction into one multimodal Gemini call, with a conversational variant that serialises the current memory dict into the system prompt each turn. *P&C Clause Recommendation Engine* binds Gemini to an `AIRecommendation` schema matching the empirical pipeline's shape.

**LLM fine-tuning & serving.** The *LLM Fine-tuning Platform* is the deepest single piece: a Qwen3.5-122B-A10B teacher on vLLM with Pydantic guided decoding labelled 553 images in 18 min 41 s (2.03 s/image); a Streamlit human-in-the-loop dashboard with reviewer provenance and save-before-navigate corrected 664 UAE and 296 Doha records with zero data loss; the corpus was restructured into Unsloth `messages` format, split 85/15 into 564/100 and versioned on Hugging Face Hub; Gemma 4 E2B was fine-tuned with 4-bit BitsAndBytes QLoRA (r=16, alpha=16, adamw_8bit, effective batch 8); and the merged model was exported to GGUF Q4_K_S (~3 GB, 4.6B params, n_ctx 65,536) and served on-premise via llama.cpp on the same OpenAI-compatible contract as the teacher. Self-hosted serving also appears through Ollama in *SLM AI OCR* and *Trade License Extractor*.

**NLP, classification & similarity.** RapidFuzz is used with real discrimination — `ratio` for single tokens and `token_sort_ratio` for phrases at cutoff 80 in the TextMatch services, `partial_ratio` at 70 for OCR pre-correction in *KYC*, token-sort/token-set at 85 for insurer entity resolution against 42 alias records in *UAE OCR AI*, emitting the canonical codes subrogation recovery needs. Arabic work spans RTL wrap helpers (`transfer_window()`, `fix_number_wrap()`, `is_label()`) in *DOHA OCR AI*, reshaping via arabic-reshaper/python-bidi, Azure Translator enrichment behind a 1024-entry LRU cache in *UAE OCR API*, and GoogleTranslator bounded by a 3-second timeout in *SLM AI OCR*. *P&C Clause Recommendation Engine* leaves documents entirely: MiniLM embeddings in FAISS `IndexFlatL2`, exact-match-first with Top-N fallback, and a recency × frequency score tiered into Highly Recommended / Review Required / Optional. *Trade License Extractor* adds a hybrid semantic/BM25 RAG sub-package with RRF fusion and a grounding check.

**Speech.** *Multilanguage Speech-to-Text Voice Data Capture* delivers three voice UX prototypes that send browser-captured WAV bytes straight to a multimodal Gemini call, filling thirteen motor-claim fields with no separate ASR or translation stage — which is what makes multilanguage support free of per-language components.

**Tooling & annotation.** *TextMatch Label Trainer* is a no-code Streamlit console letting operations staff author, empirically test and hot-deploy per-jurisdiction classification keyword sets, with automated n-gram keyword discovery and an acceptance gate that refuses to persist rules scoring below threshold on real samples. *KYC Document Validation System* ships three interchangeable front-ends over one processing layer. The fine-tuning annotation dashboard added diffs, `edited_by`/`edited_fields` provenance and unique widget keying.

**API & system design.** APIRouter prefixes and tags, `root_path` mounting behind Nginx, custom OpenAPI overrides so binary array uploads render correctly, an `OCRError` taxonomy mapped to 400/404/422/500/502, per-item error isolation, configuration-as-data with hot reload, hub-and-spoke dispatch, HashiCorp Vault credential loading, rotating audit logs, and a hardened non-root Python 3.11-slim Docker image targeting on-premise, Azure Container Apps and Kubernetes.

---

### Signature Systems

**1. LLM Fine-tuning Platform (Unsloth / Gemma 4 E2B / Qwen3.5-122B / vLLM / llama.cpp).**
KYC identity and vehicle documents could not leave internal infrastructure under UAE PDPL and Qatar data-handling rules, and per-call cloud OCR was a recurring cost. The answer was a five-stage lifecycle: a 122B teacher on vLLM with Pydantic-guided decoding produced labels (553 images in 18 min 41 s), a Streamlit HITL dashboard corrected them with provenance, the audited corpus was versioned to the Hub as a stratified 564/100 split, Gemma 4 E2B was QLoRA fine-tuned, and the merged model was quantised to GGUF Q4_K_S and served on-premise via llama.cpp on the teacher's OpenAI-compatible contract. The hardest decision was the response to Run 1's catastrophic memorisation — vision layers trained but served against a stock `mmproj`, five epochs, no validation split — which I diagnosed forensically and fixed by freezing vision layers, completion-only loss, a constant schema prompt and an eval split. Honest limitation: the v2 evaluation result was never recorded, so no measured accuracy improvement can be claimed.

**2. UAE OCR AI (ten Azure DI models, hybrid local/cloud engine).**
Motor claims arrive as police accident reports from three emirates in incompatible layouts, plus six identity and vehicle card faces, all needing one schema for claims, liability and subrogation. Ten routes sit over ten custom-trained Azure DI models in a star-shaped module layout where each emirate handler imports only from `utils.py`, fronted by a cost-optimised engine: digital PDFs parsed locally with PyMuPDF, a local Tesseract pass on the page-1 header voting between the Sharjah old/new models and filtering Dubai pages before any billed call, deterministic anchor gates at a 50% hit rate returning 422, and insurer entity resolution against 42 records producing canonical subrogation codes. The hardest decision was one model per layout rather than one composed model — more labelling and model sprawl, bought in exchange for a blast radius that stops at one template. Honest limitation: no accuracy, throughput or cost-savings figures are recorded, and schema typos (`visiblity_condition`, `chasiss_no`) are already baked into consumer contracts.

**3. Trade License Extractor (deterministic-first, CPU-only, guard-railed LLM fallback).**
UAE trade licences arrive from five emirates and nine authority templates as born-digital PDFs, print-to-PDF rasters, flatbed scans, vector-outline files with no fonts, and multi-document bundles — with no GPU, no paid APIs and a 20-second budget. The pipeline classifies scanned versus digital with fitz, extracts words with PyMuPDF, triggers OCR only when a character-count threshold or an Arabic-sanity lexicon check fails, reconstructs lines from word spans, routes to a template, and extracts by label anchors; an Ollama `qwen2.5:3b` fallback runs only for unknown templates and low-confidence fields, with every LLM-sourced value flagged for review. The hardest decision was demoting the LLM: a trialled vision-LLM path returned Chamber/DCCI numbers as the Commercial Register number and Fax as Mobile, so I replaced it with deterministic negative-label guards. Honest limitation: the evaluation harness exists but the POC stored no outputs, so no accuracy percentage was recorded, and the "under 10 seconds" headline conflicts with the documented 12–13.5 s worst case.

**4. VL_version_OCR (provider-agnostic vision-language extraction).**
GCC identity, vehicle, passport and police documents span 18+ types and four jurisdictions, and per-template parsers do not scale there. YOLOv8 splits composite scans over pypdfium2 2× renders, then a pluggable async engine with six execution modes runs a two-stage pass: anchor classification, then a strict schema synthesised for that exact document type, with a membership guard dropping crops whose predicted type is not in the catalog. Police reports bypass the VLM entirely for four bespoke parsers combining PyMuPDF tables, Wand 300-DPI rendering, Google Cloud Vision and a custom coordinate sorter. The hardest decision was keeping both AsyncOpenAI and LangChain paradigms rather than discarding one — provider portability against a doubled test surface. Honest limitation: a UAE anchor/output key mismatch disables `/extract/uae` entirely, exactly the regression a config-consistency test would catch, and there are no automated tests at all.

**5. DOHA OCR AI (dual-path extraction and synthetic training data).**
Qatar MOI accident reports and identity cards feed FNOL intake, but the regional Azure tenant allowed only template build mode — no neural training — and per-page cloud OCR was a live cost. Cards go to four Azure DI custom models; MOI PDFs are parsed entirely locally by an Arabic-aware PyMuPDF/regex parser resolving RTL wrap artefacts in roughly 10 ms at zero cloud cost, behind a uniform `messageType 'S'/'E'` envelope that never raises an unhandled 500. The hardest decision was fixing the training distribution rather than the model class: early models missed the third vehicle party because every sample was single-page, so I built a synthetic pipeline that clones real reports and regenerates only digit-bearing rectangles — v1/v2 failed on mixed Arabic encodings before v3 preserved typography pixel-for-pixel — producing 300+ balanced samples. Honest limitation: no field-level exact-match accuracy was recorded for any of those models.

---

### Resume-Ready Positioning

**Target title:** Document AI / Intelligent Document Processing Engineer, or AI Engineer — Document Intelligence & Applied GenAI (equivalent targets: GenAI/LLM Engineer; Applied ML Engineer). *No document states a title. Assessed level from documented scope is mid-level to senior AI Engineer, weighted to the upper end for document-AI work specifically; aim at senior roles only where you can also speak to the measurement and testing gaps below.*

**Professional summary (4 lines)**

> AI Engineer specialising in document intelligence and applied LLM/VLM systems for insurance — OCR, key information extraction, classification and validation across UAE, Qatar, Oman and Kuwait identity, vehicle, trade-licence and police-report documents.
> Delivered 15 systems for Qatar Insurance Group spanning FastAPI microservices, Azure Document Intelligence custom models, YOLOv8 document segmentation, self-hosted Qwen/Gemma VLMs and Arabic/English bilingual normalisation.
> Designs hybrid cost-aware architectures that parse digital PDFs locally in milliseconds and escalate to cloud OCR or an LLM only when evidence demands it, with strict output schemas downstream claims and underwriting systems can bind to.
> Comfortable across the full lifecycle: synthetic training-data generation, QLoRA fine-tuning and quantised on-premise serving, containerised deployment, and candid documentation of limits and unmeasured claims.

**Skills line**

Python · FastAPI · Uvicorn · Pydantic v2 · PyTorch · Hugging Face Transformers · scikit-learn · Ultralytics YOLOv8 · DINOv2 · OpenCV · Pillow · PyMuPDF · pypdfium2 · Azure AI Document Intelligence (custom / prebuilt-read / prebuilt-layout QUERY_FIELDS) · Azure Translator · Google Cloud Vision · RapidOCR · Tesseract · Google Gemini · Qwen · Gemma 4 · Unsloth QLoRA · vLLM · Ollama · llama.cpp / GGUF · LangChain · AsyncOpenAI · structured/constrained decoding · RapidFuzz · arabic-reshaper / python-bidi · SentenceTransformers · FAISS · BM25 + RRF · Streamlit · PyQt5 · Docker · Nginx · HashiCorp Vault · Hugging Face Hub

**Twelve strongest resume bullets**

1. Architected a dual-path document extraction service routing identity and vehicle cards to four Azure Document Intelligence custom models while parsing MOI traffic accident PDFs locally with PyMuPDF, reaching **~10 ms per report at zero cloud OCR cost** across six REST endpoints and five document types. *(DOHA OCR AI)*
2. Engineered a synthetic training-data pipeline producing **300+ balanced samples** (1–7 vehicle layouts, cross-page injury-table splits, scan-style raster twins) to overcome a regional Azure tenant limited to template-only build mode, after rewriting the generator v1/v2 → v3 to survive mixed Arabic encodings. *(DOHA OCR AI)*
3. Built an Arabic-aware police-report parser resolving RTL bidirectional wrap artefacts via purpose-built `transfer_window()`, `fix_number_wrap()` and `is_label()` helpers plus lookaround-guarded regex anchors, extracting incident, vehicle-party, driver, injury and property-damage tables into nested JSON. *(DOHA OCR AI)*
4. Architected a **ten-route UAE motor-claims extraction API integrating ten custom-trained Azure Document Intelligence models** across three police jurisdictions and six card layouts, normalising them into one JSON schema with structured 400/404/422/502 error semantics. *(UAE OCR AI)*
5. Optimised cloud OCR consumption by detecting native PDF text layers and parsing them locally with PyMuPDF, and by using a local Tesseract pass on the page-1 header purely as a router — selecting between two Azure layout models and pre-filtering scanned pages before any billed call (spend avoided is not measured in the source). *(UAE OCR AI)*
6. Distilled a **Qwen3.5-122B-A10B teacher into a Gemma 4 E2B student**, labelling **553 images in 18 min 41 s (2.03 s/image)** with Pydantic-guided decoding and zero JSON decoding failures, then fine-tuning with 4-bit QLoRA on a versioned **564-train / 100-validation** stratified split. *(LLM Fine-tuning Platform)*
7. Diagnosed a **catastrophic memorisation failure** through forensic root-cause analysis and redesigned the recipe — frozen vision layers, completion-only loss, constant schema prompt, added validation split, 5 → 2 epochs. *(LLM Fine-tuning Platform)*
8. Quantised the merged model to GGUF Q4_K_S (**4,647,450,147 params, ~3.03 GB, n_ctx 65,536**) and deployed it on an on-premise llama.cpp server exposing the same OpenAI-compatible contract as the teacher, keeping customer PII on internal infrastructure under UAE PDPL and Qatar data-handling standards. *(LLM Fine-tuning Platform)*
9. Ran a forensic label audit that found **12 of 30 flagged fields wrong (40% error rate on flagged fields)** plus day/month swaps, unit contamination and duplicate captures, and built the Streamlit review dashboard that let three reviewers verify **664 UAE and 296 Doha records with zero data loss**. *(LLM Fine-tuning Platform)*
10. Built a CPU-only trade-licence extractor covering **9 UAE authority templates across five emirates** with selective OCR gating — **65–130 ms born-digital pages, 12–13.5 s worst-case 14-page raster bundles, all 16 corpus documents inside a 20-second budget** — plus negative-label guards derived from documented vision-LLM field confusions. *(Trade License Extractor)*
11. Designed a two-stage vision-language pipeline synthesising a strict Pydantic/JSON schema per document type (`strict=True`, `temperature=0`, `additionalProperties=False`) over **18+ document types and six passport nationalities**, behind a provider-agnostic engine with **six execution modes** across on-prem vLLM, Gemini and Azure. *(VL_version_OCR)*
12. Shipped an **OCR-free document classifier** from a frozen DINOv2-base backbone and a **19,442-byte LogisticRegression artefact** over six classes against a documented **7–10 second OCR baseline**, with one shared embedding function eliminating train/serve preprocessing skew. *(Document Classification Model)*

---

### Interview Narrative

**Two-minute "tell me about yourself"**

I'm an AI engineer specialising in intelligent document processing and applied generative AI, and almost all of my recent work has been document-AI systems for an insurance group across the UAE and Qatar. I've delivered around fifteen systems end to end, from deployed services to evaluated prototypes — OCR and extraction APIs for driving licences, Emirates and residency IDs, vehicle registrations, trade licences, passports and multi-jurisdiction traffic police reports; classification and triage services; a claim-duplication detector; a clause-recommendation engine for property underwriting; and a full vision-language-model lifecycle that distilled labels from a 122B teacher, ran them through a human-in-the-loop review tool I built, QLoRA fine-tuned a small Gemma student, and deployed it quantised on-premise so customer PII never leaves internal infrastructure.

Technically that's Azure Document Intelligence, YOLOv8 segmentation, RapidOCR and PyMuPDF, Qwen and Gemini VLMs with strict schema-constrained decoding, and FastAPI services with hard response contracts, per-item error isolation and deliberate concurrency. What I care about most is judgement rather than reaching for the biggest model: parsing digital PDFs locally in milliseconds instead of paying per page, using a cheap local OCR pass purely as a router to pick the right cloud model before anything is billed, keeping extraction deterministic with a guard-railed LLM fallback only where it earns its place, and getting Arabic right-to-left and bilingual normalisation right, because that's where most pipelines in this domain break.

I'm comfortable with the messy parts too — synthetic training data when a regional tenant only allows template-mode training, or a forensic write-up when a fine-tuning run memorises its training set. What I'm pushing hardest on now is measurement: evaluation harnesses and observability, so these thresholds and routing decisions are backed by numbers rather than judgement alone.

**Six STAR stories**

**1. The synthetic data pipeline (DOHA OCR AI).**
*Situation:* The regional Azure tenant supported only template build mode — no neural training — and external cloud services were constrained. *Task:* Get usable custom extraction models for MOI accident reports anyway. *Action:* I diagnosed the symptom precisely: early models missed the third vehicle party on multi-page reports because every training sample was single-page — a distribution gap, not a model defect. I built a synthetic pipeline inside the tenant boundary. Versions 1 and 2 redacted whole spans and reinserted reshaped Arabic, and failed on a corpus mixing logical order, visual order and presentation forms. Version 3 clones a real report and regenerates only digit-bearing rectangles — IDs, VINs, dates, plates — leaving Arabic glyphs and vector borders intact. *Result:* 300+ balanced samples covering 1–7 vehicle layouts, cross-page table splits and raster twins, plus a DI Studio annotation guide. No field-level accuracy was recorded, so I present it as an auditable training asset, not a measured win.

**2. Catastrophic memorisation (LLM Fine-tuning Platform).**
*Situation:* Run 1 of the Gemma 4 E2B fine-tune produced a model that, given a driving-licence back containing no personal names, emitted a name straight from training row 115. *Task:* Diagnose and fix before anything shipped. *Action:* I wrote a forensic report identifying three compounding causes — vision layers trained but the merged model served against the stock GGUF `mmproj`, so the visual pathway was broken at inference; five epochs with no validation split, so nothing could detect divergence; and an auto-mapper misreading the column format, so loss wasn't computed where I assumed. I redesigned the recipe: frozen vision layers, completion-only loss, a constant six-schema catalog prompt, two epochs and a real eval split. *Result:* A defensible v2 recipe and a deployed on-prem student — though the v2 evaluation was never run to a number, so I claim the diagnosis, not a measured improvement.

**3. Demoting the LLM on evidence (Trade License Extractor).**
*Situation:* Trade licences from five emirates and nine authority templates needed extraction on CPU-only hardware with no paid APIs. *Task:* Choose between a vision-LLM pass and deterministic parsing. *Action:* I trialled the VLM and it confused DCCI/Chamber numbers with the Commercial Register number, and Fax with Mobile — look-alike failures that would have quietly corrupted records. So I moved *down* the ladder: PyMuPDF word extraction with a template router, label-anchor extraction with explicit negative-label guards ("registration_no is never DCCI/Chamber No", phone preferred over mobile), OCR gated behind a character-count and Arabic-sanity check, and the LLM kept only as a guard-railed fallback for unknown templates with everything it touches flagged for review. *Result:* 65–130 ms on born-digital pages against a 44,902 ms RAG recovery run on one document, and all 16 corpus documents inside the 20-second budget.

**4. Thread-local ONNX sessions (Multi-doc All-in-one OCR / Document Detection API).**
*Situation:* Crops were to be processed concurrently on a ten-worker thread pool, but RapidOCR runs on ONNX Runtime, and a single shared session serialises on a mutex — so the pool would have bought little real parallelism. *Task:* Make the concurrency actually pay. *Action:* I moved the engine into a thread-local singleton so every worker owns its own session, accepting the memory cost of ten engines, and kept Azure Document Intelligence selectable per request as an alternative backend behind the same interface. *Result:* The pool runs without mutex serialisation across high-volume ingestion. Honestly, this landed as a design decision rather than a measured before/after — per-stage timing is exactly the instrumentation I would add now.

**5. The duplicated-rules regression (KYC Document Validation System).**
*Situation:* One validation layer served three front-ends — a Jinja2 canvas UI, Streamlit and a PyQt5 desktop app — and I duplicated the helper logic into each so every front-end could run standalone, later inlining the helpers and dropping the shared module. *Task:* Keep business rules consistent across three UIs. *Action:* I didn't, and it surfaced as silent drift. The notebook and the earlier `utils.py` accepted an optional date separator; production required one, so a separator-less OCR reading like `10061987` — which the project's own notebook shows RapidOCR producing — fell through to "No expiry date found" with no exception and no log line. The same pass dropped a file-existence check, turning a clean 404 into an unhandled error. *Result:* Rules now live in one importable module with UIs as thin clients, and anything that must agree in two places gets merged or gets a consistency test.

**6. Replacing OCR with embeddings (Document Classification Model).**
*Situation:* Document typing at motor-claims intake depended on the OCR path, documented at around 7–10 seconds depending on extraction. *Task:* Type six document faces far faster, offline, on CPU. *Action:* I skipped OCR entirely and classified from vision features: a frozen DINOv2-base backbone producing 768-d L2-normalised CLS embeddings under `torch.inference_mode()`, feeding a scikit-learn LogisticRegression head trained on a stratified 75/25 split with a fixed seed. One shared `get_embedding` function serves both training and inference so preprocessing cannot drift, the checkpoint is localised so nothing calls out at request time, and provenance is stored in the artefact bundle. *Result:* A 19,442-byte shippable head against a 346 MB backbone that never changes, retraining as a single call, and adding a class becomes a folder plus a retrain. Honest limit: no per-request latency or accuracy was recorded, and `REJECT_THRESHOLD = 0.0` means nothing is ever rejected yet.

**"What was your hardest technical challenge?"**
Lead with the Run 1 memorisation failure. It's hardest because the symptom was misleading — the model produced fluent, schema-valid JSON, so nothing structural flagged it; only inspecting a specific prediction against an image with no names revealed it had stopped conditioning on the image at all. Explain the three-cause diagnosis (trained vision LoRA against a stock `mmproj`, five epochs with no validation split, a misread column mapping), then the recipe redesign, then close on what you'd add: an exact-match harness on the 100-row validation split so the next divergence is caught at epoch one rather than by eye. The honesty about the missing v2 number is part of the strength of the answer.

**"Tell me about a mistake you made and what you changed."**
Use the KYC duplicated-rules story, not a small bug. The mistake was structural: I optimised for framework independence — three deployable front-ends with no shared dependency — and paid for it with no single source of truth, which let a regex diverge silently between the notebook, the earlier module and production. The concrete artefact is the `10061987` OCR token that production drops without an error. What changed is a rule I now apply everywhere: if two artefacts must agree, either merge them or assert the agreement in CI. That same rule is why my prescribed fix for the VL_version_OCR anchor-key mismatch is a config-consistency test rather than a more careful review.

**"How do you evaluate an OCR/LLM system?"**
Answer in tiers, and be honest that the labelled tier is the gap. Tier one needs no labels: with constrained decoding, schema validity should be ~100%, so any non-zero invalid rate is a decoder or prompt regression alarm. Tier two is grounding — does every emitted value actually appear in the source? I implemented exactly that check in the Trade License RAG path, and it catches the failure constrained decoding cannot: a well-formed object full of invented values. A clever tier-two signal I already emit and nobody reads is reviewer edit rate — the annotation tool stamps `edited_fields` on every correction, which is a continuous per-field accuracy proxy. Tier three needs ground truth: in Trade License Extractor I built an evaluation harness (`eval/run_eval.py`, `values_match()`) over a 16-document ground-truth corpus, reused across both the production and RAG pipelines. Then say plainly what's missing across the portfolio — most documents record no field-level accuracy, several thresholds (the 65%/90% classification cuts, pHash distance 5, `REJECT_THRESHOLD = 0.0`) were never calibrated against a labelled set — and what you'd instrument first: a labelled set per document type and region, field-level exact match, per-stage latency spans, and production score distributions so a threshold that stops fitting reality becomes visible. Add that you would *not* reach for LLM-as-judge on identity documents, because the judge shares the extractor's failure modes on Arabic and on look-alike fields.

---

### LinkedIn Profile Draft

**Headline**

> AI Engineer — Document Intelligence & Applied GenAI | OCR, VLMs, FastAPI | Arabic/English extraction for insurance claims, KYC & underwriting

*(Alternative: `GenAI & Document AI Engineer | VLM distillation and QLoRA fine-tuning to on-prem GGUF serving | Cost- and data-sovereignty-aware AI for regulated insurance`)*

**About**

I build document intelligence systems for insurance. Across fifteen projects for a GCC insurance group I have taken identity cards, vehicle registrations, trade licences, medical claim batches and traffic-police accident reports from a scanned page to validated, bilingual JSON that underwriting and claims platforms consume directly.

Most of my work sits where machine learning meets real engineering constraints. I have trained Azure Document Intelligence custom models under a regional tenant that only allowed template build mode, and built a synthetic data pipeline that regenerated digit-bearing rectangles inside real reports to produce 300+ balanced training samples. I have distilled a Qwen3.5-122B teacher into a Gemma 4 E2B student with 4-bit QLoRA, diagnosed a catastrophic memorisation failure through forensic review, redesigned the recipe, and shipped the quantised model on-premise via llama.cpp so customer PII never leaves internal infrastructure.

I default to the cheapest mechanism that meets the bar: parse a digital PDF locally in milliseconds rather than paying per page, use a cheap local OCR pass purely as a router to pick the right cloud model, keep extraction deterministic and let an LLM in only where it earns its place — fenced by strict schemas and flagged for review. I care about Arabic/English correctness, output contracts downstream systems can trust, and documenting what a system genuinely does, including what has not been measured yet.

Python · FastAPI · PyTorch · Azure AI · YOLOv8 · vLLM · Unsloth · llama.cpp

**Experience**

**AI Engineer *(title inferred — confirm your actual title before publishing)* — Qatar Insurance Group engagement**
*Document intelligence and applied GenAI for motor claims, KYC onboarding, underwriting and claims pre-screening across the UAE, Qatar, Oman and Kuwait.*

- Delivered 15 document-intelligence and applied-AI systems — from deployed services to evaluated prototypes — covering motor claims intake and FNOL, KYC identity verification, commercial trade-licence onboarding, medical claims pre-screening, underwriting clause governance and voice data capture.
- Architected FastAPI microservices with typed contracts, uniform success/error envelopes, per-item error isolation and explicit 400/404/415/422/500/502 semantics, deployed under Uvicorn behind Nginx and, in one case, as a hardened non-root Docker image.
- Built cost-aware hybrid extraction engines routing digital PDFs to local PyMuPDF parsers (~10 ms, zero cloud OCR cost) and only raster scans to Azure Document Intelligence, with a local Tesseract pass used as a model router and page filter ahead of billed calls.
- Integrated and trained document models across the stack: Azure DI custom extraction models under a template-only regional tenant, a custom-weights YOLOv8 multi-document detector (integration and threshold tuning), and a frozen DINOv2 + logistic-regression classifier shipping a 19 KB retrainable artefact.
- Ran a full VLM lifecycle: 122B teacher distillation with guided decoding (553 images in 18 min 41 s), a Streamlit human-in-the-loop review tool (664 UAE + 296 Doha records verified, zero data loss), QLoRA fine-tuning of Gemma 4 E2B, and GGUF Q4_K_S serving on-premise via llama.cpp for PII residency.
- Engineered Arabic/English bilingual pipelines: RTL bidirectional wrap correction in PDF text layers, arabic-reshaper/python-bidi with unshaped forms preserved for lookup, right-to-left polygon sorting, and Azure Translator / GoogleTranslator enrichment behind caches and timeouts.
- Built fuzzy n-gram document classifiers over hot-reloadable per-region JSON rule sets, plus a no-code Streamlit console letting operations staff author, empirically test and deploy those rules without a release cycle.
- Designed a hub-and-spoke orchestrator dispatching document crops to 14 downstream extractor mappings with bounded timeouts and per-crop error isolation, and a decoupled detection-only triage service that scales independently of extraction.
- Delivered production concerns alongside models: HashiCorp Vault credential loading, rotating latency audit logs, health probes, thread-local ONNX inference sessions, ThreadPoolExecutor and AnyIO concurrency, and `.env`/YAML-driven configuration.
- Authored source-cited technical documentation for every system, recording verified constants, unexercised branches, known defects and unmeasured claims rather than glossing them.

**Featured projects**

- **LLM Fine-tuning Platform** — teacher–student VLM distillation, human-in-the-loop audit, QLoRA fine-tune of Gemma 4 E2B, and on-prem GGUF serving via llama.cpp.
- **UAE OCR AI** — ten-route claims extraction API over ten custom Azure DI models, three police jurisdictions and six card layouts, with a cost-avoiding local parsing path.
- **Trade License Extractor** — CPU-only deterministic-first extractor across nine UAE authority templates with gated OCR and a guard-railed local LLM fallback.
- **VL_version_OCR** — provider-agnostic vision-language service with six execution modes and dynamically synthesised strict schemas across 18+ document types.
- **DOHA OCR AI** — dual cloud/local extraction with an Arabic-aware police-report parser (~10 ms, zero cloud cost) and a synthetic training-data pipeline of 300+ samples.

**Skills** *(ordered so the first three pin well)*

Document AI / Intelligent Document Processing · Large Language Models (LLMs) · Computer Vision · FastAPI · Python · OCR · Azure AI Document Intelligence · Vision-Language Models · Prompt Engineering · Structured / Constrained Decoding · PyTorch · Hugging Face Transformers · YOLOv8 · DINOv2 · OpenCV · PyMuPDF · scikit-learn · QLoRA / LoRA Fine-tuning · Unsloth · vLLM · Ollama · llama.cpp · GGUF Quantisation · Model Distillation · Google Gemini · Qwen · Gemma · LangChain · Pydantic · REST API Design · OpenAPI / Swagger · Microservices · Nginx · Docker · Streamlit · PyQt5 · RapidFuzz · Natural Language Processing · Arabic NLP · Machine Translation · SentenceTransformers · FAISS · RAG · Perceptual Hashing · Face Recognition (ArcFace) · Synthetic Data Generation · Human-in-the-Loop Annotation · Hugging Face Hub · Insurance Technology · Technical Documentation

**Projects**

1. **DOHA OCR AI** — Dual-path FastAPI service converting Qatar licences, QIDs, vehicle registrations, doctor licences and MOI accident reports into JSON.
2. **UAE OCR API — Query-Based Extraction** — Zero-shot `QUERY_FIELDS` extraction over UAE IDs, Mulkiya and trade licences with typed Pydantic v2 contracts.
3. **SLM AI OCR** — Seven-phase pipeline: YOLOv8 cropping, Azure prebuilt-read OCR, anchor gating and schema-constrained extraction on a self-hosted Qwen.
4. **Claim Duplication Detector API** — Page-level Unique/Blank/Duplicate triage of medical claim batches via perceptual hashing and OpenCV blank gating.
5. **Document Classification Model (Deep Learning)** — Offline OCR-free classifier from frozen DINOv2 embeddings with a 19 KB logistic-regression head.
6. **KYC Document Validation System** — Polymorphic validation service: keyword compliance, expiry, resolution, ArcFace face match and ROI OCR behind three UIs.
7. **TextMatch Label Trainer** — No-code Streamlit console for authoring, testing and hot-deploying per-jurisdiction classification keyword sets.
8. **Multi-doc All-in-one OCR** — Hub service segmenting, OCRing, classifying and routing crops to 14 downstream extractor mappings in one REST call.
9. **P&C Clause Recommendation Engine** — FAISS retrieval plus recency × frequency scoring and a Gemini leg, tiering missing property clauses for underwriters.
10. **Multilanguage Speech-to-Text Voice Data Capture** — Browser audio straight to a multimodal LLM, filling 13 motor-claim fields in any spoken language.
11. **Document Detection API (TextMatch Classification)** — Detection-only triage classifying, expiry-checking and quality-scoring uploads before extraction.
12. **Trade License Extractor** — CPU-only extractor across nine UAE authority templates with gated OCR and a guard-railed local LLM fallback.
13. **UAE OCR AI** — Ten-route claims extraction API over ten custom Azure DI models with hybrid local/cloud routing and insurer entity resolution.
14. **LLM Fine-tuning Platform** — Full VLM lifecycle from 122B teacher distillation to QLoRA fine-tune and on-prem quantised serving.
15. **VL_version_OCR** — Provider-agnostic vision-language OCR with six execution modes and four bespoke police-report parsers.

---

### Before You Publish

Confirm or supply each of the following. None of it can be derived from the project documents.

**Identity and employment**
- [ ] **Years of experience** — no document states it. Do not let a reader infer a career length from fifteen 2026-dated projects.
- [ ] **Exact job title and employer** — every ownership level in these analyses is inferred. Confirm whether you were a direct employee, a contractor, or engaged through a vendor/systems integrator, and name the correct employer of record.
- [ ] **Team size and reporting line** — no document names a team, roster or manager. The only named collaborators anywhere are the annotation reviewers on the fine-tuning project ("Afsal", "waris"). If you led anyone, say so explicitly; if you did not, the current "sole end-to-end ownership" framing is the accurate one.
- [ ] **Whether you personally trained the YOLOv8 weights** (`yolo-multipage-OCR-cls.pt`) — the documents say "custom-trained" but never record the dataset, labelling protocol, class list or who produced them. Currently framed as integration and threshold tuning.
- [ ] **Whether the Azure DI card models in DOHA OCR AI pre-existed** — the document does not say.

**Dates and status**
- [ ] **Start and end date per project**, and which ran in parallel — all fifteen are dated 2026 with no durations.
- [ ] **Deployment status of each system** — production, pilot, internal tool, PoC or abandoned. Several documents describe PoC-level readiness (the Document Classification Model launched from a notebook cell; the Speech-to-Text suite explicitly scoped to pilot/internal use); others describe live deployment behind Nginx or Docker. Do not let a reviewer read all fifteen as production.
- [ ] **Current status of known defects** — e.g. the VL_version_OCR UAE anchor/output key mismatch that disables `/extract/uae`, the KYC expiry-regex drift and missing `filepath.exists()` check, the SLM AI OCR `PORT` env-case TypeError. If you fixed them after the documents were written, say so; if not, they remain honest interview material.

**Metrics you must add yourself**
- [ ] **Field-level accuracy** for any extraction service — seven of the fifteen documents record no metrics at all, and even the projects with figures record latencies and artefact sizes, not accuracy.
- [ ] **Production throughput** (documents/hour or /minute) and **volume processed** — never recorded anywhere.
- [ ] **The fine-tuning v2 evaluation result** on the 100-row validation split — the harness and split exist; the number was never recorded.
- [ ] **The Trade License `values_match()` result** over the 16-document corpus — the harness exists; the POC notebook stored no outputs.
- [ ] **Cost savings actually realised** by the local-parse and page-filtering paths — cost avoidance is a stated design driver throughout but no Azure spend avoided is recorded.
- [ ] **Any ROI, labour-hours-saved or headcount figure** — no document contains one. Do not invent any, and do not present scope counts (endpoints, fields, document types, models) as performance results.
- [ ] **Threshold provenance** — the 65%/90% classification cuts, RapidFuzz 80/85/70, pHash Hamming 5, the OpenCV quality cuts and `REJECT_THRESHOLD = 0.0` were largely not calibrated against a labelled set. Presenting them as tuned would overstate the evidence.

**Confidentiality and compliance**
- [ ] **Whether you may name Qatar Insurance Group / QIC / Anoud publicly** — check your contract or NDA. If not, use "a GCC insurance group" or "a Gulf-based multiline insurer" throughout the CV and LinkedIn profile.
- [ ] **Whether internal identifiers may appear publicly** — model names (`anoud_ocr_*`), internal hostnames and IPs, endpoint paths, the `Peterleo5/uae-ocr-test` Hugging Face namespace, and the insurer master-data example (ADNIC → 2600040). Strip or generalise anything the client would consider internal.
- [ ] **Whether any sample document values may be quoted** — the trade-licence company name and licence numbers reproduced in the query-based extraction notebook are real-looking third-party data.
- [ ] **PII and data-residency framing** — the on-prem GGUF deployment is described as aligned with UAE PDPL and Qatar financial data handling standards. Confirm that characterisation with whoever owns compliance before repeating it publicly.


# Part II — Collective Analysis

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


## E. Interview Preparation Material

Fifty technical questions with detailed answers, grouped into five themes that map to the project portfolio. Each answer names the project to anchor it in, a project tie-in line, and the follow-ups a strong interviewer would ask next. Project-specific discussion points and architecture explanation points are in sections 9 and 10 of every per-project analysis in Part I.

## E. Interview Preparation (Part III) — 50 Questions & Answers

*Fifty questions across five themes, numbered continuously Q1–Q50: Theme 1 OCR & Document AI (Q1–Q10), Theme 2 Computer Vision & Deep Learning (Q11–Q20), Theme 3 LLMs, Prompt Engineering & Fine-tuning (Q21–Q30), Theme 4 NLP, Classical ML & Similarity/Recommendation (Q31–Q40), Theme 5 System Design, APIs, Deployment & Ownership (Q41–Q50). Every answer is followed by a "Project tie-in" line and a "Follow-ups to expect" list. Project names follow the per-project analyses; seniority is never asserted as a title, only as scope.*

### Theme 1: OCR & Document AI

**Q1. Azure Document Intelligence gives you prebuilt-read, prebuilt-layout with query fields, and custom extraction models. When do you reach for each, and what did you actually use?**

Answer: I've shipped all three, and the choice comes down to how stable the layout is and whether I'm allowed to annotate.

Custom extraction models make sense when the document is a fixed template you own and you need field-level precision. In **UAE OCR AI** I built the service on ten custom-trained models (`anoud_ocr_uae_PR_sharjahnew/old`, `_abudhabi`, `_dubai`, plus six card models `anoud_ocr_custom_uae_df/db/rf/rb/vf/vb`), one per layout, deliberately so a Sharjah template change couldn't regress the Dubai path — and I'd be straight that the write-up records the models as custom-trained without recording the labelling and training runs themselves. In **DOHA OCR AI** the same pattern gave me four card models (`24_07_doha_df/rf/vf`, `28_08_doha_doctor_license`).

Query-based extraction on `prebuilt-layout` is the right call when you can't afford annotation cycles or the formats keep drifting. In **UAE OCR API — Query-Based Extraction** I ran `DocumentAnalysisFeature.QUERY_FIELDS` with per-document presets (`df`, `db`, `rf`, `rb`, `vf`, `vb`, `tr`) — zero training, and a `/api/v1/extract/query` endpoint takes caller-supplied field lists for invoices or customs docs. The trade-off is real: no per-field confidence, no document-type validation, and my accuracy baseline lives entirely in the vendor's model.

`prebuilt-read` is pure OCR — I used it in **SLM AI OCR** and **Multi-doc All-in-one OCR** purely to get line text plus polygons, then did classification and extraction myself.

The honest caveat: on the Doha regional tenant only `template` build mode was available — no neural training — so I offset layout sensitivity with a synthetic dataset rather than pretending the model generalised.

Project tie-in: Anchor the answer in the one-model-per-layout decision in UAE OCR AI, then contrast with query fields in the UAE Query-Based API as the "no annotation budget" path.

Follow-ups to expect:
- How would you measure whether query fields beat a custom model here?
- What does template build mode cost you versus neural?
- How do you version and roll back a custom model?

**Q2. Walk me through how you decide between parsing a PDF's text layer and sending it to cloud OCR.**

Answer: This is a routing decision I made explicitly in three projects, and it's driven by cost, latency and data residency — not by convenience.

In **DOHA OCR AI** I built a dual-path pipeline: card documents (licences, QIDs, vehicle registrations) go to Azure Document Intelligence custom models, but MOI traffic accident PDFs are parsed entirely locally by `police_report_extraction.py` using PyMuPDF over the native text layer. That path runs in roughly 10 ms at zero cloud OCR cost and never leaves the premises. In **UAE OCR AI** the same idea appears as `is_scanned_pdf`: digital Abu Dhabi and Dubai reports are parsed locally with PyMuPDF text blocks and tables, and only raster scans are delegated to Azure. In **Trade License Extractor** I made the gate explicit — PyMuPDF word extraction first, OCR triggered only when a character-count sufficiency threshold or an Arabic-sanity lexicon check fails, plus an `ensure_ocr()` guard that catches zero-character vector-outline PDFs where fonts have been converted to paths.

The measured payoff is in the trade licence latencies: born-digital pages complete in 65–130 ms, single-page OCR documents in 6–10.7 s, and worst-case 11- and 14-page raster bundles in 12–13.5 s.

The failure mode I have to own is hybrid PDFs — a scanned page pasted into an otherwise digital document. A pure page-count heuristic passes those through with partial text. My fix is per-page gating rather than per-document, which is exactly what `ensure_ocr()` does; and in DOHA the local parser has no OCR fallback at all, so scans fail the Arabic header gate and return a structured error instead of garbage.

Project tie-in: Lead with the ~10 ms DOHA local parse and the trade-licence latency bands — they're the strongest verbatim numbers on this theme.

Follow-ups to expect:
- What character threshold did you use and how did you pick it?
- How do you detect a page that is 80% text and 20% scan?
- What breaks if a PDF has an invisible OCR text layer of poor quality?

**Q3. How did you handle Arabic and English on the same document — reading, matching, and outputting it?**

Answer: Arabic is three separate problems and I hit all of them.

First, PDF text layers. In **DOHA OCR AI** the MOI accident reports store Arabic in visual order, so numbers and labels wrap in ways that break naive search. I wrote `transfer_window()`, `fix_number_wrap()` and `is_label()` in `police_report_extraction.py`, developed step by step in `doha.ipynb` cells 6–13, plus lookaround-guarded regex anchors — `INCIDENT_NUMBER` as `(?<![\d-])\d{4}-\d{1,5}-\d{1,6}(?![\d-])`, `PLATE` as `(?<!\d)\d{4,7}(?!\d)` — so a plate can't be swallowed by an adjacent number.

Second, representation. In **UAE OCR AI** I kept two forms of every Arabic string: the unshaped form for translation and catalogue lookup, and a reshaped form via `arabic-reshaper` plus `python-bidi` for display and legal audit. Mixing those up is how you get lookups that silently never match. For scanned Dubai vehicle strings Azure returns unordered word polygons, so I sort them right-to-left to rebuild `makeModelColorYear`.

Third, output. Azure Translator produces the `*_english` counterparts for driver names, accident narratives and vehicle attributes; **DOHA OCR AI** also goes the other way, translating foreign nationalities into Arabic for regulatory reporting. In **UAE OCR API — Query-Based Extraction** I cached translations with `@lru_cache(maxsize=1024)` because emirate names and nationalities repeat constantly.

Honest limits: machine-translating proper nouns is unreliable, and I'd prefer curated lookup tables for bounded vocabularies (emirates, nationalities) with MT reserved for open text — or read the Arabic already printed on the card with a second query set rather than translating the English.

Project tie-in: Name the three RTL helpers and the unshaped-vs-reshaped split — those are the details that prove hands-on Arabic work rather than "we called a translator API".

Follow-ups to expect:
- Logical vs visual order vs presentation forms — what's the difference?
- Why did your synthetic generator v1/v2 fail on Arabic?
- How would you validate translated Arabic against what's printed on the card?

**Q4. How do you crop and segment a page when one upload contains several documents?**

Answer: I used a YOLO detector as a pre-OCR splitter in four services, with slightly different rules each time.

In **Multi-doc All-in-one OCR** and **Document Detection API (TextMatch Classification)** the pipeline is: PyMuPDF rasterises each PDF page at 150 DPI (`matrix 150/72`), then Ultralytics YOLO (`yolo-multipage-OCR-cls.pt`, `imgsz=320`, `conf=0.25`) produces per-document crops, with a full-frame fallback if nothing clears the confidence threshold. In **SLM AI OCR** I render with pypdfium2 at `scale=2`, decode with OpenCV, crop with the same model, and divert anything the detector classes as non-document to `skipped_images/`. In **VL_version_OCR** I tuned the thresholds explicitly in `debug.ipynb` cells 116–138, settling on `conf=0.5`/`IoU=0.4` over `iou=0.9`, and kept the uncropped image as class `doc` so single-document uploads still work.

Two region-specific carve-outs matter. Qatar cards print front and back on one sheet, so **Multi-doc All-in-one OCR** bypasses YOLO entirely for the DOHA region and passes the whole image as one crop. And any crop classed as police triggers a rule that discards the other crops and substitutes a single full-frame police-report crop — a deliberate simplification I'd revisit, because it silently drops co-submitted cards.

The two-resolution choice in VL_version_OCR is worth calling out: 2.0x render for detection quality, then downscale to 1536×1536 JPEG q80 for the model payload. Detection wants pixels; inference wants a small request.

I'll also be straight that the training provenance of `yolo-multipage-OCR-cls.pt` — dataset, labelling, class list — isn't documented, so I treat it as an artefact I tuned and deployed rather than one I can defend statistically.

Project tie-in: Use the DOHA YOLO bypass and the police crop-discard rule as evidence of real-world layout exceptions, not textbook segmentation.

Follow-ups to expect:
- Why 150 DPI and why `imgsz=320`?
- What happens when two crops overlap?
- How would you evaluate the detector without labelled crops?

**Q5. In your query-based extraction service, why zero-shot query fields instead of training a custom model, and what did that cost you?**

Answer: The driver in **UAE OCR API — Query-Based Extraction** was change velocity. UAE identity and commercial documents get redesigned, and each emirate varies; a custom-model workflow means annotation sets, training runs and weight redeployment for every variation. Query fields put the field semantics in natural-language strings — `Main_Licence_No`, `Company_Name`, `Register_No`, `Issue_Date`, `Expiry_Date`, `Email`, `Phone_No`, `Mobile_No` — evaluated against `prebuilt-layout` with `DocumentAnalysisFeature.QUERY_FIELDS`. I proved it in `workbook.ipynb` first: Cell 3 pulled all eight fields off a real Dubai trade licence, and the values in that run — `Company_Name` "STRONG PLANT FOR DEWATERING & DRAINAGE SERVICES (L.L.C)", `Main_Licence_No` 120278, `Register_No` 1014241 — are the same values in the production `TradeLicenseData` sample response. That traceability from experiment to shipped contract was deliberate.

One code path then covers seven document sides plus arbitrary documents via `/api/v1/extract/query`, which returns `extracted_fields`, `query_count` and `matched_count` so a caller can detect partial extraction.

The costs are real and I don't hide them. There's no per-field confidence and no document-type validation — a driving-licence image posted to the trade-licence endpoint will return whatever the layout model can find. My accuracy is entirely the vendor's; I have no in-house baseline, and the documentation records no accuracy or throughput figures at all. The fixed endpoints don't even expose `matched_count`, so only the dynamic route surfaces partial extraction.

What I'd add: an anchor-keyword document-type gate before spending the call — exactly the pattern I used in **UAE OCR AI**, where a card must hit ≥50% of its anchor phrases or it's rejected with HTTP 422.

Project tie-in: Pair the zero-shot argument with the anchor-gate fix from UAE OCR AI — it shows you know the gap and already have the pattern.

Follow-ups to expect:
- How would you build a per-document-type accuracy benchmark?
- What does a query field actually do inside the layout model?
- When would you switch back to custom models?

**Q6. Your services return confidence in several different ways. How do you think about confidence and thresholds in a document pipeline?**

Answer: I'll be candid: across these projects confidence is handled inconsistently, and I can explain each choice and where it's weak.

Some confidence is genuinely model-derived. In the **Document Classification Model** (service title "Motor Doc Check AI") the classifier returns `predict_proba` argmax rounded to four decimals — real probabilities from a `LogisticRegression` head over frozen DINOv2 embeddings. But `REJECT_THRESHOLD = 0.0`, so nothing is ever rejected; a captured page-2 prediction at 0.7935 sailed through alongside a 0.9458 one. The fix is calibrating a threshold on the stratified 25% hold-out and emitting an `unknown` class instead of a forced label.

Other "confidence" is a coverage score, not a probability. In **Document Detection API** and **Multi-doc All-in-one OCR**, confidence is the percentage of a label's keywords matched under RapidFuzz at cutoff 80, with tiers at 65% (suggest more keywords) and 90% (low-confidence warning). That's explainable to an operator but it's a match rate, and I should name it as such rather than let a downstream system treat it like a probability. The honest wart: the PoC tested a 50% cutoff and production shipped 65/90 with no recorded recalibration.

And some gates are deliberately deterministic because they need to be auditable. **UAE OCR AI** uses an anchor hit rate of ≥50% for cards and a "more than 3 non-empty fields" gate for police reports, both returning 422. Those survive model retrains, which a confidence threshold does not.

What I'd standardise: one numeric field with a declared semantic, a stated threshold, and a labelled validation set behind it. None of these thresholds were tuned against ground truth, and that's the first thing I'd fix.

Project tie-in: Lead with `REJECT_THRESHOLD = 0.0` as a self-critique — volunteering it reads far stronger than being caught by it.

Follow-ups to expect:
- How would you calibrate a reject threshold from a hold-out set?
- Azure DI returns field confidences — why didn't you surface them?
- What's the operational cost of a false accept here versus a false reject?

**Q7. Walk me through your trade licence extractor. Why deterministic-first rather than LLM-first?**

Answer: **Trade License Extractor** handles UAE trade licences from Abu Dhabi, Dubai (mainland and JAFZA), Sharjah, Ajman and Umm Al Quwain, arriving as born-digital PDFs, print-to-PDF rasters, flatbed scans, vector-outline files with zero fonts, and multi-document bundles of licence plus register plus chamber certificate.

The pipeline is: a fitz-based scanned/digital classifier, PyMuPDF word extraction with OCR gated behind a character-count threshold and an Arabic-sanity lexicon check, coordinate-aware line reconstruction from word spans, then a template router covering nine known UAE authority templates that derives `issuing_authority` and `issuing_emirate`, then regex/label-anchor field extraction. A local LLM fallback — Ollama `qwen2.5:3b-instruct-q4_K_M` over httpx `/api/chat` with `format='json'` — runs only for unknown templates and low-confidence fields, and every LLM-sourced value lands in `review_flags`.

Deterministic-first was evidence-driven, not dogma. I trialled a vision-LLM path and it confused DCCI/Chamber No with the Commercial Register number, and Fax with Mobile. So I encoded negative-label guards: `registration_no` is the Commercial Register number, never DCCI/Chamber No; `phone` is preferred over `mobile`. The latency argument reinforces it — 65–130 ms native versus a 44,902 ms RAG recovery run with nine LLM calls on one document.

Output is a `TradeLicenseRecord` with per-field confidence, provenance (page and extraction method), nulls for absent values, and `finance_view()`/`gi_view()` projections so Finance and GI get their own vocabulary from one audited record.

Limitations I'd raise myself: the POC notebook has no stored outputs, so no accuracy percentage is recorded even though `values_match()` in `eval/run_eval.py` exists; and the "under 10 seconds" headline conflicts with my own 12–13.5 s worst case, so I quote the 20-second operational budget instead.

Project tie-in: The DCCI-vs-Register-No story is the single best anecdote here — a concrete failure that changed the architecture.

Follow-ups to expect:
- How is per-field confidence computed and what triggers fallback?
- What happens when Ollama isn't running?
- How do you add a tenth authority template?

**Q8. You used both classical OCR and vision-language models for extraction. How do you choose, and what changed when you moved to VLMs?**

Answer: I've built both and the deciding factors are latency, cost, data residency and how much structure you need guaranteed.

The classical path is Azure Document Intelligence or RapidOCR producing lines and polygons, followed by my own logic — that's **UAE OCR AI**, **DOHA OCR AI**, **Document Detection API**. It's deterministic, cheap per page on the local paths, and every field maps to a rule I can point at in an audit.

The VLM path replaces OCR plus regex with a single image-grounded pass. In **VL_version_OCR** I built a two-stage pipeline: Stage 1 classifies the document type against `DOCUMENT_ANCHORS`, Stage 2 synthesises a strict Pydantic/JSON schema from `OUTPUT_DICT` and queries the model with `strict=True`, `temperature=0`, `additionalProperties=False`, covering 18+ document types across UAE, Qatar, Oman, Kuwait and six passport nationalities. The engine is pluggable across on-premise Qwen3-VL-4B-Instruct on vLLM, Gemini-3-Flash-Preview, and Azure Llama-4-Scout, selected per request via a `model_name` enum. An earlier OCR-plus-regex prototype was abandoned for high latency and per-field regex complexity.

**SLM AI OCR** is the hybrid: Azure `prebuilt-read` does recognition, a self-hosted Qwen via Ollama does understanding — cloud for OCR, local for PII-bearing entity extraction.

What changed: I stopped writing per-field regex and started writing schemas, and constrained decoding replaced brittle JSON fence-stripping. What got harder: no per-field confidence, and a schema/anchor mismatch fails silently. In VL_version_OCR the UAE anchor keys drifted from the `OUTPUT_DICT` keys, so `/extract/uae` returns 400 on every request — a config-consistency test asserting anchor keys are a subset of output keys per country would have caught it.

Project tie-in: Owning the UAE anchor-key regression, plus naming the exact test that prevents it, is the strongest move available here.

Follow-ups to expect:
- Why two VLM stages instead of one prompt?
- What does `strict=True` actually guarantee?
- How do you evaluate a VLM extractor without ground truth?

**Q9. Post-processing: what happens to raw OCR output before it reaches a downstream insurance system?**

Answer: Raw OCR is never the contract. Several layers sit between it and the consumer.

Reading order first. Azure `prebuilt-read` returns line fragments with polygons, not logical text, so in **SLM AI OCR** I wrote `combine_text_lines`: agglomerative vertical clustering with a 10-pixel tolerance, then horizontal word sorting. **VL_version_OCR** has an equivalent `coordinates_sort.py` with the same 10 px line threshold for Sharjah scans coming out of Google Cloud Vision.

Then normalisation. Dates go to a single format — DD/MM/YYYY in **UAE OCR AI**, ISO `YYYY-MM-DD` in the fine-tuning schemas. Arabic gets shaped for display but kept unshaped for lookup. Free-text insurer names get resolved to canonical internal codes: RapidFuzz token-sort/token-set at threshold 85 against 42 records in `insurers.json`, emitting `insurance_company_code` (ADNIC → 2600040) so subrogation recovery has something to key on. Field mapping into typed schemas happens in `utils.py` parsers in **DOHA OCR AI** and Pydantic v2 models in the **UAE Query-Based API**.

Then the envelope. Every service returns `message`, `messageType` `'S'`/`'E'`, and a typed response; DOHA explicitly never raises an unhandled 500, so a malformed file, empty payload or Azure timeout returns a structured error a downstream pipeline can branch on.

Two things I'd fix rather than defend. Schema typos are now baked into contracts — `visiblity_condition` and `chasiss_no` in UAE OCR AI, `chasiss_number` in DOHA, plus a duplicate `exp_date`/`expDate` pair. The right fix is corrected keys behind a versioned route, emitting both until integrated consumers migrate — not a silent rename. And returning HTTP 200 with `messageType: 'E'` makes monitoring harder than proper status codes would.

Project tie-in: The insurer fuzzy-resolution step is the most business-legible post-processing example — it directly enables subrogation.

Follow-ups to expect:
- Why threshold 85 for insurer matching, and what happens on a near-tie?
- How do you add an insurer without a redeploy?
- Why 200-with-error rather than a 4xx?

**Q10. You were blocked from training neural custom models in one region. How did you get a usable extraction model anyway?**

Answer: In **DOHA OCR AI** the regional Azure tenant supported only `template` build mode — no neural training — and external cloud services were constrained. Template models are layout-sensitive, so the answer was to fix the training distribution rather than the model class.

The first symptom was concrete: early models failed to extract the third vehicle party on multi-page accident reports, because every training sample was single-page. That's a distribution gap, not a model defect.

So I built a synthetic data pipeline inside the tenant boundary. Version 1 and 2 redacted whole spans and reinserted reshaped Arabic — they failed, because the corpus mixed logical order, visual order and presentation forms, and reinsertion corrupted the typography. Version 3 changed the strategy: clone a real MOI report and regenerate only the digit-bearing rectangles — IDs, VINs, dates, plates — leaving Arabic glyphs and vector borders pixel-for-pixel intact. Machine-readable values vary; label alignment and typography survive. Batch scripts (`generate_split_variants.py`, `generate_table_variants.py`, `generate_final_set.py`) produced 300+ balanced samples covering 1–7 vehicle layouts, cross-page injury-table splits and scan-style raster twins, with an annotation guide for Azure DI Studio labelling.

There's a twist worth being honest about: I trained and evaluated `04_08_demo_pol` for police reports, but production ships the local PyMuPDF parser instead — ~10 ms, zero cloud cost, fully on-premise. The synthetic corpus remains an auditable training asset for when scans need covering.

What I can't claim: no field-level exact-match accuracy was recorded for those models. The documentation says so explicitly. The evaluation set is the thing I'd build first if I picked this up again.

Project tie-in: The v1/v2 → v3 rewrite is a genuine engineering-judgement story — lead with why whole-span reinsertion failed.

Follow-ups to expect:
- Why not train in another region and import the model?
- How do you know 300 samples is enough?
- What would a proper evaluation set for this look like?


### Theme 2: Computer Vision & Deep Learning

**Q11. You used YOLOv8 as a document detector in several services. Walk me through what the model is actually predicting, and why detection rather than classification or segmentation was the right primitive here.**

Answer: In SLM AI OCR, Multi-doc All-in-one OCR, Document Detection API (TextMatch Classification) and VL_version_OCR the same custom weights file — `yolo-multipage-OCR-cls.pt` — predicts axis-aligned bounding boxes around each physical card or document region on a rasterised page, plus a class label. The class label matters as much as the box: in SLM AI OCR any detection that is not a document class is written to `skipped_images/` rather than sent to Azure OCR, and in VL_version_OCR non-`doc` classes (in practice police-report pages) are diverted the same way. In Multi-doc All-in-one OCR a crop whose class contains "police" causes all other crops to be discarded and replaced by a single full-frame `police_report` crop.

Detection is the right primitive because the business problem is "one upload contains N documents in unknown positions" — a customer photographs a licence front and back on one sheet, or submits a 14-page PDF bundle. A classifier answers "what is this page", which is useless when the page holds three cards; segmentation would give pixel masks I do not need, since every target is a rectangular card and the downstream consumer is a crop buffer for OCR or a VLM. Boxes are also cheap: I ran inference at `imgsz=320`, `conf=0.25` on CPU in the Multi-doc All-in-one OCR and Document Detection API services.

Honest limitation: the documentation records the weights as "trained" but does not record the dataset, labelling protocol, class list, or mAP. I would not claim a detection accuracy figure I never measured.

Project tie-in: Anchor on SLM AI OCR or Multi-doc All-in-one OCR and describe the crop-then-recognise split as a cost decision, not just an accuracy one.

Follow-ups to expect:
- What happens when YOLO returns zero boxes?
- Why axis-aligned boxes and not oriented boxes for tilted cards?
- How would you add a new document class?

---

**Q12. In Multi-doc All-in-one OCR and the Document Detection API you ran YOLO at imgsz=320 with conf=0.25. Defend both numbers.**

Answer: Both are deliberate CPU-era trade-offs. `imgsz=320` is the letterbox size the image is resized to before inference; compute scales roughly with the square of that dimension, so 320 versus the Ultralytics default 640 is close to a 4x reduction in convolution cost. The task tolerates it because I am localising card-sized rectangles that occupy a large fraction of a 150-DPI page render (PyMuPDF matrix 150/72), not small text glyphs — the recognition step never sees the 320px tensor, it sees the crop taken from the full-resolution page. That two-resolution split is explicit in VL_version_OCR too: pypdfium2 renders at 2.0x for detection quality, then the crop is downscaled to 1536x1536 JPEG q80 as the VLM payload.

`conf=0.25` is low by classification standards, and deliberately so. The cost asymmetry is one-sided: a missed card means the document silently never reaches the extractor and the customer's claim stalls; a spurious crop costs one extra OCR call and is usually killed downstream by the fuzzy keyword classifier failing to reach the 80 cutoff, or by the anchor gate. So I bias recall over precision at the detector and let the classifier and validity gates do the filtering. The whole-frame fallback reinforces that — if nothing clears 0.25, the uncropped image is processed as class `doc` rather than the request failing.

What I cannot claim: I have no recorded precision/recall or mAP sweep behind 0.25. In VL_version_OCR I did record a comparison in `debug.ipynb` cells 116-138 — `iou=0.9` versus `conf=0.5 / iou=0.4` — and shipped the latter, but that is a qualitative verdict, not a PR curve.

Project tie-in: Cite the VL_version_OCR notebook threshold comparison as evidence you tune empirically, then be candid that the other services inherited the values.

Follow-ups to expect:
- What would a proper threshold sweep look like?
- Why is IoU/NMS threshold 0.4 rather than 0.9?
- How does a missed detection surface to the caller?

---

**Q13. Explain mAP, and how you would build an evaluation set for your document detector given none exists today.**

Answer: mAP is the mean over classes of average precision, where AP is the area under the precision-recall curve produced by sweeping the confidence threshold; a detection counts as a true positive only if its IoU with a ground-truth box exceeds a threshold — mAP@0.5 uses a single 0.5 IoU cut, mAP@0.5:0.95 averages over ten cuts and therefore penalises sloppy box regression. For my case the distinction matters less than usual: I crop and pass the crop to OCR or a VLM, so a box that is loose by a few percent is harmless as long as it fully contains the card. That argues for reporting recall at a generous IoU (0.5) plus a *containment* metric — fraction of ground-truth cards fully inside a predicted box — rather than optimising mAP@0.5:0.95.

To build the set honestly: sample real uploads across the classes the services actually see — UAE licence front/back, Emirates ID front/back, vehicle Mulkiya front/back, Doha cards, and the police-report pages that VL_version_OCR and Multi-doc treat specially — stratified by source (phone photo, flatbed scan, print-to-PDF), because Trade License Extractor showed provenance drives everything. Label boxes plus class. Then report per-class recall at conf=0.25, the false-positive rate per page, and the end-to-end metric that actually matters: fraction of uploads where every real document reached its extractor.

I would reuse the review-tool pattern from the LLM Fine-tuning Platform — the Streamlit dashboard with `edited_by`/`edited_fields` provenance — for box correction, since it already proved that a human-in-the-loop gate is what makes labels trustworthy (that audit found a 40% error rate on flagged teacher fields).

Project tie-in: Frame the missing eval set as the first thing you would add, and point at the HITL tooling you have already built as the mechanism.

Follow-ups to expect:
- Why is mAP@0.5:0.95 the wrong headline metric for you?
- How many images per class would you need?
- How do you detect detector drift in production?

---

**Q14. In the KYC Document Validation System you used DeepFace with ArcFace and `enforce_detection=False`. Explain what that flag does and what risk it introduces.**

Answer: `DeepFace.verify` normally runs a face detector first, aligns the crop, then embeds it with the chosen backbone — here ArcFace — and compares embeddings by distance against a model-specific threshold, returning `verified`, `distance` and a derived confidence. With `enforce_detection=False` the detection stage is allowed to fail without raising: DeepFace falls back to embedding whatever region it has rather than throwing an exception.

I set it in all three front-ends (the FastAPI/Jinja2 canvas UI, the Streamlit UI and the PyQt5 desktop app) because the inputs are portrait photos printed at low resolution on GCC identity documents — a passport-style headshot a few hundred pixels wide, sometimes behind a security guilloche. A hard detector failure would have turned a recoverable case into an HTTP error for the operator.

The risk is real and I would flag it in an interview rather than hide it: with detection off, ArcFace will happily embed a crop that contains no face at all, and return a distance. That distance is meaningless but structurally indistinguishable from a genuine comparison, so a non-face ROI can produce a `verified: false` with a plausible-looking confidence instead of an explicit "no face found". The fix is not to flip the flag back on — it is to run detection as a separate explicit step, return a distinct `no_face_detected` status when it fails, and only then decide whether to attempt a forced embedding as a degraded path.

The deeper gap: ArcFace's decision threshold was never calibrated on this document population, and no FAR/FRR was measured. That is what I would build first — a labelled document-photo/selfie pair set and an ROC.

Project tie-in: Lead with the operational reason for the flag, then volunteer the non-face-embedding failure mode and the FAR/FRR gap unprompted.

Follow-ups to expect:
- How does ArcFace's angular margin loss differ from a softmax classifier?
- What is FAR versus FRR and which would you tune for KYC?
- How would you handle a spoofed printed photo?

---

**Q15. The Document Classification Model uses a frozen DINOv2-base backbone with a logistic regression head instead of fine-tuning a CNN. Justify that architecture.**

Answer: The service loads DINOv2-base from a local checkpoint (`Models/dinov2-base`, 12 layers, 12 heads, hidden 768, patch 14, image_size 518) in `.eval()` under `@torch.inference_mode()`, takes `last_hidden_state[:, 0]` — the CLS token — L2-normalises it, and feeds the 768-d vector to a scikit-learn `LogisticRegression` (lbfgs, C=1.0, max_iter=2000, `coef_` shape 6x768) unpickled from `classifier.joblib`. Six classes: db, df, rb, rf, vb, vf, derived literally from the sorted subdirectory names under `dataset/`.

Three reasons this beats fine-tuning here. First, the retrainable surface is 19,442 bytes versus a 346 MB backbone that never changes — small enough to diff and version-control, and retraining is one embedding pass plus an lbfgs fit, no training infrastructure. Second, DINOv2 is self-supervised on a huge corpus, so its features are already strong for "which template is this" — a linear probe over a strong frozen representation is the low-risk default, and the document is explicit that no fine-tuning code exists. Third, it is fully offline and CPU-capable: the hub id is commented out so there is no network call at request time, and device selection falls back to CPU automatically.

The honest caveat: I should not claim "small dataset" as the justification, because the dataset size and class balance are not documented anywhere. And the accuracy ceiling is real — if a linear boundary over frozen features stops separating two similar card backs, LoRA or a fine-tuned head is the next lever, at the cost of a 346 MB artefact and a training pipeline.

Project tie-in: Emphasise the artefact-size and retrain-cost argument, and the train/serve parity from sharing one `get_embedding`.

Follow-ups to expect:
- Why the CLS token rather than mean-pooled patch tokens?
- What does L2 normalisation buy you?
- When would you move to fine-tuning?

---

**Q16. Your classifier has `REJECT_THRESHOLD = 0.0` and a captured prediction at 0.7935 confidence. What is wrong with that and how do you fix it?**

Answer: It means the reject path is dead code — argmax always wins, so the service will confidently assign one of six classes to a photo of a coffee cup. The captured evidence makes it concrete: `db_1.png` and the single-page PDF rasterised from it both scored 0.9458, while page 2 of `two_page.pdf` predicted `vf` at 0.7935. That 0.7935 is exactly the case an operator should see flagged, and today it is indistinguishable from the 0.9458.

The fix has three parts. First, calibrate rather than guess: on the stratified 25% hold-out (`test_size=0.25`, `stratify=y`, `random_state=42`) plot the distribution of max-softmax for correct versus incorrect predictions and pick a threshold at an operationally acceptable error rate — for claims intake I would accept more "unknown" verdicts than silent misroutes, so I would set it where precision on accepted predictions is high. Second, add a genuine `unknown` outcome to the response contract rather than a low-confidence class label, so the caller has something to branch on. Third — the harder problem — logistic regression probabilities on frozen embeddings are not automatically well calibrated; I would check with a reliability diagram and apply Platt scaling or isotonic regression if they are skewed.

A complementary signal I already have available for free: the notebook `openclip_test.ipynb` implements a top-3-mean-cosine kNN over the same L2-normalised embeddings. Because the vectors are normalised, cosine is just a dot product, so an out-of-distribution image whose nearest training neighbours are all far away can be rejected on distance even when softmax is high — a cheap OOD gate that the linear head alone cannot provide.

Project tie-in: Present this as a known open item you can specify the fix for, not as a defect you missed.

Follow-ups to expect:
- Why is max-softmax a poor OOD detector?
- What is a reliability diagram?
- Would you reject per-page or per-document for multi-page PDFs?

---

**Q17. Walk me through the OpenCV preprocessing and quality-gating you built across these services. Where did the thresholds come from?**

Answer: Three distinct uses. In Multi-doc All-in-one OCR and Document Detection API I compute grayscale quality metrics per crop before extraction: Laplacian variance < 80 flags "Blurry Image", mean brightness < 50 "Too Dark", > 220 "Overexposed", standard deviation < 30 "Low Contrast", and width*height < 500,000 px "Low Resolution". These are returned as per-crop feedback so the intake channel can prompt re-capture rather than letting a bad scan fail downstream.

In Claim Duplication Detector API OpenCV does blank-page gating: grayscale read, `cv2.threshold` at 240, `cv2.countNonZero`, and a page is blank when the white-pixel ratio exceeds 0.99. That gate runs *before* perceptual hashing, so blank pages are neither hashed nor eligible as duplicate match targets — a cheap-to-expensive cascade, with the documented side effect that a page which is both blank and duplicate reports as blank.

In the KYC Document Validation System OpenCV handles ROI work: decode, crop by (x, y, w, h) with zero-dimension and out-of-bounds guards, and a resolution gate implemented as `min(w,h) < 480 OR max(w,h) < 640` — deliberately orientation-agnostic, even though the surrounding prose reads as a portrait-only rule, a discrepancy the documentation itself flags.

Where the thresholds came from is the honest part: they are engineering heuristics, not values tuned against a labelled set. Laplacian variance in particular is scale- and content-dependent — a sharp image of a sparse card can score below a blurry image of dense text — so the right version normalises by image size and text density, or replaces the fixed cut with a percentile learned from a corpus of accepted versus rejected uploads.

Project tie-in: Recite the constants confidently, then immediately own that none was calibrated and describe the calibration you would run.

Follow-ups to expect:
- Why Laplacian variance for blur specifically?
- What does a grey scanner background do to the 240/0.99 blank gate?
- Would you auto-enhance instead of rejecting?

---

**Q18. The Claim Duplication Detector also flags AI-generated or edited claim pages. Walk me through that detector, the forensic fallbacks, and how far you would trust the output.**

Answer: There are three independent signals, and they sit at very different confidence levels.

The primary one is a pretrained Hugging Face image classifier, `Ateeqq/ai-vs-human-image-detector`, run on PyTorch over every rendered page. The label set comes straight from `model.config.id2label` — I did not define the classes — and I surface the softmax maximum as a percentage string to two decimals alongside the label. So this is inference-only use of somebody else's model on a domain it was almost certainly not trained for: scanned medical claim invoices, not natural images.

The two fallbacks are classical forensics. For generation, a frequency-domain measure — spectral peak concentration divided by 0.62, clamped to 1.0 — on the theory that synthesised images have unusual periodic structure. For editing, Error Level Analysis: a second, independent ELA pass scored as `min(ela_score * 1.5, 1.0)` under a documented "ELA weight 0.5", and I will say plainly that the relationship between the stated 0.5 weight and the 1.5 multiplier is not explained anywhere, which is a documentation defect I own. Third, a `File_modified` flag derived from the PDF's `creationDate` versus `modDate`.

How far I trust it: as a triage hint for a human reviewer, not as a verdict. No accuracy on claim documents is recorded, the thresholds 0.62 and 0.5/1.5 were never calibrated against a labelled set, and `File_modified` is the weakest of the three — plenty of PDF producers rewrite both timestamps regardless of edits, and metadata is trivially forged. That is exactly why these are surfaced as flags in the Streamlit review UI (`claimapp.py`) rather than as an API verdict.

Two design points I would change. All three signals live only in the UI, not in the `POST /analyze` response contract, so a downstream intake system sees Unique/Blank/Duplicate and nothing else — the fields should be promoted into the API so the UI becomes a thin client. And the classifier runs on every rendered page, including blank ones, even though the blank gate already exists ahead of hashing; extending that gate to model inference is free saving on every batch.

Project tie-in: Lead with "pretrained model on an out-of-domain corpus, used as a triage hint" — the candour is the strongest part of this answer.

Follow-ups to expect:
- How would you build a labelled set of genuine versus edited claim scans?
- Why is ELA weak on re-compressed scans?
- Would you ensemble the three signals or keep them separate?

---

**Q19. You ran these vision models CPU-only in production. What did you actually do to make inference fast enough, and what would you do differently with a GPU budget?**

Answer: Several distinct levers, each tied to a project. Resolution control: YOLO at `imgsz=320` rather than 640, and in VL_version_OCR an explicit two-resolution split — pypdfium2 renders at 2.0x so detection and crops are sharp, then the crop is downscaled to 1536x1536 JPEG q80 before the VLM call, because payload size drives both latency and cost. In DOHA OCR AI I benchmarked a 150-DPI JPEG downsampling path for scanned PDFs at 3.4-3.8 s versus 7-11 s for raw upload with identical OCR accuracy.

Concurrency and engine reuse: in Multi-doc All-in-one OCR and Document Detection API crops are processed on a `ThreadPoolExecutor(max_workers=10)`, and RapidOCR is held in a thread-local singleton (`_ocr_thread_local`) so each worker owns its own ONNX session instead of serialising on a shared one — a mutex-contention fix, not just a parallelism win. VL_version_OCR fans out with `asyncio.gather` for the IO-bound VLM calls and uses 5- and 6-worker pools for the CPU-bound police parsers. In SLM AI OCR I added an explicit `gc.collect()` per request to stop transient image matrices accumulating in a long-running service.

Avoiding work entirely: the anchor gate in SLM AI OCR rejects the wrong document type *before* any LLM call, and Multi-doc filters pages before paying Azure.

With GPU budget: batch page embeddings per PDF in the Document Classification Model — the device switch already exists via `torch.cuda.is_available()` — serve the fine-tuned VLM under vLLM rather than llama.cpp Q4_K_S, and quantise or distil the detector. I would measure first: the only latency figures I have are SLM AI OCR's 5.07-19.09 s integration-test log against a "within 10 secs" design intent, and I never established which stage dominates.

Project tie-in: Use the SLM AI OCR latency log to show you know the target was missed and can name the suspects (Azure round-trip, LLM decode, translation).

Follow-ups to expect:
- Why thread-local ONNX sessions instead of a process pool?
- What is the memory cost of 10 workers each holding an OCR engine?
- How would you profile which stage dominates?

---

**Q20. Design question: you need to detect stamps and signatures on scanned claim documents — a new requirement. How would you approach it given what you have already built?**

Answer: I would start by asking whether this is detection or verification, because they need different systems. "Is a stamp present" is object detection and reuses everything I have; "is this the *right* stamp / is this signature genuine" is verification and needs a metric-learning setup closer to the ArcFace work in the KYC Document Validation System.

For presence detection, the fastest honest path is to extend the existing YOLOv8 pipeline rather than build a parallel one. The ingestion front end already exists: PyMuPDF/pypdfium2 rasterisation at 150 DPI or 2.0x, crop extraction, and a per-crop result contract with `status`, `feedback` and `confidence`. Adding `stamp` and `signature` classes means new labelled data and retrained weights, not new architecture — and the class-routing pattern is already there, since SLM AI OCR and VL_version_OCR both branch on the detected class.

The data problem is the hard part, and I have a template for it. In DOHA OCR AI I hit a training-distribution gap — the model missed the third vehicle party because every training sample was single-page — and fixed it with a synthetic pipeline that cloned real MOI reports, regenerated only digit-bearing rectangles, and produced 300+ balanced samples covering 1-7 vehicle layouts and cross-page splits. The same logic applies here: composite real stamps and signatures onto real document backgrounds at varied scale, rotation, opacity and overlap with printed text, so the detector sees the ink-over-text case that dominates in practice.

Evaluation would be recall-first at a low confidence threshold — a missed signature on a claim form is a compliance issue — with per-class precision reported separately, and a human review queue for anything under threshold, mirroring the `review_flags` contract from the Trade License Extractor.

Project tie-in: Anchor on the DOHA OCR AI synthetic-data story; it is the strongest evidence you can diagnose a data problem rather than reach for a bigger model.

Follow-ups to expect:
- How do you label an overlapping stamp-on-text region?
- Would signature verification need a siamese network?
- What false-positive rate is acceptable to a compliance team?


### Theme 3: LLMs, Prompt Engineering & Fine-tuning

**Q21. You have used three different mechanisms to force structured output from an LLM — Gemini's response MIME type, vLLM guided decoding with Pydantic, and OpenAI-style `strict` JSON schemas. Explain how they differ and when each is sufficient.**

Answer: These are three points on a spectrum from "ask nicely" to "constrain the decoder". In **Multilanguage Speech-to-Text Voice Data Capture** I used the weakest form: the google-genai SDK with a forced JSON response MIME type and the field schema written in prose inside the prompt. That guarantees the response parses as JSON but guarantees nothing about keys, types or required fields — which is exactly why `app.py` line 34 could swallow a malformed payload with a bare `except: pass`. That is a real defect; the fix is Pydantic validation with a repair-retry before the merge into session state, and surfacing `st.error` the way `chat_interaction.py` already does.

In **LLM Fine-tuning Platform** the teacher pipeline used vLLM's `response_format` with an actual Pydantic class (Pydantic 2.13.4) and read `choices[0].message.parsed`. That is grammar-constrained decoding — the sampler cannot emit a token that breaks the schema — so I dropped the earlier regex fence-stripping (`find("{") … rfind("}")`) that had been failing on unclosed brackets and markdown fences. Zero hallucinated keys, absent attributes explicitly nulled.

In **VL_version_OCR** I went further and made the schema dynamic: Stage 1 classifies the document against `DOCUMENT_ANCHORS`, Stage 2 synthesizes a strict Pydantic/JSON schema from `OUTPUT_DICT` for that exact document type, with `strict=True`, `temperature=0`, `additionalProperties=False`, plus a membership guard at `llm_engine.py:162` that drops any crop whose predicted type is not in the catalog.

Rule of thumb: MIME-type forcing for prototypes, decoder-level constraint whenever output feeds a downstream system of record.

Project tie-in: Lead with the VL_version_OCR two-stage strict-schema design; use the Speech-to-Text `except: pass` as your candid "what I'd fix" example.
Follow-ups to expect: What happens when the schema is satisfied but the values are wrong? Does constrained decoding hurt accuracy? How do you version a schema that downstream claims systems consume?

---

**Q22. Walk me through the teacher–student distillation pipeline in your fine-tuning project. Why distil at all instead of labelling by hand?**

Answer: **LLM Fine-tuning Platform** (doc AT/UNSLOTH/V1) is a five-stage pipeline: document capture → teacher VLM → human audit → QLoRA fine-tune → on-prem serving. The teacher is Qwen3.5-122B-A10B GPTQ-Int4 on vLLM 0.27.1 (port 8001), run at temperature 0.0, top_p 1.0, `enable_thinking=False`, `cache_prompt=True`, 60 s timeout with 2 retries, decoding into six Pydantic schemas covering six UAE and three Doha card categories. It processed 553 images in 18 min 41 s — 2.03 s/image (notebook cell 127).

The economic argument for distillation is that correcting a prepopulated field is far cheaper than transcribing it from scratch. The critical caveat is that it is only safe behind a quality gate: my Phase 1 forensic audit (`UAE_Card_Extraction_Audit.md`) found 12 of 30 flagged field labels wrong — a 40% error rate on flagged fields — including a day/month swap (`1984-11-03` vs `11-03-1984`), a unit letter "K" contaminating a weight field, garbled Arabic place-of-issue, and two duplicate images. Training on that would have taught the student to reproduce errors confidently.

So I built the Streamlit 1.60.0 review dashboard (`demo.py`) as a mandatory gate: session-state login, real-time diffs, `edited_by` / `edited_fields` provenance stamps, and save-before-navigate inside `move(delta)` so a reviewer clicking Next never loses an edit. Three reviewers verified 664 UAE and 296 Doha records with zero data loss. Only then did the corpus become Unsloth `messages` format, stratified 85/15 into 564 train / 100 validation, and get pushed as Parquet shards to Hugging Face Hub.

Project tie-in: Frame the audit as the decision that made the whole project defensible — the 40% flagged-field figure is your strongest concrete number.
Follow-ups to expect: How did you decide which fields to flag for review? Would cross-teacher voting have replaced the human gate? Why a 122B teacher rather than a mid-size one?

---

**Q23. Your Run 1 fine-tune failed with catastrophic memorization. Diagnose it for me, and tell me what you changed.**

Answer: Run 1 (18 Aug 2026) of **LLM Fine-tuning Platform** produced a model that, given `dba_10.jpeg` — a driving-license *back* containing no personal names — emitted "Waheed Ahmad Khan Ghalib", the target from training row 115. That is the classic signature of a model that has stopped conditioning on the image and is reproducing memorized text.

Three causes compounded, recorded in `FINETUNE_REPORT.md`. First, `finetune_vision_layers: true` meant the vision tower drifted during training, and the merged model was then quantized and served against the *stock* GGUF `mmproj` — trained vision LoRA against an untrained multimodal projector, so the visual pathway was effectively broken at inference. Second, 5 epochs on ~564 rows with no validation split and no early stopping gave nothing to detect the divergence. Third, Unsloth Studio's auto-mapper misread the 3-column format, so loss was not being computed where I assumed.

The redesign in `unsloth_config_v2.yaml`: freeze the vision layers (`finetune_vision_layers: false`) so the stock `mmproj` stays valid; `train_on_completions: true` so loss is masked to the assistant JSON only and the model is not rewarded for reciting prompts; a constant six-schema catalog prompt so the student cannot condition on prompt variation as a shortcut to the answer; 2 epochs instead of 5; and an actual evaluation split. LoRA config stayed r=16, alpha=16, dropout 0.0, lr 1e-4, adamw_8bit with linear decay, batch 2 × accumulation 4 (effective 8), targeting q/k/v/o/gate/up/down_proj under 4-bit BitsAndBytes.

Honest caveat: the v2 evaluation result is not recorded in my documentation, so I would not claim a measured improvement — the fix is an exact-match harness on the 100-row validation split.

Project tie-in: This is your best "engineering judgement under failure" story — lead with the symptom, then the three-cause diagnosis.
Follow-ups to expect: How would you export a matching custom mmproj instead of freezing? What eval metric would have caught this at epoch 1? Why r=16?

---

**Q24. What does QLoRA actually do, and justify your hyperparameters.**

Answer: QLoRA quantizes the frozen base weights to 4-bit (BitsAndBytes NF4-style) and trains only low-rank adapter matrices injected into the linear projections, with an 8-bit optimizer to shrink optimizer state. Base weights are never updated, so gradients flow through a dequantized forward pass into a handful of small matrices — that is what makes a ~4.6B-parameter multimodal model trainable on a single GPU.

In **LLM Fine-tuning Platform** I fine-tuned Google Gemma 4 E2B with Unsloth 2026.8.18 / TRL 0.24.0: rank 16, alpha 16 (so scaling α/r = 1, a neutral, well-behaved default), dropout 0.0, adapters on all seven projections — q, k, v, o, gate, up, down — rather than attention only, because the extraction task changes what the MLP blocks emit, not just where attention looks. Learning rate 1e-4 with linear decay under adamw_8bit; per-device batch 2 with gradient accumulation 4, i.e. effective batch 8, which is what fit alongside vision soft tokens plus JSON targets at max sequence length 2048 (YAML) / 3072.

Rank 16 is a deliberate capacity choice: the task is a narrow, closed-vocabulary field-extraction mapping over nine card categories, not new world knowledge, so a higher rank mostly buys memorization risk — which Run 1 demonstrated. Vision layers were frozen in v2 for the `mmproj` compatibility reason.

The honest gap: I do not have an ablation. The document records no whole-corpus exact-match accuracy, so I would present these as principled defaults validated by not failing, and say the next step is sweeping rank ∈ {8, 16, 32} against exact-match on the held-out 100 rows.

Project tie-in: Emphasise the "narrow task ⇒ low rank" reasoning and tie r=16 back to the memorization failure.
Follow-ups to expect: Why 8-bit AdamW over paged AdamW? What breaks if you merge LoRA into a 4-bit base? Would full fine-tuning have been better here?

---

**Q25. How did you turn a corrected annotation corpus into a training dataset, and what could go wrong in that step?**

Answer: In **LLM Fine-tuning Platform** the annotation store is JSONL with `image_path`, `document_type`, and `target.fields`, plus the audit stamps `edited_by` / `edited_fields` written by the Streamlit tool. Converting that into training data meant restructuring each record into Unsloth's multimodal `messages` format — a system/user turn carrying an explicit image token reference and the schema catalog, and an assistant turn containing the strict JSON target — then materialising `messages` and `images` columns as Arrow/Parquet shards pushed to Hugging Face Hub (`Peterleo5/uae-ocr-test`). The split is stratified 85/15 by document type, giving 564 train / 100 validation — a row count that matches the 664 human-verified UAE records exactly, with the 296 Doha records audited in the same tool — so every document type in the split appears on both sides.

The failure modes I actually hit or guarded against: duplicate images (the audit found two cards photographed twice) which leak across the split and inflate any eval; unresolvable image paths, which is why path resolution is an explicit input contract; and label-side contamination — the day/month swaps and the "K" unit letter — that no amount of training discipline recovers from. Legacy `resident_back` fields (`date_of_birth`, `sex`, `exp`) are ~96% null on newer cards, so those columns are near-constant and a model can learn to always emit null; that is a distribution fact worth stating rather than hiding.

The other subtle one is prompt variation. In Run 1 the prompt varied per document type, which let the model use the prompt as a shortcut to the answer. v2 pins a constant six-schema catalog prompt so the image is the only distinguishing signal.

What I would add: perceptual-hash dedup before splitting, and rule-based validators for dates, VINs and plate formats as a pre-training lint pass.

Project tie-in: Stress dedup-before-split and the constant-prompt change; both are dataset decisions, not training decisions.
Follow-ups to expect: How do you stratify when one class has very few examples? How do you version datasets for reproducibility? Would you keep the 96%-null legacy fields in the schema?

---

**Q26. In the Speech-to-Text project you used the LLM for conversational field collection, not just extraction. How was that prompt designed?**

Answer: **Multilanguage Speech-to-Text Voice Data Capture** has two apps with two different prompting strategies. `app.py` is one-shot: browser-captured WAV bytes plus a schema-specifying prompt go to gemini-3.1-flash-lite-preview in a single multimodal call, and thirteen vehicle-insurance attributes come back as JSON into an editable `st.data_editor` table (Field column disabled so the schema cannot drift, Value editable for human correction).

`chat_interaction.py` is the conversational one, on gemini-2.0-flash-lite. The design decision that makes it work is that the model is stateless about slots — I serialise the *current memory dict* as JSON into the system prompt on every turn. So the model is never asked to remember; it is shown what is already filled and instructed to ask only for what is missing. The reply contract is three keys: `response` (the spoken-style question or acknowledgement), the field values it managed to extract this turn, and a boolean `completed`. I read completion with `data.get('completed', False)` so a missing key degrades to "not done" rather than prematurely closing the conversation.

The merge semantics matter as much as the prompt: only truthy values overwrite, so a later recording adds or corrects without wiping earlier fields, and Streamlit's re-execution model is handled by the `audio_bytes != st.session_state.last_audio` dedup gate so one recording is never billed twice.

The honest limitation: this is prompt-level slot filling with no external slot tracker and no schema validation, so an empty string returned for a field the user tried to *clear* will not clear it. Function calling with a deterministic field-update handler is the cleaner design.

Project tie-in: Lead with "memory lives in the prompt, not the model" — it is the crisp architectural statement here.
Follow-ups to expect: How do you stop the model re-asking for a field it already has? Why not function calling? How do you handle a user correcting a previous answer?

---

**Q27. Explain your on-prem serving stack: vLLM for the teacher, llama.cpp for the student, GGUF quantization in between. Why that split?**

Answer: Different workloads, different engines. In **LLM Fine-tuning Platform** the teacher is a 122B GPTQ-Int4 model running on vLLM 0.27.1 (`max_model_len` 100,000) doing high-throughput batch labelling — vLLM's continuous batching and paged attention are exactly right for grinding through 553 images at 2.03 s each, and its guided-decoding support gave me Pydantic-constrained JSON for free.

The student is the opposite profile: single-request, latency-sensitive, must live on-premise for PII. I merged the LoRA adapters, exported to GGUF Q4_K_S (`gemma-4-E2B-it-Q4_K_S.gguf` — 4,647,450,147 params, 3,028,113,548 bytes on disk, `n_ctx` 65,536, `n_vocab` 262,144), and served it via llama.cpp with a multimodal `mmproj` sidecar at `http://172.20.132.97:8000/`. Q4_K_S is a k-quant that keeps a mixed-precision arrangement across tensor types; it is the accuracy/footprint compromise that fits a ~4.6B multimodal model into ~3 GB.

The design decision I care about most is that both engines expose the *same* OpenAI-compatible `/v1/chat/completions` contract, so client code is interchangeable — swapping teacher for student is a base-URL change, which is what makes an A/B evaluation cheap. I verified the deployment rather than assuming it, with `GET /v1/models` (notebook cell 237) returning capabilities `['completion','multimodal']`.

Two candid gaps: the llama.cpp host runs with `Bearer none` and hard-coded IPs, and I have no measured accuracy delta from quantization. A thin FastAPI gateway for auth, rate limiting and request logging, plus an exact-match comparison of FP16 versus Q4_K_S on the validation split, are the obvious next steps.

Project tie-in: The "same OpenAI contract on both ends" point is the reusable architectural idea — state it explicitly.
Follow-ups to expect: What does Q4_K_S cost you in accuracy? Why not serve the student on vLLM too? How would you A/B teacher vs student in production?

---

**Q28. You used LangChain in some places and the raw SDK in others. Justify that.**

Answer: In **VL_version_OCR** this is explicit product surface, not accident: `services/llm_engine.py` exposes six execution modes — three models (Qwen3-VL-4B-Instruct on a local vLLM host, Gemini-3-Flash-Preview via Google GenAI, Dev-Llama-4-Scout-17B-16E-Instruct on Azure AI Services) × two paradigms (AsyncOpenAI and LangChain Core), selected per request via a required `model_name` enum. The reason two paradigms exist is historical and honest: I had duplicated Qwen prototypes (`qwen_langchain.py`, `qwen_openai.py`) and unified them behind one `config_dict`-driven engine rather than throwing one away, because LangChain's provider adapters made Gemini and Azure integration cheap while the raw AsyncOpenAI path gave tighter control over the strict-schema `response_format` and lower overhead for the local vLLM route.

**LLM Fine-tuning Platform** shows the same duality with observable consequences: the OpenAI SDK 2.52.0 client runs at top_p 1.0 with a 60 s timeout and 2 retries, while the LangChain Core 1.5.4 / LangChain OpenAI 1.5.0 client runs at top_p 0.1 with 120 s. Those are different decoding and reliability behaviours against the *same* endpoint — a real maintenance hazard, and a good example of why abstraction layers should not silently redefine sampling parameters.

The trade-off, stated plainly: LangChain buys provider portability and less glue per new vendor; it costs you an indirection layer that can mask exactly the parameters that determine determinism, and it doubles your test surface. Given that **P&C Clause Recommendation Engine** and **Speech-to-Text** both call their providers directly, my default going forward is a thin in-house adapter interface with one canonical decoding config, and reach for LangChain only where its integrations save real work.

Project tie-in: Own the top_p 1.0 vs 0.1 drift as the concrete evidence — it makes the trade-off argument credible.
Follow-ups to expect: How do you test six execution modes without a test suite? What would your adapter interface look like? Where does LangChain genuinely pay for itself?

---

**Q29. How do you evaluate LLM extraction output when you don't have ground-truth labels at scale?**

Answer: For LLM output specifically I do not think "no labels" excuses "no measurement" — it changes which measurements you take. There are three tiers, and I have built the cheapest ones.

The tier that needs no labels at all is structural: strict decoding should make schema-valid output ~100%, so any non-zero invalid rate is a decoder or prompt regression signal, and I log it. Next is grounding — does every emitted value actually appear in the source? I implemented exactly that check in `rag_extractor/llm.py` in the **Trade License Extractor**, and it catches the failure mode constrained decoding cannot: a well-formed JSON object full of invented values. Third, and the clever one, is the reviewer edit rate: the **LLM Fine-tuning Platform**'s annotation tool already stamps `edited_by` and `edited_fields` on every corrected record, so per-field edit frequency is a continuous accuracy proxy nobody was reading.

Only the top tier needs ground truth, and there I built the right shape. In **Trade License Extractor** I wrote an evaluation harness — `eval/run_eval.py` with a `values_match()` comparator — over a 16-document ground-truth corpus, reused across both the deterministic pipeline and the RAG pipeline, and I measured latency per document class against a 20-second operational budget (65–130 ms born-digital, 6–10.7 s single-page OCR, 12–13.5 s for 11- and 14-page raster bundles). The comparator is the interesting part: naive string equality over-penalises formatting, so date normalisation to ISO and whitespace/case tolerance have to live inside `values_match()`, not in the model.

Where I have to be candid is that the labelled tier was never run to a number: the **LLM Fine-tuning Platform** has a 100-row stratified validation split and no recorded v2 evaluation result, and **VL_version_OCR** records no accuracy figure at all — every number in that project is a configuration constant. I would rather state that than quote something I did not measure.

In production I would add per-field null rate and per-document-type rejection rate as drift alarms, and I would deliberately not reach for LLM-as-judge here: on identity documents the judge shares the extractor's failure modes on Arabic and on look-alike fields like Chamber No versus Commercial Register No, so it would agree with the mistakes.

Project tie-in: Use the `edited_fields` audit stamps as the clever answer — the annotation tool already emits an accuracy signal nobody is reading.
Follow-ups to expect: How do you handle fields that are legitimately null? What is your sample size for a meaningful exact-match estimate? How do you catch regressions after a model swap?

---

**Q30. Deterministic parsing, a small local LLM, and a large cloud VLM are all options for extraction. Give me your decision framework, using your own projects.**

Answer: My framework is: use the cheapest mechanism that meets the accuracy bar, and let evidence — not preference — move you up the ladder.

**Trade License Extractor** sits at the deterministic end deliberately. Born-digital licence pages parse in 65–130 ms through PyMuPDF with label-anchor extraction across nine UAE authority templates; OCR fires only when a character-count sufficiency check or an Arabic-sanity lexicon check fails. The LLM (Ollama qwen2.5:3b-instruct-q4_K_M via httpx `/api/chat` with `format='json'`) is a guard-railed *fallback*, restricted to unknown templates and low-confidence fields, and every LLM-sourced value lands in `review_flags`. Crucially I moved *down* the ladder on evidence: a vision-LLM path returned DCCI/Chamber No as the Commercial Register number and Fax as Mobile on this corpus, so I replaced it with deterministic negative-label guards.

**VL_version_OCR** sits at the other end because the problem is different: composite scans with multiple ID cards per page, 18+ document types across five jurisdictions, six passport nationalities. Writing per-template parsers there does not scale, so single-pass image-grounded VLM extraction with dynamically synthesized strict schemas wins — and Azure Document Intelligence was abandoned specifically because it struggled on multi-document scans.

**LLM Fine-tuning Platform** is the third answer: when you need VLM-class capability but cannot pay per call or export PII, distil a big teacher into a small student and serve it locally.

The axes I weigh: template stability (stable ⇒ deterministic), latency budget, per-call cost at volume, data residency, and auditability — a regex has provenance, a VLM needs a grounding check. Where I have hard latency data, the deterministic path is two orders of magnitude cheaper, which is why it stays the default.

Project tie-in: The DCCI/Fax confusion is your proof that you demote LLMs on evidence, not dogma — it reads as senior judgement.
Follow-ups to expect: At what template count does deterministic parsing stop paying? How do you decide the confidence threshold that triggers fallback? Would you re-test the VLM on that corpus today?


### Theme 4: NLP, Classical ML & Similarity/Recommendation

**Q31. Several of your systems classify documents with fuzzy keyword matching rather than a trained text classifier. Why, and when does that choice stop being defensible?**

Answer: In the Document Detection API (TextMatch Classification) and Multi-doc All-in-one OCR I classify document type with RapidFuzz over 1-3 word n-grams generated from normalised OCR text, using `ratio` for single tokens and `token_sort_ratio` for phrases at a cutoff of 80, scored as a percentage of matched keywords per label against a per-region JSON dictionary (`labels_DB/<region>_labels.json` for UAE, DOHA, OMAN, KUWAIT). The reason is operational rather than statistical. There was no labelled corpus for any jurisdiction, the label set is small and visually distinctive, and every rule has to be readable and editable by non-engineers — which is exactly what the TextMatch Label Trainer exists for. A rule change is a JSON write that the FastAPI services hot-reload with no restart, so onboarding Oman or Kuwait is a config exercise, not a retraining cycle. `token_sort_ratio` matters because OCR reorders and fragments multi-word headers; `ratio` on single tokens avoids the substring permissiveness of `partial_ratio`.

Where it stops being defensible: the score is keyword coverage, so a term that appears on every document scores as well as one unique to a label — frequency is not discriminativeness. Confusable templates (Sharjah old vs new police report) and OCR-degraded scans are exactly the cases fuzzy overlap handles worst, and I have no confusion matrix to prove otherwise. Honestly, the thresholds carry the biggest debt: the PoC notebook tested a 50% cutoff, production enforces 65% for suggestions and 90% for confidence warnings, and no recalibration is recorded. Once a labelled set exists per region, I would move to TF-IDF or embedding features with a linear classifier, keeping the keyword rules as an explainable fallback and a cold-start path for new jurisdictions.

Project tie-in: Anchor on the Document Detection API plus the TextMatch Label Trainer, and be explicit that the rules were a deliberate no-labels, non-engineer-editable choice — not a shortcut.

Follow-ups to expect:
- How would you make the score discriminative rather than frequency-based?
- What would your first evaluation set look like?
- Why `token_sort_ratio` and not `partial_ratio` or `token_set_ratio`?

---

**Q32. Walk me through how the TextMatch Label Trainer decides a keyword set is good enough to go live.**

Answer: The Label Trainer is a Streamlit tool that binds a session to one jurisdiction through a non-dismissible `@st.dialog('Select Region')` modal, so every read and write targets one `labels_DB/<region>_labels.json`. An operator uploads real sample documents (JPG/PNG/PDF, PDF page 0 rendered by PyMuPDF at 150 DPI and JPEG-encoded at quality 95), picks an OCR engine — local RapidOCR or Azure Document Intelligence prebuilt-read, captioned with the speed/accuracy trade-off — and gets automated suggestions from `suggest_keywords()`: 3/2/1-gram harvesting, minimum token length 6, appearing in at least 2 documents, then near-duplicates merged with RapidFuzz `partial_ratio` at 80.

The gate is `extract_and_score()`. Each uploaded sample is scored by `keyword_score()` — a keyword counts as matched at RapidFuzz `ratio` >= 80, and the score is matched/total * 100. Those per-file scores are averaged, and the label/keyword mapping is written to the regional JSON only if the average clears the slider threshold, defaulted to 75% after 50/65/75 trials in `test.ipynb` cells 63-66. Because the FastAPI detection and extraction services read the same file, the accepted write is the deployment.

I would flag three real weaknesses. Coverage is blind to match quality once past 80, so one dead keyword costs a fixed fraction regardless of importance. The value 80 does the work of two different jobs — keyword matching and suggestion clustering — with no evidence they should share a constant. And PDFs are only evaluated on page 0. The fixes are per-keyword diagnostics showing which term failed, separate constants, and multi-page sampling.

Project tie-in: This is the strongest "human-in-the-loop, no labelled data" story you own — present the threshold gate as governance, not as a model.

Follow-ups to expect:
- What stops an operator writing a weak rule that passes on three cherry-picked samples?
- How would you version or roll back a label set?
- Why is there no concurrency control on that JSON file?

---

**Q33. Explain perceptual hashing and why you chose it for duplicate claim pages instead of checksums or embeddings.**

Answer: In the Claim Duplication Detector API each PDF page is rendered by PyMuPDF at 2x (`fitz.Matrix(2, 2)`) to PNG, then hashed with `imagehash.phash`. pHash downscales to greyscale, takes a DCT, keeps the low-frequency coefficients and thresholds them against their median, producing a bit string that is stable under resampling, mild compression and small tonal shifts. Two pages match when Hamming distance is < 5.

The business case rules out checksums: claim documents get re-scanned, re-exported and re-printed, so bytes and PDF metadata differ on every copy while the visual content is identical. Content-based matching is the point. Embeddings (CLIP or a CNN) would tolerate crops and rotations that pHash will not, but they cost a model load per worker, GPU or slow CPU inference per page, and a vector index — heavy for a stateless service whose selling point is seven dependencies, one module, no database, running behind `--workers 4`.

Design detail worth noting: blank detection runs first (OpenCV greyscale threshold 240, white pixel ratio > 0.99) and blank pages are neither hashed nor added to the index, so a cheap gate keeps the expensive comparison clean — with the documented consequence that a page which is both blank and duplicate is reported as blank. Matching uses a request-scoped `global_hashes` map spanning every non-blank page of every file in the batch, first match wins.

The honest limitations: distance 5 was never validated against a labelled set; comparison is linear against all prior hashes, so a large batch is quadratic; and with no persistence, a duplicate resubmitted next week is invisible. Fixes are a labelled threshold sweep, a BK-tree for sub-linear Hamming search, and a persistent index keyed by claim or member ID.

Project tie-in: Lead with the re-scan failure mode that checksums miss — it is the concrete reason pHash was correct here.

Follow-ups to expect:
- How would you pick the Hamming threshold empirically?
- What duplicates does pixel matching miss entirely?
- How does a BK-tree beat linear scan here?

---

**Q34. Embeddings versus keyword/fuzzy methods — how do you decide, given you shipped both?**

Answer: I have shipped both and the deciding factors were labelled data, explainability and the shape of the text. In the P&C Clause Recommendation Engine I embed business activity descriptions (`AC_DESC`) with SentenceTransformers `all-MiniLM-L6-v2` and index them in FAISS `IndexFlatL2`, because underwriting submissions paraphrase the same risk class in dozens of ways and lexical overlap fails on them; retrieval is exact-match first, with a Top-N (1-20) nearest-neighbour fallback when fewer than 50 exact matches exist. In the Document Detection API and Multi-doc All-in-one OCR I use RapidFuzz keyword scoring, because the discriminating signal is short printed headers, the text arrives OCR-noisy at the character level rather than paraphrased at the semantic level, and the rules must be auditable and editable per jurisdiction.

That is the general rule: dense embeddings when the variance is semantic and you can tolerate an opaque score; fuzzy lexical matching when the variance is character-level noise and someone must be able to read and change the rule. Cost matters too — MiniLM plus FAISS needs a model, an index and a rebuild strategy, while RapidFuzz is a function call.

I have also rejected embeddings on evidence. In the KYC Document Validation System I prototyped `all-mpnet-base-v2` for semantic field matching (notebook cell 21) against RapidFuzz `partial_ratio` (cell 20) and shipped RapidFuzz: identity fields are short lexical tokens on a CPU-only box. Honest caveat — neither cell stored an output, so that was a judgement call, not a measured comparison, and I would redo it with a small labelled field set.

For a rebuild I would test hybrid retrieval with rank fusion, which is exactly the direction the Trade License RAG sub-package took with semantic plus BM25 and RRF.

Project tie-in: Contrast P&C (semantic) with the Document Detection API and KYC (lexical) — having chosen each direction deliberately is the answer.

Follow-ups to expect:
- Why `IndexFlatL2` and not IVF or HNSW?
- When would you move from MiniLM to a larger encoder?
- How do you evaluate retrieval quality with no ground truth?

---

**Q35. Take me through the hybrid Recency x Frequency score in the P&C Clause Recommendation Engine.**

Answer: The engine helps commercial property underwriters spot missing risk clauses. Given a business activity, it retrieves historical records — exact match on activity description first, FAISS nearest neighbours over MiniLM embeddings when fewer than 50 exact matches exist — then ranks the clauses attached to those records by a Recency Score multiplied by a Frequency Score, normalised to 0.0-1.0. Recency comes from `TPI_UW_YEAR`, frequency from clause occurrence counts. The output buckets into Highly_Recommended, Review_Required and Optional, each item carrying `{clause, confidence, reason}`, where the reason cites the year and occurrence count.

The product form is the design decision I would defend. A weighted sum lets one signal carry a clause on its own: pure frequency keeps recommending terms the book has moved away from, pure recency promotes one-off clauses from last year's odd submission. A product demands both — a clause must be both prevalent and still in current use — which matches how an underwriter actually reasons about wordings.

Two things I would be candid about. The exact normalisation and the tier boundaries are not written down in the project documentation, so I should be able to state them from the code rather than the doc, and the fact that they were set by inspection rather than calibration is a gap. And low-count clauses are unstable: three occurrences in one recent year can outscore a well-established clause. Bayesian shrinkage toward the portfolio mean, or exponentially decayed frequency instead of a separate recency term, would fix that.

The measurement gap is the real one: no accuracy, precision@k or acceptance figures were captured. The natural evaluation is underwriter acceptance rate and precision@k against clauses actually bound on subsequent policies.

Project tie-in: Explain the product-versus-sum choice in underwriting language — that is what makes it a domain answer rather than a formula answer.

Follow-ups to expect:
- How would you handle a clause with a single occurrence?
- Where do the tier thresholds come from?
- Would you learn the ranking from underwriter acceptance data?

---

**Q36. How do you handle noisy OCR text before it reaches a matcher or classifier?**

Answer: Preprocessing is deliberately shallow and matcher-specific, because heavy normalisation destroys the signal the matcher relies on. In the Document Detection API and Multi-doc All-in-one OCR the OCR text is normalised, then expanded into 1-3 word n-grams so a keyword can match a phrase that OCR fragmented; matching itself absorbs character errors via RapidFuzz at cutoff 80 rather than trying to repair the text. In the KYC Document Validation System I do the opposite order: RapidFuzz `partial_ratio` at 70 pre-corrects the RapidOCR token list against the operator's keywords, then matching is a plain case-insensitive substring test with a 65% pass rule (`ceil(0.65 * N)`). So one system tolerates noise at match time, the other repairs before matching.

Arabic needs its own handling and it is not generic cleanup. In DOHA OCR AI the MOI police-report parser works on the PDF text layer where RTL bidirectional wrapping breaks naive search, so I wrote `transfer_window()`, `fix_number_wrap()` and `is_label()` plus lookaround-guarded regex anchors (`INCIDENT_NUMBER`, `PLATE`, `CHASSIS`) to reassemble values, gated by an Arabic report-header check. In UAE OCR AI I keep two forms of every Arabic string — unshaped for translation and catalogue lookup, reshaped via `arabic-reshaper` and `python-bidi` for display — because shaped presentation forms will not match a lookup table. In the Trade License Extractor an Arabic-sanity lexicon check is one of the two gates that decide whether a page needs OCR at all.

The honest gap: I do not measure preprocessing in isolation. I would build a small noisy/clean parallel set and measure match rate per stage, because right now the 70/80/65 constants are heuristics with no ablation behind them.

Project tie-in: Use the Arabic dual-form rule (unshaped for lookup, reshaped for display) — it is a specific, non-obvious detail.

Follow-ups to expect:
- Why pre-correct in KYC but tolerate noise in the Document Detection API?
- What breaks if you lowercase and strip punctuation aggressively?
- How would you ablate each preprocessing step?

---

**Q37. You set a lot of thresholds — 80, 65, 90, 75, distance 5, 70. How were they chosen and how would you defend them?**

Answer: Honestly, most were chosen by inspection on small samples, and I would rather say that than dress them up. There is one exception with real evidence: in the Label Trainer the 75% acceptance default came from testing 50%, 65% and 75% cutoffs for false-acceptance behaviour in `test.ipynb` cells 63-66, and I shipped 75 with a 0-100 slider so operators can move it per label set. Everything else is a heuristic: RapidFuzz cutoff 80 in the Document Detection API and Multi-doc All-in-one OCR, 65% suggestion and 90% confidence thresholds in production, `partial_ratio` 70 and the 65% keyword pass rule in the KYC system, and pHash Hamming distance < 5 in the Claim Duplication Detector.

There is also a documented drift I own: the PoC notebook tested a 50% classification cutoff while production enforces 65% and 90%, with no recalibration recorded between them. That is a real defect, not a nuance, and I flagged it in the project documentation rather than hiding it.

What defending them properly looks like: for each decision, build a modest labelled set — say a few hundred crops per region for classification, a few hundred page pairs for duplication — sweep the threshold, and plot precision against recall to pick an operating point that matches the cost asymmetry. In claims duplication a false positive blocks a legitimate invoice and a false negative pays twice, so those errors are not symmetric and the threshold should reflect that. In document classification a wrong type contaminates enterprise records, so I would bias toward routing to human review over confident misclassification.

I would also instrument production: log the score distribution and the accepted/rejected verdict per request so thresholds can be re-tuned against real traffic instead of a notebook.

Project tie-in: Lead with the Label Trainer calibration (real evidence), then own the 50 -> 65/90 drift openly.

Follow-ups to expect:
- Which of your thresholds is riskiest and why?
- How do error costs differ across your systems?
- How would you monitor threshold drift in production?

---

**Q38. Talk about class imbalance and skewed data in your work — where did it bite you?**

Answer: The sharpest case was distributional, not class-frequency, imbalance. In DOHA OCR AI the early Azure Document Intelligence models failed to extract the third vehicle party on multi-page MOI accident reports. That was not a model-capacity problem — the training samples were single-page only, so multi-party and cross-page layouts were absent from the distribution entirely. I diagnosed it as a training-distribution gap and fixed it on the data side: a synthetic generation pipeline producing 300+ balanced samples covering 1-7 vehicle layouts, cross-page injury-table splits and scan-style raster twins. That is deliberately balanced coverage of the rare configurations, not more of the common one.

Getting there took an iteration worth telling: v1/v2 of the generator redacted and reinserted whole text spans with Arabic reshaping and broke on mixed Arabic encodings (logical vs visual order vs presentation forms). v3 regenerates only digit-bearing rectangles — IDs, VINs, dates, plates — preserving Arabic typography and vector borders pixel-for-pixel, which keeps field-level label alignment intact.

The other imbalance is one I have not solved. In the fuzzy classifiers, document types are not equally represented in real intake and the score is keyword coverage, which has no notion of a prior — a rare template with few distinctive keywords loses to a common one with many. In the LLM fine-tuning project the analogue was legacy `resident_back` fields being 96% null on newer cards, so a model can score well by predicting null.

Practically: I would report per-class recall rather than aggregate accuracy, stratify any evaluation split (as I did with the 85/15 split in the fine-tuning work), and keep generating targeted synthetic coverage for rare layouts rather than reweighting a loss I do not control.

Project tie-in: The 3rd-vehicle-party diagnosis is the anchor — it shows you debugged a model failure to a data cause.

Follow-ups to expect:
- How did you validate the synthetic samples were realistic?
- Why per-class recall over accuracy here?
- What would you do if you could not generate synthetic data?

---

**Q39. Design question: you have three annotation and review tools across these projects. How would you design a labelling workflow that actually improves the models?**

Answer: I have built three, each solving a different piece. The Label Trainer is a rule-authoring console with an empirical acceptance gate — operators write keywords, the tool scores them against real uploads and refuses to persist below threshold. The KYC system's ROI tool lets an operator drag a region on a canvas, scales the selection to original coordinates, crops with OpenCV under zero-dimension and out-of-bounds guards, and OCRs the field — human-directed field-level extraction without a layout model. The LLM fine-tuning project's Streamlit dashboard is closest to a real labelling loop: reviewers correct teacher-VLM predictions with real-time diffs, provenance stamps (`edited_by`, `edited_fields`) and save-before-navigate, and it produced 664 UAE and 296 Doha verified records with zero data loss.

The lesson that generalises came from the audit before that loop: 12 of 30 flagged teacher fields were wrong — a 40% error rate on flagged fields, including a day/month swap, unit contamination and duplicate images. Pre-populated labels are cheaper to correct than to transcribe, but only behind a mandatory human gate; without it you train on the model's own errors.

A proper workflow would add three things I did not build. Active learning: prioritise the queue by model uncertainty and disagreement rather than file order, so reviewer time buys the most information. Agreement measurement: double-annotate a slice and track inter-annotator agreement, because provenance stamps tell you who edited, not whether they were right. And a closed loop: every correction becomes a versioned dataset row and every model release is evaluated on a frozen held-out split — my 85/15 stratified split pushed to the Hub was the start of that, but the evaluation result on the redesigned recipe was never recorded, which is exactly the gap that lets a regression ship.

Project tie-in: Sequence the three tools as an evolution ending at the fine-tuning dashboard, and lead with the 40% flagged-field audit finding.

Follow-ups to expect:
- How would you rank items for review by uncertainty?
- How do you catch a reviewer who is systematically wrong?
- What does the frozen evaluation set look like?

---

**Q40. Trade-off question: your recommendation engine runs an empirical pipeline and an LLM pipeline in parallel, both returning a confidence. Defend that.**

Answer: The P&C engine exposes `/historical_response`, driven by FAISS retrieval plus the Recency x Frequency score, and `/AI_response`, driven by Google Gemini constrained to an `AIRecommendation` JSON schema. Both return the same three-tier `{clause, confidence, reason}` shape, so the Streamlit UI and any downstream underwriting platform consume one contract.

The rationale is that the two legs answer different questions. The historical leg answers "what does this book actually attach to this risk class", which is auditable, cheap, deterministic and defensible to a compliance reviewer because the reason cites year and occurrence count. It is structurally incapable of recommending a clause the portfolio never contained. The Gemini leg answers "what exposures does this business description imply", which covers novel risks but has no portfolio grounding and can name a clause that does not exist in our wordings library. Keeping them separate means an underwriter can see which kind of evidence supports each suggestion.

The weakness I would concede without prompting: both legs emit a field called `confidence`, and nothing shows they are on the same scale. One is a computed statistic, the other is an LLM's self-report. Presenting them identically invites a reader to compare them, which is not valid. I would either label them distinctly or calibrate the LLM score against portfolio outcomes before letting it share the name.

The design I would move to is fusion: feed the historical shortlist into the prompt so the model reasons over real clause names and cannot hallucinate a wording, then return one merged list annotated with which evidence supported each item. That constrains the generative leg and gives underwriters a single ranked list rather than two they must reconcile.

Project tie-in: Own the shared-`confidence` flaw before the interviewer finds it, then offer the fused RAG design as the fix.

Follow-ups to expect:
- How would you calibrate the LLM confidence?
- What stops the historical leg recommending clauses already on the policy?
- What does the fused pipeline cost in latency and API spend?


### Theme 5: System Design, APIs, Deployment & Ownership

**Q41. You've built roughly a dozen FastAPI services. What is your default service skeleton, and how do you decide between a single polymorphic endpoint and one endpoint per document type?**

Answer: My default skeleton is: an `APIRouter` with a prefix and tag, multipart `UploadFile` intake, validation at the edge, a uniform response envelope, a `/health` route under a `monitoring` tag, and framework-generated OpenAPI/Swagger. In **DOHA OCR AI** that was `APIRouter(prefix='/doha', tags=['Document Processing'])` for the three card endpoints, root-mounted `/doctor_license` and `/police_report`, `/health` reporting version 1.0.0, and a `{message, messageType: 'S'|'E'}` envelope so malformed files, empty payloads, unreadable scans and Azure timeouts never surface as an unhandled 500. In **UAE OCR API — Query-Based Extraction** I versioned it properly: ten routes under `/api/v1`, Pydantic v2 response models, and explicit 400/415/422/500 semantics.

On granularity, I've done both and the trade-off is real. In the **KYC Document Validation System** I used one polymorphic `POST /process` dispatching on a `type` form field across five modes — face similarity, ROI OCR, keyword match, expiry, resolution. It kept the UI trivial, but the cost was weak per-mode typing in OpenAPI and a shared `{"error": ...}` shape returned instead of proper status codes. In **UAE OCR AI** I went the other way: nine typed extraction endpoints, each with its own anchor-verification rule and documented error catalogue, so a Sharjah template change couldn't regress vehicle registration.

My rule now: one endpoint per contract when downstream systems bind to distinct schemas; polymorphic only when the payload shape is genuinely identical and the consumer is a single internal UI. If I rebuilt KYC, I'd split the routes and return real status codes.

Project tie-in: Lead with DOHA OCR AI's envelope and router layout, then use KYC's polymorphic endpoint as the "what I'd change" example.

Follow-ups to expect:
- Why return HTTP 200 with `messageType: 'E'` instead of a 4xx?
- How do you evolve a response schema without breaking integrated clients?
- What does your `/health` endpoint actually check?

---

**Q42. Walk me through your error-handling and request-validation strategy for a service that accepts arbitrary customer uploads.**

Answer: Three layers. First, edge validation before spending anything: in **UAE OCR API — Query-Based Extraction** I enforced file presence, non-zero length and an extension whitelist (`.jpg .jpeg .png .pdf .tiff .bmp`) mapped to 400/415/422, so a junk binary never reaches a paid Azure call. Second, semantic gates: in **UAE OCR AI** each card type has anchor phrases requiring a ≥50% hit rate and police reports must yield more than three non-empty fields, both rejecting with 422 ("Provided image is not a license front"). **DOHA OCR AI** has an Arabic report-header check on normalized spans for the same reason. **Multi-doc All-in-one OCR** adds OpenCV quality gates — Laplacian variance < 80 blurry, brightness < 50 / > 220, std dev < 30, w·h < 500,000 px — returning actionable per-crop feedback so intake can prompt a re-capture.

Third, a typed exception contract. In **UAE OCR AI** a custom `OCRError` maps to 400/404/422/502 with a structured envelope, and a generic handler logs `logger.critical` with a stack trace behind a 500 that leaks nothing.

Two honest weak spots. In the **Claim Duplication Detector API**, any exception fails the whole multi-PDF batch with a 500 carrying raw exception text, and `documents_processed` counts files received rather than parsed — the fix is per-file error isolation with a partial-success shape and a sanitized error body. In the **Document Classification Model** there are no handlers at all, so an undecodable upload raises `PIL.UnidentifiedImageError` as a bare 500; that PoC needs 400/415 handlers and size/page-count caps.

Project tie-in: Pair one strong example (UAE OCR AI's OCRError taxonomy) with one candid gap (Claim Duplication's all-or-nothing batch) and the concrete fix.

Follow-ups to expect:
- 415 vs 422 — where do you draw the line?
- How would you validate MIME type rather than extension?
- What belongs in an error body a customer-facing portal can see?

---

**Q43. Several of your services orchestrate three or four models in one request. How do you design that pipeline so it stays debuggable and cheap?**

Answer: I stage from cheapest to most expensive and make each stage fail fast. **SLM AI OCR** is the clearest example: seven phases — Basic-Auth from HashiCorp Vault, pypdfium2 rendering at 2× and OpenCV decode, a custom YOLOv8 crop that diverts non-documents to `skipped_images/`, Azure Document Intelligence `prebuilt-read` for line text and polygons, a `DOCUMENT_ANCHORS` multi-token gate that rejects a wrong document type *before* any LLM call, then schema-constrained extraction on a self-hosted Qwen via Ollama, then translation. The anchor gate is deliberately placed ahead of the most expensive stage so a mismatched upload never burns inference.

**UAE OCR AI** applies the same logic to cost: `is_scanned_pdf` routes digital PDFs to local PyMuPDF at zero cloud cost, and a local Tesseract pass on the top third of page 1 at 150 DPI is used purely as a *router* — voting between the Sharjah new/old Azure models and filtering Dubai scan pages at >2 of 7 keywords — so cheap local OCR never reaches the output, it just picks the right expensive model.

Debuggability comes from three things: per-crop inline statuses rather than whole-request failure (**Multi-doc All-in-one OCR** returns `Error`/`Not valid` per item), provenance on every field (**Trade License Extractor** records page and extraction method plus `review_flags`), and an append-only `output.json` audit of every transaction.

The gap I'd close: none of these emit per-stage timing or per-model call metrics. I'd add structured span logging per stage so I could answer "which stage dominates latency" with data rather than inference.

Project tie-in: Use SLM AI OCR's seven phases as the spine and UAE OCR AI's Tesseract-as-router trick as the cost insight.

Follow-ups to expect:
- What happens when a mid-pipeline stage times out?
- How do you decide what fails the request versus what degrades it?
- Where would you add a queue?

---

**Q44. How did you handle concurrency and blocking I/O in your FastAPI services, and where did that decision bite you?**

Answer: Three different approaches, chosen per workload. In **DOHA OCR AI** I declared route handlers as synchronous `def` so Starlette offloads blocking Azure HTTPS calls and PyMuPDF extraction to the anyio worker thread pool, with Uvicorn running two recommended production workers. **UAE OCR AI** does the same and additionally wraps the Dubai editable-PDF parser in a `ThreadPoolExecutor(max_workers=6)` for page and party extraction. **Multi-doc All-in-one OCR** and the **Document Detection API (TextMatch Classification)** use a `ThreadPoolExecutor(max_workers=10)` over document crops, and there I hit a real problem: RapidOCR on ONNX Runtime serialized under mutex contention when a shared session was used across threads. The fix was a thread-local singleton (`_ocr_thread_local`) so each worker holds its own engine — memory cost of ten sessions, but no serialization during high-volume ingestion.

Where it bit me: the **Claim Duplication Detector API** does CPU-bound PyMuPDF rendering, OpenCV and PyTorch inference inside an `async def` handler, which blocks the event loop. It scales only because it's stateless and I run four Uvicorn workers, but the correct fix is a threadpool offload or a task queue. Similarly **SLM AI OCR** is async end-to-end but the integration log from 31 March 2026 shows 5.07–19.09 s per endpoint against a stated "within 10 secs" CPU-bound intent — concurrency didn't fix a latency problem that is really LLM decode on CPU.

Honest position: I know which stages are I/O-bound versus CPU-bound, and I've measured latency only end-to-end, not per stage.

Project tie-in: Lead with the thread-local ONNX fix — it's a specific, non-obvious bug. Then own the async-handler mistake in Claim Duplication.

Follow-ups to expect:
- Why not async Azure SDK clients?
- Threads vs processes for CPU-bound OCR — GIL implications?
- How would you get SLM AI OCR consistently under 10 s?

---

**Q45. You built Streamlit apps as well as APIs. How do you handle state and correctness in Streamlit's rerun model?**

Answer: Streamlit re-executes the whole script on every interaction, so anything expensive or side-effecting needs an explicit guard. In the **Multilanguage Speech-to-Text Voice Data Capture** apps, a single recording would otherwise be re-sent to Gemini on every rerun — a duplicated paid call. I gated it with `audio_bytes != st.session_state.last_audio` so processing only happens when the recording actually changed, and used a truthy-only merge so a new recording adds or corrects fields without wiping earlier values, then `st.rerun()` for immediate feedback. The conversational app reads completion as `data.get('completed', False)` so a missing key defaults safely.

In the **LLM Fine-tuning Platform** review dashboard (Streamlit 1.60.0), the risk was annotator data loss: reviewers navigating Prev/Next without pressing Save. `move(delta)` calls `save_current()` *before* adjusting the index, so edits commit on navigation — 664 UAE and 296 Doha records were reviewed with zero data loss. Widget keys were made unique as `f"{image_path}:{field_name}"` to stop Streamlit reusing state across rows, and a session-state login gate stamps `edited_by` / `edited_fields` for audit provenance.

In the **TextMatch Label Trainer**, region selection is a non-dismissible `@st.dialog` rendered before any control, because region binds every read and write to `labels_DB/<region>_labels.json` — a mis-set sidebar dropdown would silently write one jurisdiction's keywords into another's file.

The honest gap: in Speech-to-Text, `app.py` swallows JSON parse failures with a bare `except: pass`, so a malformed model response gives the user nothing. That should surface `st.error`, log the raw response and retry with a repair prompt.

Project tie-in: The dedup gate and the save-before-navigate guard are the two concrete artefacts; lead with whichever the role cares about (cost vs data integrity).

Follow-ups to expect:
- When would you use `st.cache_data` or fragments instead?
- Streamlit as the whole backend — when does that stop working?
- How do you isolate per-user sessions?

---

**Q46. These are insurance systems handling passports, Emirates IDs and medical claims. Walk me through your security and PII posture — including what's missing.**

Answer: I'll be direct: the posture varies by project and several have real gaps I can name precisely.

What I did well. **SLM AI OCR** loads Basic-Auth credentials from HashiCorp Vault via `hvac` KV v2 at runtime, so no plaintext secrets sit in the repo. In the **KYC Document Validation System** I removed a hardcoded Azure Form Recognizer endpoint and key that lived in a superseded `main-old.py`, shipping a CPU-only production build with no external dependency at all. **DOHA OCR AI** processes identity documents statelessly in memory with no persistence layer, and its synthetic training-data pipeline was built specifically so training data stayed inside the regional Azure tenant boundary. The **LLM Fine-tuning Platform** is the strongest case: I quantized the fine-tuned Gemma 4 E2B to GGUF Q4_K_S and served it on-premise via llama.cpp precisely so customer PII never leaves internal infrastructure, aligned with UAE PDPL and Qatar financial data handling standards.

What's missing, honestly. Most of the services document no authentication, rate limiting or upload size caps — including the **Document Detection API**, which appends identity documents to an unrotated local `output.json` with no retention policy. **SLM AI OCR** pairs Vault-managed credentials with CORS `allow_origins/methods/headers=['*']` plus `allow_credentials=True`, which is contradictory. **VL_version_OCR** stores the Google service-account key (`OCR_key.json`) in the project directory. The **LLM Fine-tuning Platform**'s llama.cpp host uses `Bearer none` with hard-coded IPs.

The fix set is consistent: gateway-level auth, size and page-count caps, secrets in a manager not the repo, tightened CORS, and an encrypted audit store with a defined retention period.

Project tie-in: Open with the on-prem GGUF deployment for sovereignty — it's a deliberate compliance decision — then volunteer the CORS and audit-store gaps before the interviewer finds them.

Follow-ups to expect:
- What's the first control you'd add to a customer-facing upload endpoint?
- How do you handle audit logs that contain PII?
- What changes if the service is exposed outside the VPN?

---

**Q47. How do you serve models on-premise versus in the cloud, and what drove those decisions?**

Answer: The driver was almost always cost, latency or data residency rather than accuracy.

**LLM Fine-tuning Platform** is the full arc: a remote Qwen3.5-122B-A10B teacher on vLLM 0.27.1 generated labels, but the production student is Gemma 4 E2B fine-tuned with 4-bit QLoRA, merged, quantized to `gemma-4-E2B-it-Q4_K_S.gguf` (~3 GB, 4.6B params, n_ctx 65,536) and served on-premise via llama.cpp with a multimodal `mmproj`, exposing the same OpenAI-compatible `/v1/chat/completions` contract as the teacher — so client code is interchangeable. That gave zero per-call API cost and full PII residency. I verified the deployment live via `GET /v1/models` rather than assuming it.

**VL_version_OCR** generalizes that into a pluggable engine: six selectable modes across on-premise Qwen3-VL-4B on vLLM, Gemini-3-Flash-Preview and Azure Llama-4-Scout, chosen per request via a required `model_name` enum, so a sovereignty-sensitive document can be pinned to the local host while others use cloud capacity. **SLM AI OCR** splits the pipeline itself: Azure does OCR only, and PII-bearing entity extraction runs on a self-hosted Qwen via Ollama.

The trade-off I'd own: local serving on CPU costs accuracy and latency. SLM AI OCR's observed 5.07–19.09 s per endpoint against a 10-second design intent is the evidence. The stated paths forward are GPU serving, quantized variants, or batching — but the test hardware isn't recorded, so I can't claim a measured speedup, only the mechanism.

Project tie-in: Use the teacher/student distillation-to-GGUF story; it demonstrates the whole lifecycle rather than just a deployment choice.

Follow-ups to expect:
- What accuracy do you lose to Q4_K_S quantization?
- Why llama.cpp rather than vLLM for the student?
- How would you A/B a local model against the cloud one in production?

---

**Q48. Your documentation repeatedly says accuracy and throughput metrics were not recorded. How do you defend that, and what would you instrument first?**

Answer: I won't defend it as a good outcome — I'll defend the honesty. Across most of these projects the documentation explicitly states that ROI, throughput and field-level accuracy are "Not specified in the project"; what exists are configuration constants and a few POC timings. I'd rather cite the numbers I actually have than invent any: DOHA OCR AI's local PyMuPDF parse at ~10 ms with a benchmarked scanned-PDF alternative at 3.4–3.8 s versus 7–11 s raw; SLM AI OCR's 5.07–19.09 s from the 31 March 2026 integration log; the LLM Fine-tuning Platform's teacher run of 553 images in 18 min 41 s (2.03 s/image); the Document Classification Model's 1.38–2.35 images/s CPU embedding throughput against a documented 7–10 s OCR baseline; and Multi-doc All-in-one OCR's 0.125 s per image and 1.074 s per PDF page for YOLO cropping — which I'd label as stored PoC notebook cell outputs, not production benchmarks.

The real cost shows up as decisions nobody can audit. The **Document Classification Model** ships `REJECT_THRESHOLD = 0.0`, so a 0.7935-confidence page is accepted identically to a 0.9458 one and no reject path was ever exercised. The **Claim Duplication Detector API**'s pHash Hamming threshold of 5 and blank-page ratio of 0.99 were never validated against a labelled set, and its only recorded evidence is one `api.log` run of 4 files and 9 pages. **Multi-doc All-in-one OCR**'s OpenCV quality cuts — Laplacian variance 80, brightness 50/220, std dev 30 — are engineering heuristics that will behave differently on a phone photo than on a flatbed scan, and I have no corpus that proves otherwise.

What I'd instrument first, in order: a labelled evaluation set per document type and region; field-level exact-match accuracy (the **Trade License Extractor** already has a `values_match()` harness over a 16-document ground-truth corpus, and the LLM Fine-tuning Platform has a 100-row stratified validation split — neither was run to a reported number); per-stage latency spans, since today I can only say a request took 19 s, not which stage owned it; and production score distributions so a threshold that stops fitting reality becomes visible rather than assumed.

Project tie-in: Quote the DOHA and LLM Fine-tuning Platform numbers verbatim, then pivot to `REJECT_THRESHOLD = 0.0` — volunteering an unexercised code path shows you read your own system critically.

Follow-ups to expect:
- How would you build the labelled set without leaking PII?
- What's your target metric for a document classifier — accuracy or per-class recall?
- How do you monitor for drift once thresholds are live?

---

**Q49. Design question: you have a hub that classifies and routes documents to a dozen downstream extractor microservices. What are the failure modes and how would you harden it?**

Answer: I built exactly that — **Multi-doc All-in-one OCR** is a hub that segments with YOLO, OCRs, classifies by fuzzy keyword match, validates, then forwards each crop's file buffer to a downstream extractor resolved from an `.env` routing table of 14 document-type mappings (13 distinct URLs) over a `ThreadPoolExecutor(max_workers=10)`, with `CONNECT_TIMEOUT=3.0s` and `READ_TIMEOUT=30.0s`.

Failure modes I designed for: an unmapped document type returns `Endpoint URL not configured` rather than a crash; a failed extractor produces an inline per-crop `Error` status so one bad crop doesn't fail the whole request; HTTP 400/500 are reserved strictly for region-config problems. Classification changes hot-reload from `labels_DB/<region>_labels.json`, so tuning is a file write, not a deployment — which the **TextMatch Label Trainer** exposes as a no-code UI with a 75% acceptance gate before persistence.

Failure modes I did *not* handle, and would fix: no retries, no exponential backoff, no circuit breaker — a slow extractor just occupies a thread-pool slot until the 30 s read timeout. All 14 mappings point at a single host (`192.168.86.213:8000`), so that host is a single point of failure. The audit store is a local `output.json` with no locking, rotation or concurrency guarantee. And the hub POSTs raw crop bytes downstream rather than the OCR text it already produced, so the document is OCR'd twice — extractor autonomy bought at the cost of duplicated work.

Hardening plan: async HTTP client with bounded retries and a per-extractor circuit breaker, extractors spread across hosts behind service discovery, the audit moved to a database keyed by `source_id`, and a decision on whether to pass OCR text to spare the second pass.

Project tie-in: The `.env` routing table plus the three-tier error semantics is the strongest part; the double-OCR observation shows you critique your own architecture.

Follow-ups to expect:
- How do you version the routing table safely?
- What's your backpressure story when all 10 threads are busy?
- Would you make the hub async end-to-end?

---

**Q50. Tell me about an architectural decision you got wrong on one of these systems, how it surfaced, and what you changed as a result.**

Answer: The **KYC Document Validation System** is where I got a structural decision wrong, and the consequence is documented in my own write-up.

The system has three front-ends over one set of validation rules: an HTML/JS + Jinja2 canvas UI, a Streamlit UI and a PyQt5 desktop app. I wanted each to be deployable standalone — web server or local desktop, no code changes — so I duplicated the helper logic (`keyword_match`, `find_expiry`, `check_resolution`, `compare_faces`) into each front-end "with consistent logic". Between the 19-03-2026 and 20-03-2026 versions I went further and inlined the helpers into `main.py`, dropping the `utils.py` import entirely. Framework independence, at the price of no single source of truth.

It surfaced as a silent behavioural drift in the expiry check. The notebook (cell 19) and the earlier `utils.py` accepted an optional separator — `[-/]?` — in the date pattern; production requires the separator: `\b\d{2}[-/]\d{2}[-/]\d{4}\b`. So a separator-less OCR reading like `10061987` — which is exactly what RapidOCR produced on a real sample in cell 17 — falls straight through to "No expiry date found". No exception, no log line, no failing test, because there are no tests. Nothing in the architecture makes it possible for one copy of a rule to be right and another wrong *and* for anyone to notice. The same version pass also dropped the `filepath.exists()` check that the earlier `main.py` had on `GET /image/{filename}`, so a missing file now raises an unhandled `FileNotFoundError` instead of a 404 — a second regression introduced by editing copies rather than a module.

What I changed in how I work: business rules go in one importable module with the UIs as thin clients, and anything that exists in more than one place gets a consistency test — the same lesson applies to config, which is why the fix I specify for **VL_version_OCR**'s anchor-key regression is a test asserting the anchor keys are a subset of the output-schema keys per country, not a careful code review. The rule I now apply is: if two artefacts must agree, either merge them or assert the agreement in CI.

Project tie-in: The `10061987` example is what makes this concrete — a real OCR token from the project's own notebook that production silently drops.

Follow-ups to expect:
- Why not just add tests and keep the duplication?
- How would you have caught the regex drift at review time?
- What is the minimum test suite you would add to KYC first?


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


# Part I — Per-Project Analyses

# Project 1: DOHA OCR AI (FastAPI / Azure Document Intelligence / PyMuPDF)

Source: `3-8-DOHA_OCR_Documentation.md` (Document no. AT/doha-ocr/V1, dated 01-Sep-2026). Client: Qatar Insurance Group. Role statements below are inferred from the system's scope unless the document states them; such items are marked "(inferred)".

## 1. Project Overview

- **Project name:** DOHA OCR AI
- **Business domain:** Insurance (Qatar Insurance Group) — motor claims intake (MOI accident reports, vehicle registrations, driving licenses), identity verification (QID residency permits), and medical-practitioner credential extraction (doctor licenses), feeding first-notice-of-loss (FNOL) processing, core insurance platforms, and underwriting portals. Use of doctor-license data for medical claims or provider onboarding is not stated (inferred). Documents in scope are State of Qatar Driving Licenses, Residency Permits (QID), Vehicle Registration cards, Doctor Licenses, and Ministry of Interior (MOI) traffic accident (police) reports.
- **Problem statement:** Claims adjusters and underwriting specialists manually key identity, vehicle, credential, and accident data from Arabic/English official documents into core insurance platforms — slow and error-prone, with Arabic names transcribed into English by hand (inferred from the stated benefit "transcribing Arabic names into English automatically"). Sending every document to cloud OCR incurs per-page fees and consumes Azure Document Intelligence transaction quotas, and model training was constrained by a regional Azure tenant limited to 'template' build mode (no neural training in region; external cloud services constrained).
- **Project objective:** Deliver a headless FastAPI microservice that converts uploaded images/PDFs into structured JSON via a dual-path pipeline "designed for efficiency and regional deployment flexibility" — Azure AI Document Intelligence custom models for cards and licenses, and a zero-cost, in-memory PyMuPDF rule-based parser for MOI accident PDFs — with automated Arabic<->English translation and a standardized success/error envelope.
- **Expected business outcome:** Elimination of manual keying across five document types, faster claims settlement through instant FNOL intake, zero cloud OCR cost for traffic reports, standardized cross-lingual data for regulatory reporting, resilient downstream pipelines via a uniform error envelope, and an extensible router for future document models (e.g., medical invoices, commercial licenses). The document explicitly states that ROI, labor cost reduction, throughput, and field-level accuracy figures are "Not specified in the project".

## 2. Role and Responsibilities

**Core responsibilities (inferred from system scope)**

- Designed the dual-path processing architecture: cloud-hosted Azure DI custom extraction for identity/vehicle/credential cards versus local rule-based PyMuPDF parsing for MOI police reports.
- Built the FastAPI backend (`DOHA_OCR(27_08)/main.py`), the `/doha`-prefixed `APIRouter`, root-mounted `/doctor_license` and `/police_report` endpoints, and the `/health` monitoring endpoint.
- Implemented the Azure Document Intelligence client integration in `utils.py` and the four field-mapping parsers (`parse_driver_info`, `parse_resident_info`, `parse_vehicle_info`, `parse_doctor_license_info`).
- Trained and iterated Azure DI custom extraction models in Azure DI Studio (`24_07_doha_df`, `24_07_doha_rf`, `24_07_doha_vf`, `28_08_doha_doctor_license` in production) and evaluated a police-report model (`04_08_demo_pol`) in `doha.ipynb`.
- Engineered `police_report_extraction.py`: a regex- and span-based parser that reads the native PDF text layer, resolves Arabic RTL bidirectional wrap artifacts, and extracts incident metadata plus vehicle party, driver, injury, and property-damage tables.
- Integrated Azure AI Translation Text for bidirectional Arabic<->English normalization of names, nationalities, vehicle makes, body shapes, colors, license classes, and accident narratives.
- Engineered the synthetic training-data pipeline (`synthetic/` — `synthetic_report_generator.py` v1 through v3 and batch generators) that produced 300+ balanced samples to overcome the template-only build-mode limitation and multi-page failures.

**Supporting tasks (inferred)**

- Jupyter notebook prototyping (`doha.ipynb`, 41 cells) with stored outputs verifying every pipeline milestone.
- Benchmarking scanned-PDF alternatives: Chrome GUI-automation OCR (`chrome_pdf_extract.py`) and a hybrid local/Azure `prebuilt-read` path (`scanned_pdf_text.py`) with 150-DPI downsampling.
- Defining validation gates (Arabic header check, empty-payload rejection) and the uniform `messageType: 'E'` failure envelope.
- Configuration: model IDs in `.env`; `requirements.txt`; README runtime guidance (Python 3.10+, 2 Uvicorn workers).
- Documentation: this system document, `synthetic/SESSION_SUMMARY.md`, and `azure_di_annotation_guide.md` (Azure DI Studio labelling guidance).
- Corpus curation: verified 30 police-report PDFs in `police_documents`.
- No user interface or prompt design is documented: the only UI is FastAPI's auto-generated Swagger UI, and the system contains no LLM components.

**Estimated ownership level: Senior AI Engineer (inferred)**

Rationale: The document describes one end-to-end system spanning six endpoints, two Azure cognitive services, four production custom models plus one evaluated model, a bespoke Arabic-aware PDF parser, and a synthetic-data pipeline rewritten (v1/v2 abandoned, v3 shipped) after encoding failures. The candidate made architecture-level decisions (dual path to avoid cloud OCR cost; synchronous `def` handlers to leverage Starlette's anyio thread pool; template-mode training with synthetic augmentation) and production-readiness choices (health endpoint, error envelope, worker count). No evidence indicates leading other engineers or governing multiple systems, so "Solution Architect" or "Technical Lead" would overstate the documented scope; the combination of model training, data engineering, and backend delivery exceeds a plain "Developer" or "ML Engineer" label.

## 3. End-to-End Workflow

**Processing stages**

1. **Ingestion:** Client POSTs `multipart/form-data` with a single `file` (`UploadFile`) — image or PDF — to one of five document endpoints.
2. **Routing and validation:** The FastAPI routing layer inspects the target endpoint. `/driving_license`, `/residency_permit`, and `/vehicle_registration` live under `APIRouter(prefix='/doha', tags=['Document Processing'])`; `/doctor_license` and `/police_report` are mounted directly on the root app without the prefix; `/health` is mounted under tag 'Monitoring'. Empty payloads on `/police_report` return the error envelope immediately. Card/license images are streamed to the Document Intelligence client in `utils.py`; `/police_report` forwards to `police_report_extraction.py`. The architecture figure (Figure 1) labels the pipeline as "four synchronous stages" drawn left to right: FastAPI Svc (Validation & Routing) -> Routing endpoints (Validation & Routing) -> Azure doc intelligence (Extracts fields) -> Validation (No of docs) -> JSON (Converts into json format).
3. **Cloud path (cards/licenses):** Bytes are dispatched to the Azure DI custom model named by the `.env` variable (`DRIVING_LICENSE_MODEL`, `RESIDENCY_PERMIT_MODEL`, `VEHICLE_REGISTRATION_MODEL`, `DOCTOR_LICENSE_MODEL`); the returned field dict is mapped by the endpoint parser (`utils.py:26-40`, `41-61`, `63-100`, `103-116`).
4. **Local path (police reports):** PyMuPDF (`fitz`) reads raw PDF text spans in memory. A gate requires the Arabic report header in the normalized spans (mojibake in the source document; decodes to "تقرير حادث طرق", "road accident report" — inferred decoding); scanned or irrelevant PDFs fail it and receive the error envelope. `transfer_window()`, `fix_number_wrap()`, and `is_label()` resolve RTL wrap artifacts and anchor regex extraction (`SLASH_DATE`, `HYPHEN_DATE`, `INCIDENT_NUMBER`, `CHASSIS`, `PLATE`). The top-level entry function exercised in the notebook is `police_report_details()`, which returns the `messageType: 'S'` envelope with populated party and incident dictionaries (doha.ipynb cell 22).
5. **Translation:** Arabic strings -> English via Azure AI Translation Text (e.g., `insurance_company_english`, `vehicle_make_and_model_english`, `claim_english`); English passport/nationality -> Arabic (`nationalityAra` from `nationalityEng`, `name_ar` from `nameEng`).
6. **Validation (number of documents)** per the architecture figure, then envelope assembly.
7. **Response:** `{"message": "success", "messageType": "S", "response": {...}}` or `{"message": "Irrelevant document to proceed/Document quality is low, Please try another...!", "messageType": "E"}` — both HTTP 200, never an unhandled 500.

**Data flow:** Multipart upload -> in-memory bytes -> Azure DI field dict (cards) or PyMuPDF span list -> normalized Python dicts -> Azure Translator calls for selected strings -> flat JSON (cards) or nested JSON with `vehicle_party[]`, `injured_party[]`, `property_damage_party[]` (police report). No persistence layer is described; processing is stateless.

**Integrations and dependencies:** Azure AI Document Intelligence (`azure-ai-documentintelligence` 1.0.2), Azure AI Translation Text (`azure-ai-translation-text` 1.0.0b1), `azure-core` 1.32.0, PyMuPDF 1.28.0, `arabic-reshaper` 3.0.0, FastAPI 0.141.1 on Uvicorn 0.52.0, Python 3.12.7 (3.10+ declared); all are unpinned in `requirements.txt`, versions verified in the `plp` environment. The app is served via the Uvicorn ASGI server "with standard multi-threaded worker pooling"; concurrency relies on Starlette offloading synchronous `def` handlers to the anyio thread pool.

**Flow line:**

`Client -> FastAPI (multipart /doha/* | /doctor_license | /police_report) -> [Azure DI custom model -> utils.py parser] OR [PyMuPDF spans -> header gate -> regex/RTL parser] -> Azure Translator (ar<->en) -> Envelope (S/E) -> JSON`

**Architecture Summary:** DOHA OCR AI is a stateless FastAPI microservice with a bifurcated extraction core designed for efficiency and regional deployment flexibility. Stable-layout cards are delegated to purpose-trained Azure DI custom models (template build mode is documented for the regional resource in the context of police-report training; that the four card models are also template-mode is inferred), each addressed by an environment-configured model ID and normalized through a thin per-document parser. Digitally generated MOI accident PDFs, which carry a native text layer, bypass cloud OCR and are parsed locally with PyMuPDF and anchored regular expressions that account for Arabic RTL wrapping. Both paths converge on Azure Translator and a shared success/error envelope, so downstream claims systems consume one contract regardless of document type. Model training was unblocked by a synthetic data pipeline that clones real reports while regenerating only digit-bearing rectangles.

## 4. Technologies and Tools Used

| Category | Technologies (with versions where documented) |
|---|---|
| Programming languages | Python — 3.10+ declared in README.md; 3.12.7 verified in environment |
| Frameworks | FastAPI (unpinned; 0.141.1 verified), Starlette/anyio thread pool (via FastAPI), Uvicorn ASGI server (unpinned; 0.52.0 verified) |
| Libraries | PyMuPDF / `fitz` (unpinned; 1.28.0 verified), `arabic-reshaper` (3.0.0 verified), `azure-core` (1.32.0 verified), `azure-ai-documentintelligence` (1.0.2 verified), `azure-ai-translation-text` (1.0.0b1 verified), Python `re` regular expressions; POC-only: `pyautogui`, `pyperclip`, `pygetwindow` |
| AI/ML models | Azure DI custom extraction models `24_07_doha_df` (driving license), `24_07_doha_rf` (residency permit), `24_07_doha_vf` (vehicle registration), `28_08_doha_doctor_license` (doctor license); `04_08_demo_pol` (police report, evaluated only); Azure DI `prebuilt-read` (POC only); Azure AI Translation Text |
| OCR tools | Azure AI Document Intelligence (custom models, template build mode); PyMuPDF native text-layer extraction (no OCR engine); Chrome built-in PDF "searchify" OCR (POC, abandoned) |
| Databases | Not stated in documentation (no persistence layer described) |
| Cloud platforms | Microsoft Azure (Document Intelligence, Translator Text, regional tenant with template-only build mode) |
| APIs | REST endpoints: `GET /health` (tag 'Monitoring'), `POST /doha/driving_license`, `POST /doha/residency_permit`, `POST /doha/vehicle_registration` (under `APIRouter(prefix='/doha', tags=['Document Processing'])`), `POST /doctor_license`, `POST /police_report` (root-mounted); Azure DI API; Azure Translator Text API; Swagger UI (FastAPI auto-docs — the document has a "Swagger UI Version" heading with no body text) |
| DevOps tools | Not stated in documentation |
| Version control | Not stated in documentation |
| Deployment tools | Uvicorn with 2 recommended production worker processes (README.md:54); `.env` configuration for model IDs; `requirements.txt` |
| Document processing tools | PyMuPDF (text-span reading; its use for the synthetic generator's sub-rectangle redaction and raster twins is inferred — the document does not name the redaction tool), Azure DI Studio (annotation/training), `arabic-reshaper` (used in the abandoned v1/v2 generator's Arabic reshaping), synthetic generators (`synthetic_report_generator.py` v3, `generate_split_variants.py`, `generate_table_variants.py`, `generate_final_set.py`) |
| Automation tools | Synthetic dataset batch scripts (above); Jupyter notebook (`doha.ipynb`, nbformat 4); Windows GUI automation stack (POC only) |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Modular FastAPI application (`main.py`, `utils.py`, `police_report_extraction.py`, `scanned_pdf_text.py`); `APIRouter` with prefix/tags; uniform envelope contract; graceful handling of malformed files, unreadable scans, empty payloads, and Azure timeouts without HTTP 500s; environment-driven configuration.
- **AI Engineering:** Selection and orchestration of Azure DI custom models per document type; hybrid cloud/local strategy driven by cost, quota, and "regional deployment flexibility"; post-processing of model output into a stable schema; evaluation of a cloud model (`04_08_demo_pol`, notebook cells 26-32) while the local PyMuPDF parser is what production uses (the document does not state the comparison outcome; the choice is inferred from what shipped).
- **Machine Learning:** Custom extraction model training in Azure DI Studio under template build mode; diagnosing a generalization failure (3rd vehicle party missing on multi-page reports) as a training-distribution problem (single-page-only samples) and fixing it with 300+ balanced synthetic samples covering 1-7 vehicle layouts, cross-page injury-table splits, and scan-style raster twins.
- **NLP:** Arabic RTL bidirectional wrap resolution, Arabic encoding handling (logical vs visual order vs presentation forms), `arabic-reshaper`, regex anchoring in mixed Arabic/Latin text, bidirectional machine translation orchestration for names, nationalities, and free-form narratives.
- **Computer Vision:** Document-image OCR via Azure DI; 150-DPI JPEG downsampling benchmark (3.4-3.8 s vs 7-11 s raw); scanned-page detection via a 30-character `min_chars` threshold; scan-style raster twin generation for training data.
- **Data Engineering:** Synthetic PDF dataset pipeline (redact-and-regenerate digit rectangles, batch variant generation, 300+ samples); corpus verification (30 PDFs); annotation guide for labelling.
- **Cloud:** Azure DI and Translator SDK integration, regional tenant constraints, transaction-quota awareness.
- **MLOps:** Model IDs externalized to `.env` for swap-without-code-change; versioned model naming (`24_07_*`, `28_08_*`, `04_08_*`); notebook-based verification of model outputs; `/health` liveness endpoint reporting version `1.0.0`. CI/CD, monitoring beyond liveness, and model registry are not demonstrated.
- **API Development:** Six documented endpoints with multipart `UploadFile` inputs, typed response schemas (e.g., `damage_type: "Damage | No Damage"`, `injury_type: "minor | death | major"`), Swagger UI.
- **Prompt Engineering:** Not demonstrated in this project (no LLM/prompt components are described).
- **System Design:** Dual-path routing, synchronous handlers to exploit Starlette's anyio thread pool for blocking I/O, stateless in-memory processing, extensibility via router modularity, cost-aware separation of digital vs scanned inputs.

## 6. Detailed Technical Contributions

**Features implemented**

- `GET /health` (tag 'Monitoring'): returns `{"status": "healthy", "service": "DOHA OCR AI", "version": "1.0.0"}`; no declared error responses.
- `POST /doha/driving_license`: model `24_07_doha_df`; fields `personal_number`, `name`, `nationality`, `date_of_birth`, `blood_group`, `first_issue`, `validity`, `licensing_authority_number`. Documented nuance: `personal_number` and `licensing_authority_number` both read `licensing_authority_number` from the extraction dict.
- `POST /doha/residency_permit`: model `24_07_doha_rf`; 14 attributes including `idNo`, `dob`, `expiry`, `nationalityEng`, `nationalityAra`, `occupation`, `nameEng`, `name_ar`, `passportNumber`, `passportExpiry`, `serialNo`, `residencyType`, `employer`, `address`. Translation: `nationalityAra` <- `nationalityEng` (en->ar); `name_ar` <- `nameEng` (en->ar).
- `POST /doha/vehicle_registration`: model `24_07_doha_vf`; 31 attributes spanning owner (`ownerEng`/`ownerAra`), dates (`regDate`, `renewDate`), technical specs (`cylinder`, `totalWeight`, `seatCapacity`, `chassisNo`, `engineNo`), paired Arabic/English descriptors (`vehicleType`/`vehicleTypeEng`, `shape`/`shapeEng`, `color`/`colorEng`, `country`/`countryEng`), and insurance (`insuranceCompany`, `policyNo`, `insuranceType`, `propertyType`). Documented nuance: duplicate keys `exp_date` (line 76) and `expDate` (line 95) both read `expdate`.
- `POST /doctor_license` (root-mounted): model `28_08_doha_doctor_license`; fields `licenseNo`, `name`, `organization`, `issueDate`, `expiryDate`, `qatarID`, `occupation`.
- `POST /police_report` (root-mounted): nested payload with incident metadata (`report_status`, `report_date`, `incident_date`, `incident_time`, `incident_day`, `incident_number`, `nature_of_loss`, `reason_of_accident`, `accident_location`, `street_name`, `road_condition`, `weather_condition`), booleans `is_unknown_damage`/`is_legal_claim`, `guilty_charge`, `claim_description {claim_arabic[], claim_english[]}`, and arrays `vehicle_party[]` (`insurance_detail`: company/company_english/validity/policy_no/type; `vehicle_detail`: `chasiss_number`, `plate_no`, `owner_id`, `owner_name`, `damage`, `damage_type` "Damage | No Damage", `damage_est` boolean, `vehicle_color`, `vehicle_first_registration`, `vehicle_model`, `vehicle_make_and_model`/`_english`, `is_blamed_party` true | false | null; `driver_detail`: `driver_id`, `driver_name`, `driver_nationality`, `license_type`/`_english`, `license_expiry_date`, `driver_age`, `driver_gender`, `driver_gender_english` "Male | Female | null"), `injured_party[]` (`person_identity`, `person_name`, `nationality`, `age`, `injured_person`/`_english`, `injured_vehicle`, `injury_type` "minor | death | major"), `property_damage_party[]` (`owner_number`, `property_name`, `damage_description`). Documented nuance: `chasiss_number` misspelling at `police_report_extraction.py:334`.

**Models used:** Four production Azure DI custom models plus `04_08_demo_pol` (evaluated in notebook cells 26-32, not in production). Azure `prebuilt-read` only in the abandoned hybrid POC. Azure AI Translation Text for ar<->en.

**Pipelines built:** (1) Cloud card pipeline: upload -> Azure DI -> parser -> translator -> envelope. (2) Local police-report pipeline: upload -> PyMuPDF spans -> header gate -> RTL/regex parsing -> translator -> envelope. (3) Synthetic training pipeline: real report -> sub-rectangle redaction of digit-bearing values (IDs, VINs, dates, plates) -> variants via `generate_split_variants.py`, `generate_table_variants.py`, `generate_final_set.py` -> 300+ samples -> Azure DI Studio.

**APIs integrated:** Azure Document Intelligence (custom model analyze), Azure Translator Text, exposed via FastAPI REST with Swagger UI.

**Data extraction methods:** Azure DI field extraction for cards; PyMuPDF text-layer span reading for PDFs; regex anchors — `SLASH_DATE` `\d{4}/\d{2}/\d{2}`, `HYPHEN_DATE` `\d{4}-\d{2}-\d{2}`, `INCIDENT_NUMBER` `(?<![\d-])\d{4}-\d{1,5}-\d{1,6}(?![\d-])`, `CHASSIS` `[()]([A-Z0-9]{10,})[()]` (parenthesized VIN), `PLATE` `(?<!\d)\d{4,7}(?!\d)`; helpers `transfer_window()`, `fix_number_wrap()`, `is_label()`.

**Validation logic:** Arabic header gate on `/police_report`; empty-payload short-circuit; "Irrelevant document / quality is low" envelope for invalid scans or Azure failures; scanned-page classification at `min_chars` = 30 characters (`scanned_pdf_text.py`); a "Validation (No of docs)" stage in the architecture figure.

**Automation workflows:** Synthetic dataset batch generation; notebook-driven verification (cells 1, 6-13, 22, 23, 26-32, 35-37 with stored outputs, e.g., incident date `2025-02-04`, time `19:01`, incident day and nature-of-loss Arabic strings (mojibake in source; inferred decoding "Tuesday" and "two-vehicle collision"), nationality `YEMEN`, blood group `B+`, plus sample `idNo` and `vehicleNo` values). Cell 1 verified the `28_08_doha_doctor_license` model returning name, license number, organization, and QID; cell 22 executed `police_report_details()` end to end; cells 26-32 tested `04_08_demo_pol` on police-report images, extracting incident numbers and insurance fields. The document states `doha.ipynb` "evolved directly into `utils.py` and `police_report_extraction.py`".

**Optimization techniques:** Bypassing cloud OCR for digital PDFs (~10 ms parse); synchronous handlers offloaded to anyio worker threads; 150-DPI downsampling with a 500,000-byte `MAX_UPLOAD_BYTES` trigger (POC); 2 Uvicorn workers.

**Performance improvements (documented benchmarks):** Hybrid POC achieved 3.4-3.8 s on `sample_3.pdf` at 150 DPI versus 7-11 s raw upload with identical OCR accuracy; Chrome GUI OCR took ~15 s and was abandoned; production local parse of digital PDFs runs in ~10 ms.

## 7. Challenges and Solutions

1. **Regional Azure tenant supports only 'template' build mode (technical/business).** Neural training was unavailable in region and external cloud services were constrained. Solution: train template-mode models and offset their layout sensitivity with a balanced synthetic dataset (300+ samples) generated inside the tenant boundary, documented in `synthetic/SESSION_SUMMARY.md` and `azure_di_annotation_guide.md`. Alternatives (analyst suggestion): train in another region and import the model (data-residency risk), or a fully local layout-aware parser for all document types.
2. **Early models missed the 3rd vehicle party on multi-page reports (technical).** Root cause: training samples contained only single-page examples. Solution: the batch scripts `generate_split_variants.py`, `generate_table_variants.py`, and `generate_final_set.py` produced 300+ balanced samples covering multi-party layouts (1-7 vehicles), cross-page injury-table splits, and scan-style raster twin artifacts (the document attributes these collectively to the three scripts). Alternative (analyst suggestion): page-wise inference with post-hoc party merging.
3. **Synthetic generator v1/v2 failed on mixed Arabic encodings (technical).** Whole-span redact-and-reinsert with `arabic-reshaper` broke because source PDFs mixed logical-order, visual-order, and presentation-form encodings. Solution: v3 replaces only machine-readable digit rectangles, preserving Arabic typography and vector borders pixel-for-pixel. Alternative (analyst suggestion): image-level augmentation only, at the cost of field-level label alignment.
4. **Arabic RTL bidirectional wrap artifacts in PDF text layers (technical).** The document states the parser must resolve "Arabic Right-to-Left (RTL) bidirectional wrap artifacts"; that numbers and labels wrap in visual order and break naive text search is the inferred mechanism (consistent with the helper name `fix_number_wrap()`). Solution: `transfer_window()`, `fix_number_wrap()`, `is_label()` (developed step by step in doha.ipynb cells 6-13) plus lookaround-guarded regexes. Alternative (analyst suggestion): coordinate-based table reconstruction from span bounding boxes.
5. **Per-page cloud OCR cost and quota consumption (business).** Solution: route digital MOI PDFs to the local PyMuPDF parser (~10 ms, zero cloud cost). Documented alternatives: Chrome GUI automation OCR (abandoned — needs desktop focus) and hybrid `prebuilt-read` with 150-DPI downsampling (benchmarked, not shipped).
6. **Scanned or irrelevant PDFs on the local path (technical, documented limitation).** The production parser relies solely on the text layer; scanned reports fail the header gate and return the error envelope rather than falling back to OCR. Current handling: fail fast with the uniform envelope. Alternatives (analyst suggestion): integrate the benchmarked `scanned_pdf_text.py` hybrid (`is_scanned` at `min_chars` = 30, `prebuilt-read` at 150 DPI), or route scans to the evaluated `04_08_demo_pol` Azure DI model.
7. **Blocking I/O inside an ASGI app (technical).** Solution: synchronous `def` handlers so Starlette runs them in the anyio thread pool. Alternative (analyst suggestion): async Azure SDK clients, or a task queue for batch load.
8. **Schema hygiene issues (technical, documented).** `personal_number` aliasing `licensing_authority_number`; duplicate `exp_date`/`expDate` both reading `expdate`; `chasiss_number` misspelling at `police_report_extraction.py:334`. No fix is documented. Alternatives (analyst suggestion): add corrected keys behind a versioned route and deprecate the legacy ones, or emit both keys until integrated consumers migrate — interview material on contract versioning and backward compatibility.

## 8. Impact Analysis

- **Business impact:** Automates structured extraction for five high-volume Qatar documents feeding core insurance platforms, underwriting portals, and customer-facing mobile/web claim-submission channels; enables instant FNOL intake and faster liability attribution and repair approvals. Establishes an auditable synthetic-data asset (300+ samples) for training within regional tenant boundaries where neural build modes or external cloud services are constrained, and an extensible router for new document models (e.g., medical invoices, commercial licenses).
- **Productivity improvements:** Removes manual keying for claims adjusters and underwriting specialists (qualitative; no labor-reduction percentage is given).
- **Accuracy improvements:** Field-level exact-match accuracy across production datasets is explicitly "Not specified in the project". The only accuracy statement is that the 150-DPI POC path produced "identical OCR accuracy" to raw upload.
- **Cost savings:** Zero cloud OCR cost for traffic reports — no Azure DI transaction quota or per-page fees for digital PDFs. Total financial ROI is not specified.
- **Time savings:** Digital MOI PDFs parsed in ~10 milliseconds; POC benchmark 3.4-3.8 s vs 7-11 s for scanned PDFs. Production throughput (documents/minute under concurrency) is not specified.
- **User benefits:** One uniform JSON contract with an error envelope that prevents downstream pipelines from crashing; automatic Arabic->English normalization of names, makes, colors, and narratives; English->Arabic nationality translation for regulatory reporting.

## 9. Interview Discussion Points

- Why a dual path: cost/quota reasoning, the text-layer assumption for MOI PDFs, and why `04_08_demo_pol` was evaluated but the local parser shipped.
- How template-only build mode shaped the ML strategy, and why synthetic data — not more annotation — was the lever.
- The v1/v2 -> v3 generator story: mixed Arabic encodings, why whole-span reinsertion failed, why digit-only redaction preserved label alignment.
- The multi-page 3rd-vehicle-party failure: diagnosing a training-distribution gap and fixing it with 1-7 vehicle variants, cross-page splits, and raster twins.
- Arabic RTL wrap handling: what `transfer_window()`, `fix_number_wrap()`, `is_label()` do, and why lookarounds in `INCIDENT_NUMBER` and `PLATE` matter.
- Concurrency: why synchronous `def` handlers, how Starlette/anyio offloads blocking Azure calls, when you would switch to async clients.
- Error contract: why failures return HTTP 200 with `messageType: 'E'`, and the trade-off for consumers and monitoring.
- Known defects (`exp_date`/`expDate`, `personal_number` aliasing, `chasiss_number`): how to fix them without breaking integrated clients.
- What is missing: scanned-PDF fallback, auth, rate limiting, CI/CD, observability beyond `/health`, measured production accuracy.
- Data residency and PII: in-memory processing of identity documents, training data kept in the regional tenant, synthetic pipeline as a privacy-preserving asset.
- Extensibility: adding a medical-invoice model — `.env` model ID, `utils.py` parser, router entry.
- Benchmarks to cite verbatim: ~10 ms local parse, 3.4-3.8 s vs 7-11 s, ~15 s Chrome POC, 300+ synthetic samples, 30-file corpus.
- "Regional deployment flexibility" as a stated design driver: how the dual path keeps accident-report parsing fully on-premise while card extraction still uses the regional Azure tenant.
- Route-mounting inconsistency: three card endpoints sit under `APIRouter(prefix='/doha', tags=['Document Processing'])` while `/doctor_license` and `/police_report` are root-mounted; (analyst observation) the later model naming (`28_08_doha_doctor_license` vs `24_07_*`) suggests these were later additions — be ready to explain and to propose consolidating under the prefix.
- The architecture figure's "Validation (No of docs)" stage is not explained in the text; be ready to describe what document-count validation you implemented (or clarify that it is a diagram placeholder).
- Notebook-to-production path: `doha.ipynb` "evolved directly into `utils.py` and `police_report_extraction.py`"; how you promoted exploratory cells (transfer_window/fix_number_wrap/is_label developed in cells 6-13) into a tested module.
- Chrome "searchify" POC: why GUI automation (`pyautogui`/`pyperclip`/`pygetwindow`) needing desktop focus and an active window lock is unsuitable for a headless server, and what you learned about scanned-PDF cost/latency from it.

## 10. Architecture Explanation Points

Start with the contract: clients upload one file via multipart and always get a JSON envelope — `messageType` 'S' with a `response` object, or 'E' with a message — never an unhandled 500. The document's own Figure 1 shows the pipeline as "four synchronous stages" left to right (FastAPI Svc validation & routing -> routing endpoints -> Azure DI field extraction -> Validation (No of docs) -> JSON conversion); use it as the top line, then draw two lanes beneath the router (`/doha`-prefixed router tagged 'Document Processing', root-mounted `/doctor_license` and `/police_report`, `/health` tagged 'Monitoring'). Lane one (cards and licenses): `/doha/driving_license`, `/doha/residency_permit`, `/doha/vehicle_registration`, `/doctor_license` stream bytes to Azure DI custom template models selected by `.env` model IDs; a thin `utils.py` parser maps fields to the schema. Lane two (`/police_report`): PyMuPDF reads the text layer in memory, a header gate confirms an MOI accident report, and anchored regexes with RTL-wrap correction build nested party, injury, and property-damage arrays in ~10 ms at no cloud cost. Both lanes call Azure Translator for Arabic<->English fields before returning.

Key decisions: cost-driven separation of digital vs image inputs, framed in the document as "efficiency and regional deployment flexibility"; synchronous handlers so Starlette's thread pool absorbs blocking Azure I/O, served by Uvicorn with standard multi-threaded worker pooling; model IDs externalized to `.env` for swap-without-deploy; 2 Uvicorn workers recommended (README.md:54). Below the runtime, sketch the training loop: the regional tenant only offers template build mode, so a synthetic generator clones real reports, redacts only digit rectangles, and emits 300+ variants (1-7 vehicles, cross-page tables, raster twins) into DI Studio.

Trade-offs: template models are layout-sensitive; the local parser assumes a text layer and rejects scans; HTTP 200 on errors simplifies clients but complicates monitoring. Next: integrate the benchmarked 150-DPI `prebuilt-read` fallback for scans, fix schema duplicates behind a versioned contract, add async clients or a queue for burst load, add auth and structured logging, and measure field-level accuracy on a labelled production sample.


# Project 2: UAE OCR API — Query-Based Extraction (FastAPI / Azure Document Intelligence)

Source document: `4-8-Query_OCR_Documentation.md` (Document no. AT/query-ocr/V1, dated 01-Sep-2026, Client: Qatar Insurance Group). Service self-identifies as "Query-Based OCR & Bilingual Extraction API", version 1.0.0.

## 1. Project Overview

- **Project name:** UAE OCR API (project name `Query_OCR`), technology line "FastAPI / Azure Document Intelligence (Query-Based)".
- **Business domain:** Insurance — customer onboarding / KYC, policy issuance, automated underwriting, and claims registration for client Qatar Insurance Group, covering UAE/GCC identity and commercial documents. Documents handled are UAE Driving Licenses (front/back), Emirates ID / Resident Cards (front/back), Vehicle Registration Cards / Mulkiya (front/back), and Department of Economy & Tourism Commercial Trade Licenses. The document states that the repository contains no user persona documentation, access control rosters, or role declarations; it infers the intended consumers (upstream insurance administration systems, digital customer onboarding portals, automated underwriting pipelines, third-party policy verification microservices) from code artifacts and schema design.
- **Problem statement:** Insurance onboarding and policy administration require transcribing identity and vehicle credentials by hand. Traditional template-based OCR and bounding-box coordinate scraping break under rotation, formatting shifts, resolution degradation, and document redesigns; the document also ties bilingual (Arabic/English) enrichment to "core operational compliance standards", implying conventional OCR does not produce the bilingual values insurers need for compliance and core-system ingestion (inferred).
- **Project objective:** Deliver a stateless, synchronous RESTful microservice that runs zero-shot semantic queries against document layouts using Azure AI Document Intelligence's `QUERY_FIELDS` feature on the `prebuilt-layout` model, enriches key fields with Arabic/English translations via Azure AI Translation, and returns strongly typed Pydantic JSON in a standard enterprise envelope — with no custom model training, annotation datasets, or weight deployment.
- **Expected business outcome:** Elimination of manual data entry ("up to 11 discrete structured fields per document side in a single API call"), zero-shot adaptability to new formats through `/api/v1/extract/query`, deterministic bilingual normalization, contract-safe ingestion into insurance core systems, and support for UAE Central Bank / KYC mandates. No ROI, cost, accuracy, or throughput figures are recorded.

## 2. Role and Responsibilities

**Core responsibilities (inferred from what was built):**

- Designed the end-to-end architecture: FastAPI gateway -> Azure Document Intelligence query extraction -> Azure translation enrichment -> typed bilingual JSON (Figure 1 in the document).
- Ran feasibility research in `workbook.ipynb` (Cells 1-5), proving that `QUERY_FIELDS` on `prebuilt-layout` extracts key fields from real UAE scans without custom models.
- Authored per-document query presets — `df`, `db` (driving license), `rf`, `rb` (resident ID), `vf`, `vb` (vehicle Mulkiya), `tr` (trade license) — carried into `services/ocr_service.py`.
- Built the Azure client layer (`services/azure_client.py`) wrapping `azure-ai-documentintelligence` 1.0.2 and `azure-ai-translation-text` 2.0.0.
- Implemented the FastAPI application (`main.py`, `routers/`): 10 declared routes (2 health/root, 8 extraction), CORS middleware, async lifespan events, `/api/v1` namespace.
- Designed the Pydantic v2 domain models in `models/schemas.py` (`DrivingLicenseFrontData`, `PermittedVehicle`, `TradeLicenseData`, and the `BaseResponse` envelope).
- Implemented `services/translation_service.py` with `@lru_cache(maxsize=1024)`, default `en` -> `ar`, and graceful exception fallbacks.
- Hardened the prototype into production: Pydantic `BaseSettings` configuration, `UploadFile` stream ingestion, extension whitelisting, empty-file detection, centralized exception handling.

**Supporting tasks (inferred):**

- Settings management via `config.py` (translator region `centralus`, host `0.0.0.0`, port `8000`, `.env` loading with environment fallbacks).
- Test tooling setup with `pytest` 9.1.1 and `httpx` 0.28.1 (listed in the stack as automated test runner and HTTP test client; test coverage details are not described).
- Producing the technical documentation itself, including the "Verified Implementation Constants" table with file/line references and the notebook-to-production parameter evolution record.
- Curating sample data for experimentation (e.g. `Dubai - Sample 1.pdf` trade license).

**Estimated ownership level:** AI Engineer (end-to-end owner; inferred).

Rationale: the document shows full-lifecycle ownership — POC notebook, architecture, two Azure cognitive service integrations, API contract design, typed schemas, caching, validation, error handling, and documentation. Design drifts were "intentionally resolved" during the notebook-to-production transition, evidence of engineering judgment beyond a task-executing developer. Scope, however, is a single microservice with no described deployment infrastructure, authentication, observability, or team coordination, so Solution Architect / Technical Lead is not supported. Production-readiness elements (health probes, enterprise envelope, extension whitelist, pytest/httpx test tooling — actual tests are not described) place this at the upper end of AI Engineer, bordering Senior AI Engineer.

## 3. End-to-End Workflow

**Processing stages:**

1. **Client upload.** An upstream system (underwriting platform, onboarding portal, mobile app, verification microservice) sends `multipart/form-data` with field `file` (binary scan: `.jpg`, `.jpeg`, `.png`, `.pdf`, `.tiff`, `.bmp`) to one of the eight `POST /api/v1/extract/...` endpoints. The dynamic endpoint additionally accepts `query_fields` as a comma-separated list or JSON-array string (e.g. `Tax_Invoice_No, Total_Amount, IBAN`).
2. **Gateway validation.** FastAPI (`main.py`, `routers/`) applies CORS headers and, in the document's words, "validates MIME extensions and payload boundaries": file existence, non-zero length, and extension whitelist (`routers/driving_license.py:20`). Extraction routers are bound under the `/api/v1` prefix at `main.py:47-52`. Failures map to `400 Bad Request` (missing/empty file or empty query fields), `415 Unsupported Media Type` (invalid extension), or `422 Unprocessable Entity` (malformed form data).
3. **Query dispatch.** `services/ocr_service.py` selects the preset query list for the endpoint (or the caller-supplied list) and forwards the raw bytes to Azure AI Document Intelligence with `model_id = prebuilt-layout` (`ocr_service.py:116`) and `features = QUERY_FIELDS` (`ocr_service.py:118`).
4. **Cloud layout analysis.** Azure analyzes layout geometry and resolves each natural-language query field against the document, returning string values.
5. **Bilingual enrichment.** For designated fields (holder name, place of issue, nationality, occupation, employer, owner, company name, permitted vehicle classes — grouped by the document as "key demographic, geographic, and organizational values") `services/translation_service.py` calls Azure AI Translation (default source `en`, default target `ar`, both at `translation_service.py:16`; region `centralus`, `config.py:19`), fronted by an in-memory `@lru_cache(maxsize=1024)` with graceful exception fallback. The document's introduction describes the enrichment as "bidirectional Arabic and English translations", but the only documented defaults are `en` -> `ar`; no `ar` -> `en` path is shown.
6. **Typed serialization.** Values are mapped into Pydantic v2 models (`DrivingLicenseFrontData`, `PermittedVehicle`, `TradeLicenseData`, etc.) and wrapped in `BaseResponse` (`message`, `messageType` `'S'`/`'E'`, `response`).
7. **Response / error.** `200 OK` with the envelope; cloud API failures surface as `500 Internal Server Error` through centralized exception handling.

**Data flow:** binary document bytes -> multipart stream (`UploadFile`) -> Azure Document Intelligence request with query strings -> resolved field strings -> translation cache/Azure Translator -> Pydantic model -> JSON envelope. The document describes the pipeline as stateless and synchronous; no persistence layer is mentioned (absence of persistence is inferred from "stateless").

**Integrations and dependencies:** Azure AI Document Intelligence (`prebuilt-layout`, `DocumentAnalysisFeature.QUERY_FIELDS`), Azure AI Translation (multi-service endpoint, default region `centralus`), plus the pinned Python stack listed in Section 4. FastAPI also exposes `GET /docs`, `GET /redoc`, and `GET /openapi.json` (OpenAPI 3.1.0).

**Flow line:**

`Client -> FastAPI (/api/v1, CORS, file validation) -> ocr_service (query preset) -> Azure Document Intelligence (prebuilt-layout + QUERY_FIELDS) -> translation_service (LRU cache -> Azure Translator en->ar) -> Pydantic schema -> BaseResponse JSON`

**Architecture Summary:** A stateless, synchronous ASGI microservice. The document's Figure 1 shows four components (Client Consumer, FastAPI Gateway, Azure Doc Intel, Bilingual Output); regrouped by responsibility (analyst framing) these are: gateway (routing, validation, CORS, exception handling), cognitive-service (Azure client wrappers and per-document query presets), enrichment (cached bilingual translation), and contract (Pydantic v2 schemas inside a fixed envelope). The key design bet is to put field semantics into natural-language queries executed by a foundation layout model rather than trained templates, so document variance is absorbed by the cloud model and new formats are served by the generic `/api/v1/extract/query` endpoint. The gateway is decoupled from the cognitive provider, which the document cites as enabling future substitution with local LLMs or on-premise OCR engines.

## 4. Technologies and Tools Used

| Category | Technologies (with pinned versions where stated) |
|---|---|
| Programming languages | Python (inferred from FastAPI/Pydantic/Jupyter stack; version not stated) |
| Frameworks | FastAPI 0.141.1; Pydantic 2.13.4; Pydantic-Settings 2.14.2 |
| Libraries | azure-ai-documentintelligence 1.0.2; azure-ai-translation-text 2.0.0; azure-core 1.36.1; python-multipart 0.0.32; pytest 9.1.1; httpx 0.28.1; `functools.lru_cache` (standard library) |
| AI/ML models | Azure Document Intelligence `prebuilt-layout` foundation model with `QUERY_FIELDS` feature; Azure AI Translation text service (underlying translation model not named in the document) |
| OCR tools | Azure AI Document Intelligence (query-based layout extraction) |
| Databases | Not stated in documentation (stateless service; no persistence) |
| Cloud platforms | Microsoft Azure (Document Intelligence; Translator multi-service endpoint, default region `centralus`) |
| APIs | REST API built on FastAPI: `GET /`, `GET /health`, 8 `POST /api/v1/extract/*` endpoints; Swagger UI `/docs`, ReDoc `/redoc`, OpenAPI 3.1.0 `/openapi.json`; Azure Document Intelligence API; Azure Translator API |
| DevOps tools | Not stated in documentation (health probe described as intended for container orchestrators and load balancers, but no container/CI tooling is named) |
| Version control | Not stated in documentation |
| Deployment tools | Uvicorn 0.52.0 ASGI server (default bind `0.0.0.0:8000`); no further deployment tooling stated |
| Document processing tools | Azure Document Intelligence; FastAPI `UploadFile` multipart ingestion parsed by python-multipart 0.0.32; supported inputs `.jpg`, `.jpeg`, `.png`, `.pdf`, `.tiff`, `.bmp` |
| Automation tools | Jupyter notebook (`workbook.ipynb`) for POC; pytest for automated tests |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Layered package structure (`main.py`, `routers/`, `services/`, `models/`, `config.py`); Pydantic v2 domain models; `BaseSettings` configuration with `.env` fallbacks; `lru_cache` memoization; centralized exception handling; explicit HTTP status semantics (400/415/422/500); pytest + httpx tooling.
- **AI Engineering:** Zero-shot use of a foundation layout model via natural-language query fields instead of custom extractors; per-document query preset design; notebook POC to production-hardened service; composing two Azure cognitive services into one pipeline.
- **Machine Learning:** Not demonstrated in this project beyond model selection (`prebuilt-layout`); no training, fine-tuning, or evaluation is described.
- **NLP:** Bilingual entity normalization (English -> Arabic) via Azure AI Translation for names, places, nationalities, occupations, employers, company names, and vehicle classes (the document's "key demographic, geographic, and organizational values"); semantic natural-language field querying over document text.
- **Computer Vision:** Document image/PDF layout analysis delegated to Azure Document Intelligence; handling of rotation, formatting shifts, and resolution degradation is attributed to the cloud model, not custom CV code.
- **Data Engineering:** Multipart binary stream ingestion with `UploadFile`; extension whitelisting and empty-file detection; mapping unstructured extraction output into fixed typed schemas per document class.
- **Cloud:** Azure SDK integration (`azure-ai-documentintelligence`, `azure-ai-translation-text`, `azure-core`); region configuration (`centralus`); multi-service Translator endpoint.
- **MLOps:** Not demonstrated in this project (no model versioning, monitoring, or evaluation pipeline is described; the recorded POC-to-production evolution table is the closest artifact).
- **API Development:** RESTful design under a versioned `/api/v1` namespace; consistent `multipart/form-data` contract; standard response envelope (`message`, `messageType`, `response`); health/liveness endpoints; auto-generated OpenAPI 3.1.0 docs; CORS middleware; async lifespan management.
- **Prompt Engineering:** Query-field design — choosing field names such as `Main_Licence_No`, `Company_Name`, `Register_No`, `Phone_No`, `Mobile_No` that the layout model resolved in the recorded notebook run (all 8 trade-license fields extracted in Cell 3; no broader reliability data is recorded); supporting caller-defined query lists for arbitrary documents.
- **System Design:** Stateless synchronous pipeline; provider decoupling to allow future local LLM / on-prem OCR substitution; in-memory caching to cut translation round-trips and cloud cost; type-safe contracts to protect downstream insurance core systems.

## 6. Detailed Technical Contributions

**Features implemented**

- Ten declared routes: `GET /` (service identity: `service`, `version` "1.0.0", `status` "healthy", `docs` "/docs"), `GET /health` (`status` "UP"), and eight extraction endpoints:
  - `POST /api/v1/extract/driving-license/front` -> `license_authority_id`, `license_number`, `license_issue_date`, `license_expire_date`, `date_of_birth`, `issue_place_english/arabic`, `name_english/arabic`, `nationality_english/arabic` (11 fields).
  - `POST /api/v1/extract/driving-license/back` -> `traffic_code`, `transmission_type`, `permitted_vehicle` (`{english: [...], arabic: [...]}` via typed `PermittedVehicle` model).
  - `POST /api/v1/extract/resident-id/front` -> `id_number`, `name_english/arabic`, `date_of_birth`, `nationality_english/arabic`, `issue_date`, `expiry_date`, `sex`.
  - `POST /api/v1/extract/resident-id/back` -> `card_number`, `occupation_english/arabic`, `employer_english/arabic`, `issuing_place_english/arabic`, `family_sponsor`.
  - `POST /api/v1/extract/vehicle/front` -> `traffic_plate_no`, `place_of_issue_english/arabic`, `tc_no`, `owner_english/arabic`, `nationality_english/arabic`, `expiry_date`, `registration_date`, `insurance_expiry_date`, `policy_no`.
  - `POST /api/v1/extract/vehicle/back` -> `model`, `number_of_passengers`, `origin`, `vehicle_type`, `gvw`, `engine_number`, `chassis_number`, `empty_weight`.
  - `POST /api/v1/extract/trade-license` -> `main_licence_no`, `company_name_english/arabic`, `issue_date`, `expiry_date`, `register_no`, `email`, `phone_no`, `mobile_no`.
  - `POST /api/v1/extract/query` -> `extracted_fields` map, `query_count`, `matched_count`.
- Enterprise envelope on all business endpoints: `message`, `messageType` (`'S'` success / `'E'` error), typed `response`.
- Strict upload validation: file presence, non-zero length, whitelist `.jpg .jpeg .png .pdf .tiff .bmp`.
- CORS middleware and async lifespan events mounted in `main.py`; centralized exception handling in the gateway layer (`main.py`, `routers/`); extraction routers bound under `/api/v1` at `main.py:47-52`.
- The trade-license sample response in the API reference reproduces the values recorded in notebook Cell 3 (`main_licence_no` "120278", `register_no` "1014241", the same company name), so the documented contract is traceable to a real extraction run.

**Models used**

- Azure Document Intelligence `prebuilt-layout` with `DocumentAnalysisFeature.QUERY_FIELDS` (no custom-trained model).
- Azure AI Translation text service (default `en` -> `ar`).

**Pipelines built**

- Stateless synchronous request pipeline: validate -> query Azure DI -> translate -> serialize -> envelope.
- POC pipeline in `workbook.ipynb`: direct initialization of the Azure Document Intelligence client (`prebuilt-layout`, `DocumentAnalysisFeature.QUERY_FIELDS`) and Azure Text Translation client plus definition of the seven query lists (Cells 1-2); trade license test on `Dubai - Sample 1.pdf` with 8 query fields `Main_Licence_No`, `Company_Name`, `Issue_Date`, `Expiry_Date`, `Register_No`, `Email`, `Phone_No`, `Mobile_No` (Cell 3, execution count 20, all 8 extracted; recorded output `Company_Name` 'STRONG PLANT FOR DEWATERING & DRAINAGE SERVICES (L.L.C)', `Main_Licence_No` '120278', `Register_No` '1014241'); driving-license front mapping (`DL_front`) with `to_arabic` translation on place of issue and nationality (Cell 4, execution count 15; recorded output `issue_place_english` 'Dubai' and `nationality_english` 'Philippines' with their Arabic translations); driving-license back skeleton (`DL_back`, Cell 5, null execution count — no stored output).

**APIs integrated**

- Azure Document Intelligence SDK (`azure-ai-documentintelligence` 1.0.2) via `services/azure_client.py`.
- Azure Translator SDK (`azure-ai-translation-text` 2.0.0), multi-service endpoint, region `centralus` (`config.py:19`).

**Data extraction methods**

- Zero-shot semantic layout queries: natural-language field names passed as query fields; the model resolves them against layout geometry, avoiding template coordinates.
- Preset query lists per document side (`df`, `db`, `rf`, `rb`, `vf`, `vb`, `tr`) in `services/ocr_service.py`; ad-hoc lists for `/query`.

**Validation logic**

- HTTP-layer: 400 for missing/empty file or empty query list; 415 for non-whitelisted extension; 422 for malformed form data; 500 for cloud failure.
- Schema-layer: "Strict Pydantic response models guarantee uniform JSON contracts across all document classes"; `BaseResponse` wraps every business payload. (The document does not state that fields are required — notebook Cell 4 recorded null license fields — so field-presence enforcement is not confirmed.)

**Automation workflows**

- Single-call extraction of a full document side (up to 11 fields) replacing manual transcription; pytest/httpx automated test harness (details not described).

**Optimization techniques**

- `@lru_cache(maxsize=1024)` on translation calls (`translation_service.py:16`) so high-frequency tokens (emirate names, issuing authorities, common nationalities) are translated once per process.
- "Graceful exception fallbacks" around translation (the document's wording); that this prevents a translator outage from failing the whole extraction is inferred — the fallback behaviour is not specified.
- Stream-based ingestion (`UploadFile`) rather than disk paths.

**Performance improvements**

- Qualitative only: fewer translator round-trips and lower cloud consumption from caching; the document records no latency, throughput, or accuracy measurements.

## 7. Challenges and Solutions

1. **Template brittleness of conventional OCR (technical).** Template/bounding-box extraction fails on rotation, formatting shifts, resolution loss, and redesigns. *Solution:* zero-shot `QUERY_FIELDS` on `prebuilt-layout`, so semantics live in query strings rather than coordinates. *Alternatives:* traditional template-based OCR and naive bounding-box coordinate scraping (explicitly contrasted in the document); custom-trained extraction models per document type (the document states the design avoids custom model training, annotation datasets, and weight redeployment); multimodal LLM vision extraction (analyst suggestion — the document only mentions future local LLM integration).

2. **Bilingual normalization for compliance (business).** Insurance core systems and KYC checks need consistent Arabic and English values. *Solution:* Azure AI Translation on selected fields, producing paired `*_english` / `*_arabic` attributes. *Alternatives:* curated lookup tables for bounded vocabularies such as emirates and nationalities, with MT only for open-ended names (analyst suggestion); reading the Arabic text already printed on the card via a second query set (analyst suggestion).

3. **Translation latency, cost, and connection drops (technical).** The notebook's raw translation calls "failed under high latency or connection drops". *Solution:* `@lru_cache(maxsize=1024)` plus graceful exception fallbacks. *Alternatives:* shared cache (Redis) across workers, retry with exponential backoff, batching all fields of a document into one translation call (analyst suggestions).

4. **Notebook-to-production drift (technical).** The POC loaded `.env` from an absolute path (`C:\...\5-3-2026-azure+llm\.env`), read files via `open(path, 'rb')`, and returned unstructured dicts. *Solution:* Pydantic `BaseSettings` with environment fallbacks, `UploadFile` streams with whitelist/empty-file checks, and rigid Pydantic v2 schemas inside `BaseResponse`. *Alternatives:* wrapping the notebook code in a thin CLI, or a plain `.env` loader with untyped dicts (analyst suggestions; the document records only the chosen refactor).

5. **New document types without retraining (business).** Emirate variations and new formats arrive faster than model retraining cycles. *Solution:* `/api/v1/extract/query` accepts arbitrary query lists and reports `query_count` / `matched_count`. *Alternatives:* per-type custom models (rejected implicitly by the zero-shot design).

6. **Untrusted uploads (technical).** Callers submit arbitrary binary payloads over multipart HTTP; missing, empty, or non-document files must be rejected before a cloud call is spent. *Solution:* existence, non-zero length, and extension whitelisting with explicit 400/415/422 semantics. *Alternative:* content-sniffing MIME validation and size caps (analyst suggestion).

**Documented limitations and bugs (honest talking points):**

- Notebook Cell 4 returned null license fields "because trade license data was piped" — a POC test-wiring bug. The document says the cell was "carried into production" as the driving-license/front endpoint but does not explicitly state the wiring was corrected (inferred from the per-endpoint routing design).
- Notebook Cell 5 (driving-license back) was never executed and used static permitted-vehicle classes ("Light Vehicle"); production adds a typed `PermittedVehicle` model, but the document does not confirm dynamic extraction of classes.
- No quantified accuracy, throughput, or ROI evidence exists.
- `lru_cache` is process-local, so cache benefit does not span multiple Uvicorn workers (analyst observation).
- No authentication, rate limiting, or persistence is described; dates are returned as raw strings without normalization.
- The endpoint summary table lists 8 rows while the text describes 10 routes — a minor documentation inconsistency (the trade-license and query endpoints are detailed in the text but absent from the table). The "Swagger UI Version" subsection is empty.
- The document describes the gateway as performing "MIME type filtering" / validating "MIME extensions", but the only concrete check documented is the extension whitelist (415 is raised for "invalid extension"); content-based MIME sniffing is not evidenced (inferred).
- The introduction claims "bidirectional" Arabic/English translation, yet the only documented defaults are `en` -> `ar` (`translation_service.py:16`); no Arabic-source path is shown.
- The document calls the service "high-accuracy" without recording any accuracy figure, and the repository has no user persona documentation, access control rosters, or role declarations (both stated in the document).

## 8. Impact Analysis

- **Business impact:** Typed, bilingual identity/vehicle/commercial data feeds onboarding, underwriting, claims, and third-party verification; supports UAE Central Bank and KYC mandates. The document claims cycle times move "from hours or days to sub-second automated verification" — a stated strategic benefit, not a measured result. A third stated strategic benefit is "Architectural Agility & Modularity": the REST gateway is decoupled from the cognitive cloud providers, which the document says enables straightforward future integration with local LLMs or on-premise OCR engines.
- **Productivity improvements:** Manual transcription replaced by a single API call returning "up to 11 discrete structured fields per document side"; new document formats served immediately via `/query` without retraining, annotation, or redeployment.
- **Accuracy improvements:** Not quantified. The document states no OCR character accuracy rates are recorded; qualitatively, deterministic translation removes spelling variation in identity values, and typed schemas prevent downstream ingestion errors.
- **Cost savings:** Not quantified. Qualitatively, the 1024-entry LRU cache reduces redundant translator calls and cloud consumption, and zero-shot extraction avoids model training/annotation costs.
- **Time savings:** Not quantified beyond the unmeasured "hours or days to sub-second" claim.
- **User benefits:** Uniform JSON contract across seven fixed document-side endpoints (four document types) plus a generic endpoint; explicit error codes; self-documenting Swagger/ReDoc; `GET /health` liveness probe for container orchestrators and load balancers; `GET /` service identity with a `/docs` pointer.

The document's "Quantified Evidence Disclaimer" is explicit: no ROI percentages, human-hour cost reduction figures, OCR character accuracy rates, or throughput statistics are recorded.

## 9. Interview Discussion Points

- Why zero-shot query fields over a custom-trained Document Intelligence model: no annotation datasets, resilience to redesigns, and instant coverage of new formats via `/query`; trade-off is dependence on cloud model quality with no in-house accuracy baseline.
- How the query presets (`df`, `db`, `rf`, `rb`, `vf`, `vb`, `tr`) were derived and validated in `workbook.ipynb`, including the Cell 4 wiring bug and what changed when moving to routers.
- Justify `@lru_cache(maxsize=1024)`: which tokens repeat (emirates, authorities, nationalities), why 1024, and the limitation that the cache is per-process and lost on restart.
- Machine-translating proper nouns (person and company names) — expected fidelity issues, and how a lookup table or reading printed Arabic could improve it.
- Error taxonomy: 400 vs 415 vs 422 vs 500, and why `messageType 'S'/'E'` exists alongside HTTP status codes for downstream insurance systems.
- The notebook-to-production refactor: config decoupling to `BaseSettings`, `UploadFile` streaming, typed `BaseResponse`, and graceful translator fallback.
- Synchronous vs asynchronous design: the pipeline is synchronous and stateless; how you would handle long Azure analysis times or bursty upload volumes (queueing, async job endpoints).
- Security gaps: extension-only validation, no auth, no size caps — what you would add before exposing this to third-party verification services.
- How `query_count` / `matched_count` on the dynamic endpoint lets callers detect partial extraction, and why the fixed endpoints do not expose per-field confidence.
- Testing approach with pytest and httpx, and how you would build an accuracy benchmark given the absence of recorded metrics.
- PII handling (Emirates ID numbers, sponsor details) in a stateless service that still transits data through two cloud services.
- Why the generic `prebuilt-layout` model plus `QUERY_FIELDS` rather than a document-specific prebuilt identity model: the document records no such comparison, so present it as your own reasoning (analyst suggestion) — one code path covers all seven document sides and arbitrary documents, at the cost of no built-in field confidence or document-type validation.
- Traceability from experiment to contract: the trade-license sample response in the API reference reproduces the values from notebook Cell 3 (execution count 20), so you can walk an interviewer from the recorded POC run to the `TradeLicenseData` schema.
- The "bidirectional" translation claim versus the documented `en` -> `ar` defaults: how you would support Arabic-source documents or verify that the translated Arabic matches the Arabic printed on the card.

## 10. Architecture Explanation Points

Whiteboard narrative (about two minutes):

1. **Components.** Draw four boxes left to right: Client Consumer (underwriting platform / onboarding portal / mobile app), FastAPI Gateway (`main.py`, `routers/`, `/api/v1`, CORS, validation, exception handling), Azure Document Intelligence (`prebuilt-layout` + `QUERY_FIELDS`), and Bilingual Output layer (`ocr_service.py`, `translation_service.py`, `schemas.py`). Note Azure Translator hanging off the output layer with an LRU cache (1024) in front.
2. **Data flow.** A multipart `file` arrives; the gateway checks presence, non-zero size, and extension whitelist; bytes plus a preset query list go to Azure; resolved strings come back; selected fields are translated `en` -> `ar` (cache hit or translator call); everything is bound into a Pydantic model and wrapped in `{message, messageType, response}`. Stateless, synchronous, no database.
3. **Key design decisions.** Semantics in query strings, not templates, so document variance is absorbed by the foundation model; a generic `/query` endpoint for new formats; typed contracts to protect insurance core systems; caching to cut translator cost and latency; provider boundary in `azure_client.py` for future substitution.
4. **Trade-offs.** Cloud dependency and PII transit through two external services; no measured accuracy; process-local cache; synchronous request holding while Azure analyzes; extension-based rather than content-based validation; machine-translated proper nouns.
5. **What to improve next.** Per-document-type accuracy benchmark; shared cache (Redis) with retry/backoff; authentication and size limits; content-based MIME validation to back the document's "MIME type filtering" wording; date normalization and per-field confidence; an Arabic-source (`ar` -> `en`) path to make the "bidirectional" claim true; async job mode for batch ingestion; observability to produce the quantified evidence the document says is missing. (All analyst suggestions — the document names only the local-LLM / on-premise OCR substitution as future work.)


# Project 3: SLM AI OCR (FastAPI / YOLOv8 / Azure Document Intelligence / Qwen LLM)

Source: `5056_AI_OCR_Documentation.md` (Document no. AT/slm-ocr/V1, dated 01-Sep-2026). Client: Qatar Insurance Group.

## 1. Project Overview

- **Project name:** SLM AI OCR (`SLM_AI_OCR`) — an automated document-processing microservice for UAE identity and vehicle documents.
- **Business domain:** Insurance (Qatar Insurance Group). Consuming systems named in the document: automated insurance underwriting platforms, retail motor claims systems, digital customer onboarding workflows, and fleet management portals ingesting UAE identity and vehicle cards. The architecture table names the calling client as an "underwriting portal, claims management system, or mobile app backend". The `SLM_AI_OCR_Project_Specification.docx` classifies the project as a "Management Overview for UAE Document Ingestion"; individual end-user roles, operator personas and admin hierarchies are "Not specified in the project."
- **Problem statement (inferred from the document's Business Benefits section):** Underwriting, claims and onboarding processes depend on data printed on UAE Driving Licenses, Emirates ID / Residence Cards and Vehicle Mulkiya (registration) cards issued across different Emirates and regulatory authorities. Manual rekeying of these credentials is error-prone, documents arrive as loose images or multi-page PDFs containing several cards, wrong document types get submitted, and UAE regulatory and localized reporting requires bilingual (English/Arabic) values. There was no centralized, standardized service abstraction across these document types.
- **Project objective:** Build a stateless, asynchronous FastAPI microservice that ingests a single image or multi-page PDF, localizes and crops each card with a trained YOLOv8 model, runs Azure AI Document Intelligence (`prebuilt-read`) OCR, validates document type against configured anchor tokens, reconstructs reading order, extracts fields with a locally hosted Qwen model under strict JSON-schema constraints (via Ollama's OpenAI-compatible endpoint), and returns normalized snake_case JSON enriched with automated Arabic translations.
- **Expected business outcome:** Elimination of clerical transcription errors, automatic multi-page separation without operator cropping, bilingual enrichment without human translators, fail-fast rejection of misclassified documents before they reach enterprise databases, lower cloud token cost and better PII sovereignty from the hybrid cloud-OCR / self-hosted-LLM design, and centralized credential management via HashiCorp Vault. The document also frames the design as an "Open-source approach": a small language model that "adapts to CPU bound" hardware so that "the extraction may happen with good accuracy within 10 secs timeframe" (stated as intent, not a measurement). ROI, cost-saving percentages, accuracy rates and throughput are explicitly "Not specified in the project."

## 2. Role and Responsibilities

**Core responsibilities (inferred from the delivered system):**

- Designed the seven-phase stateless pipeline (Section 3) and its per-request memory-purging execution model.
- Built the FastAPI service (`main.py`): CORS, HTTP Basic Authentication against Vault secrets, `log_requests` latency middleware, and a `create_route` factory registering six POST routes under the `/uae` APIRouter plus `GET /api/test`.
- Implemented the CV splitter (`utils/yolo_splitter.py`): pypdfium2 rasterization at `scale=2`, OpenCV decoding, YOLOv8 inference with `yolo-multipage-OCR-cls.pt`, bounding-box cropping, and diversion of non-documents to `skipped_images/`.
- Integrated Azure AI Document Intelligence `prebuilt-read` (`logic/ocrllm.py`) for line text plus polygon coordinates, with `HttpResponseError` handling.
- Designed the `DOCUMENT_ANCHORS` multi-token lists and anchor-scoring validation for all six document types, and wrote the `combine_text_lines` clustering algorithm (10-pixel vertical tolerance, horizontal word sort).
- Engineered LLM extraction: pipe-delimited field schemas (`OUTPUT_DICT`, `logic/ocrllm.py:18-25`), `/no_think` prompt control, `temperature=0`, strict `json_schema` response format, via `AsyncOpenAI` against Ollama.
- Built the output mapper (`utils/mapping_output.py`): snake_case normalization and English-to-Arabic derivation with `GoogleTranslator` under a 3-second thread timeout and `max_retries=0`.

**Supporting tasks (inferred):**

- Secrets integration with HashiCorp Vault via `hvac` KV v2 (`utils/vault_credentials.py`, `retail-dev` path).
- Observability: daily-rotating `OCR_API.log` with 24-day retention, plus `gc.collect()` after each request.
- Standardized error payload contract (`{"message": ..., "messageType": "E"}`); dependency pinning in `requirements.txt` and a `.venv`.
- Integration test runs recorded in `log_databases/ocr_api_logs/OCR_API.log` (March 31, 2026) capturing per-endpoint latency.
- Possible contribution to this technical documentation and the `SLM_AI_OCR_Project_Specification.docx` management overview (inferred; authorship is not stated anywhere in the document).
- YOLOv8 model training is implied by the "trained PyTorch YOLOv8 model" wording; data and procedure are not documented.

**Estimated ownership level: Senior AI Engineer (inferred).**
Rationale: the document describes an architecturally broad system: a custom-trained CV model, a cloud OCR integration, a self-hosted LLM with constrained decoding, a bespoke spatial-clustering algorithm, a timeout-isolated translation step, Vault-backed authentication, and production concerns (rotating audit logs, latency profiling, memory hygiene, standardized error contracts, six endpoints). The hybrid cloud/edge decision (Azure OCR plus on-prem Qwen for PII sovereignty and token cost) is an architectural trade-off, not just implementation. No team leadership, CI/CD, containerization or automated tests are evidenced, so "Solution Architect" or "Technical Lead" would overstate what the document supports.

## 3. End-to-End Workflow

**Processing stages:**

1. **Request ingestion & authentication** — Client POSTs `multipart/form-data` with HTTP Basic Authentication to one of the six `/uae/*` routes. Credentials are checked against Vault secrets; exactly one file is required (`len(file) == 1`, `main.py:53`).
2. **Rendering & decoding** — Images (JPEG, PNG, etc.) are decoded from the byte buffer with OpenCV; PDFs are rendered page-by-page with pypdfium2 at `scale=2`.
3. **YOLOv8 localization & cropping** — `yolo-multipage-OCR-cls.pt` (loaded from `./utils/yolo-multipage-OCR-cls.pt`, `utils/yolo_splitter.py:12`) detects document boundaries, crops distinct cards, isolates document classes from extraneous content, and diverts non-document images to `skipped_images/` (directory constant defined at `logic/ocrllm.py:161`). A decode/parse failure in `yolo_split` returns the "Unsupported file type" error; an empty flattened crop list returns "No cropped images found."
4. **Azure OCR** — Each crop is sent to Azure AI Document Intelligence `prebuilt-read` for line text and polygon coordinates.
5. **Anchor validation & reading-order reconstruction** — OCR tokens are scored against `DOCUMENT_ANCHORS` for the targeted type; mismatches are rejected before any LLM call. Valid documents pass through `combine_text_lines`.
6. **Schema-constrained LLM extraction** — Ordered text plus the document type's pipe-delimited field list go to the local Qwen model (`model="qwen"`, `temperature=0`, `/no_think`, strict `json_schema`) via `AsyncOpenAI` at `http://172.20.132.97:11434/v1`.
7. **Normalization, translation & delivery** — `utils/mapping_output.py` maps raw keys to snake_case and derives Arabic (`target='ar'`) values under a 3-second timeout; JSON is returned, `log_requests` records latency (described as "millisecond-accurate timing") and request metadata, then runs `gc.collect()`. If the results list contains no valid documents, the route raises HTTP 400 `{"detail": "No valid documents found"}`.

**Data flow:** multipart binary file -> NumPy image arrays (OpenCV decode / pypdfium2 render) -> cropped card arrays -> Azure line tokens + polygons -> ordered text string -> schema-constrained LLM JSON -> normalized bilingual JSON dict -> HTTP JSON response; side channels: `skipped_images/` on disk and `OCR_API.log`.

**Integrations and dependencies:** Azure AI Document Intelligence (`prebuilt-read`); Ollama serving Qwen behind an OpenAI-compatible API consumed with `AsyncOpenAI`; Ultralytics YOLOv8 with local PyTorch weights; OpenCV and pypdfium2 for decoding and rendering; deep-translator (GoogleTranslator); hvac against HashiCorp Vault; Requests, python-multipart, Pydantic and Uvicorn. Pinned versions are in Section 4.

**Flow line:**
`Client -> FastAPI /uae/<doc_type> (Basic Auth via Vault) -> yolo_split (pypdfium2 / OpenCV -> YOLOv8 crop) -> Azure DI prebuilt-read -> DOCUMENT_ANCHORS check -> combine_text_lines -> Ollama Qwen (AsyncOpenAI, json_schema, temp=0) -> mapping_output (snake_case + GoogleTranslator ar) -> JSON`

**Architecture Summary:** A stateless asynchronous linear pipeline behind a single FastAPI process (Uvicorn on port 5055). Perception is split between an on-box CV model that decides *where* documents are and a cloud OCR model that decides *what text* is present; understanding is delegated to a self-hosted small language model forced into deterministic, schema-valid JSON; enrichment is a bounded-latency translation step. Cross-cutting concerns (Vault auth, CORS, rotating latency logs, garbage collection) live in middleware. Document-type specificity is data (anchor lists and pipe-delimited schemas keyed by endpoint), so six endpoints come from one `create_route` factory.

## 4. Technologies and Tools Used

| Category | Technologies (with pinned versions where documented) |
|---|---|
| Programming languages | Python |
| Frameworks | FastAPI ==0.114.0; Uvicorn ==0.30.6 (ASGI server); Ultralytics YOLOv8 ==8.4.14 |
| Libraries | OpenCV (opencv-python) ==4.11.0.86; NumPy ==2.2.6 (declared; 2.5.2 installed in .venv); PyPDFium2 ==4.30.0; OpenAI Python SDK (AsyncOpenAI) ==2.26.0; deep-translator ==1.11.4 (GoogleTranslator); hvac ==2.4.0; Requests ==2.32.3; python-multipart ==0.0.22; Pydantic ==2.10.6; PyTorch (YOLO weights); asyncio; logging (`TimedRotatingFileHandler`); gc |
| AI/ML models | YOLOv8 custom weights `yolo-multipage-OCR-cls.pt`; Azure Document Intelligence `prebuilt-read`; Qwen (Ollama model id `qwen`, version not specified) |
| OCR tools | Azure AI Document Intelligence ==1.0.2 (`prebuilt-read`) |
| Databases | Not stated in documentation (service is stateless; only file-based logs under `log_databases/`) |
| Cloud platforms | Microsoft Azure (AI Document Intelligence); self-hosted Ollama endpoint `http://172.20.132.97:11434/v1` |
| APIs | REST: `GET /api/test`, six `POST /uae/*` routes; FastAPI auto-docs `/docs` (Swagger UI), `/redoc` (ReDoc), `/openapi.json` (OpenAPI 3.0 specification); Azure Document Intelligence API; Ollama OpenAI-compatible API; Google Translate via deep-translator; HashiCorp Vault KV v2 API |
| DevOps tools | HashiCorp Vault (secrets); Python `.venv`; `.env` file (`port = 5055` at `.env:3`); `requirements.txt`. No CI/CD or containerization stated in documentation |
| Version control | Not stated in documentation (document mentions avoiding credentials "in source control" but names no VCS) |
| Deployment tools | Uvicorn (`uvicorn main:app --host 0.0.0.0 --port 5055`, the verified command); `python main.py` only after setting the `PORT` env var (`set PORT=5055` or `$env:PORT='5055'`) |
| Document processing tools | pypdfium2 (PDF rasterization, scale=2); OpenCV (image decoding); YOLOv8 (card cropping); Azure Document Intelligence (OCR) |
| Automation tools | Custom FastAPI middleware (`log_requests`), `TimedRotatingFileHandler` daily rotation; no external workflow/automation tool stated in documentation |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Modular layout (`main.py`, `utils/`, `logic/`, `log_databases/`); `create_route` factory generating six endpoints from data; standardized error contract; pinned dependencies; middleware for cross-cutting concerns; `gc.collect()` memory hygiene.
- **AI Engineering:** CV + cloud OCR + local LLM + MT composed into one deterministic pipeline; constrained decoding via `json_schema` at `temperature=0`; `/no_think` control; fail-fast anchor gate ahead of the LLM.
- **Machine Learning:** Deployment of a trained YOLOv8 detector (`yolo-multipage-OCR-cls.pt`) with Ultralytics 8.4.14. Training methodology not documented.
- **NLP:** Structured entity extraction from noisy OCR text with an SLM; pipe-delimited schema-to-field mapping; snake_case normalization; English-to-Arabic translation of name, place, nationality, occupation and employer fields.
- **Computer Vision:** PDF rasterization at 2x; OpenCV buffer decoding; YOLOv8 detection and cropping; non-document filtering to `skipped_images/`; polygon-driven line clustering (10-px vertical tolerance, horizontal sort).
- **Data Engineering:** Multi-format ingestion (JPEG/PNG/PDF); flattening multi-page, multi-card crops into a results list; JSON normalization; rotating logs with 24-day retention.
- **Cloud:** Azure AI Document Intelligence SDK integration with `HttpResponseError` handling; hybrid cloud/on-prem rationale (token cost, PII sovereignty).
- **MLOps:** Local weight management (`./utils/yolo-multipage-OCR-cls.pt`); self-hosted LLM serving via Ollama; per-endpoint latency profiling in `OCR_API.log`; version pinning. No model registry, CI/CD or monitoring stack documented.
- **API Development:** FastAPI + Uvicorn; `APIRouter` with `/uae` prefix; multipart uploads; HTTP Basic Auth; CORS; auto-generated OpenAPI/Swagger docs; health endpoint.
- **Prompt Engineering:** `/no_think` system prompt; extraction prompts built from per-type pipe-delimited field lists; strict JSON schema response_format; zero-temperature determinism.
- **System Design:** Stateless linear pipeline separating localization (edge CV), recognition (cloud OCR) and understanding (local SLM); data-driven type configuration (`DOCUMENT_ANCHORS`, `OUTPUT_DICT`); bounded external calls (3-second translation timeout, `max_retries=0`); secrets delegated to Vault.

## 6. Detailed Technical Contributions

**Features implemented**
- `GET /api/test` health check returning the string `'cloud vision data'`.
- Six POST extraction endpoints under `/uae`: `license_front` (identity and license attributes), `license_back` (traffic code, transmission type, permitted vehicle categories), `residence_front` (Emirates ID Number, demographics, validity dates), `residence_back` (Card Number, occupation, employer, issuing place, demographics), `vehicle_front` (ownership, traffic code, plate number, registration dates, insurance policy details), `vehicle_back` (technical specs, model year, origin, chassis number, engine number, gross weight).
- Cross-cutting: Vault-backed Basic Auth; `CORSMiddleware` configured with `allow_origins=['*']`, `allow_credentials=True`, `allow_methods=['*']`, `allow_headers=['*']`; request logging and per-request garbage collection.
- Framework-provided routes: `GET /docs` (Swagger UI), `GET /redoc` (ReDoc), `GET /openapi.json` (OpenAPI 3.0 specification).

**Models used**
- YOLOv8 (Ultralytics 8.4.14) with custom PyTorch weights `./utils/yolo-multipage-OCR-cls.pt` (`utils/yolo_splitter.py:12`).
- Azure Document Intelligence `prebuilt-read` (`logic/ocrllm.py:107`).
- Qwen served by Ollama, model id `qwen`, `temperature=0` (`logic/ocrllm.py:137`), endpoint `http://172.20.132.97:11434/v1`, consumed via `AsyncOpenAI`.
- GoogleTranslator (deep-translator 1.11.4), target `'ar'` (`utils/mapping_output.py:103`).

**Pipelines built**
- Seven-phase asynchronous pipeline described in Section 3; document-type-specific behaviour is driven by `DOCUMENT_ANCHORS` and `OUTPUT_DICT` (`logic/ocrllm.py:18-25`) rather than per-type code.

**APIs integrated**
- Azure AI Document Intelligence (SDK 1.0.2) with `HttpResponseError` capture.
- Ollama OpenAI-compatible chat completions with `response_format` `json_schema` (strict = True, `logic/ocrllm.py:139`).
- Google Translate via deep-translator over a Requests session with `HTTPAdapter(max_retries=0)` (`utils/mapping_output.py:109`).
- HashiCorp Vault KV v2 via hvac (`VAULT_BASE=retail`, `APP_OCR_ENV=retail-dev`, fallback `APP_VAULT_URL=https://hctest.anoudapps.com`; `utils/vault_credentials.py:5-9`).

**Data extraction methods**
- OCR line tokens + polygons -> `combine_text_lines` agglomerative vertical clustering at a 10-pixel threshold (`logic/ocrllm.py:41`), then horizontal word sorting -> ordered text.
- LLM extraction against pipe-delimited field definitions, zero temperature, `/no_think` system prompt (`logic/ocrllm.py:89, 99`), strict JSON schema enforcement.
- Post-extraction mapping of raw keys into snake_case schema fields (`utils/mapping_output.py:6-100`).

**Validation logic**
- File count: `len(file) == 1` (`main.py:53`) else `{"message": "Please attach at least one file/document to proceed...!", "messageType": "E"}`.
- Decode/parse failure: `{"message": "Unsupported file type:  <filename>", "messageType": "E"}`.
- Empty crop list: `{"message": "No cropped images found.", "messageType": "E"}`.
- Azure dimension error: `{"message": "The input image dimensions are out of range. ...: <error_msg>", "messageType": "E"}`.
- Anchor mismatch: `{"message": "Provide image is not <detected_doc>, Please provide valid image of type <doc_type>", "messageType": "E"}` — evaluated before LLM inference.
- No valid results: HTTP 400 `{"detail": "No valid documents found"}`.

**Automation workflows**
- `log_requests` middleware: latency in seconds ("millisecond-accurate timing"), client host, timestamp, method, path, status -> `OCR_API.log`; then `gc.collect()` to clear transient image matrices (`main.py:66-76`, `log_databases/logging_config.py:24-70`).
- `TimedRotatingFileHandler` with `when='midnight'`, `interval=1`, `backupCount=24` (`log_databases/logging_config.py:39-41`).
- Translation executed in a worker thread with `asyncio.wait_for` 3-second timeout (`utils/mapping_output.py:115`).

**Optimization techniques**
- Documented mechanisms: 2x PDF render scale producing "high-resolution image arrays"; YOLO cropping of card regions before OCR; anchor gate rejecting wrong documents "before invoking LLM extraction"; deterministic decoding (`temperature=0`, strict `json_schema`); `/no_think` which "disables internal reasoning preamble"; bounded translation latency (3 s, `max_retries=0`); explicit `gc.collect()` "to clear transient image matrices"; a small language model chosen to suit CPU-bound hardware ("Open-source approach").
- Rationale attributed to these mechanisms beyond the document's own wording (e.g., cropping limits Azure payload size, determinism avoids parse retries, `/no_think` reduces token count) is analyst inference.

**Performance improvements**
- No before/after comparison is documented. The integration test log (March 31, 2026) records end-to-end latencies between 5.07 s and 19.09 s across the six endpoints (per-endpoint ranges quoted verbatim in Section 8), against a stated design intent of extraction "with good accuracy within 10 secs timeframe" on CPU-bound hardware.

## 7. Challenges and Solutions

1. **Multi-page PDFs and mixed content (technical).** PDFs contain several cards plus unrelated pages. *Solution:* pypdfium2 rasterization at 2x, YOLOv8 cropping of each card, non-documents diverted to `skipped_images/`. *Alternatives:* fixed-grid splitting or classical contour detection (analyst suggestion), brittle against varied layouts.
2. **Fragmented OCR reading order (technical).** `prebuilt-read` returns line fragments with polygons rather than logically ordered text. *Solution:* `combine_text_lines` vertical clustering at 10 px plus horizontal sort. *Alternatives:* Azure `prebuilt-layout` reading order, or feeding polygons to the LLM (analyst suggestion).
3. **Wrong document uploaded to an endpoint (business).** Misfiled documents would yield plausible but wrong records. *Solution:* `DOCUMENT_ANCHORS` multi-token scoring rejects mismatches with a descriptive message before LLM inference. *Alternatives:* a YOLO classifier head or an LLM classification call (analyst suggestion); anchors are cheaper and deterministic.
4. **Unreliable LLM output format (technical; challenge inferred from the controls applied).** *Solution:* `temperature=0`, strict `json_schema` response_format, `/no_think`, schema from pipe-delimited field lists. *Alternatives:* regex/rule extraction, Azure custom extraction models, or function-calling with Pydantic validation (analyst suggestion).
5. **PII sovereignty and cloud token cost (business).** *Solution:* hybrid architecture: Azure performs OCR only; the Qwen SLM runs on a self-hosted Ollama endpoint. *Alternatives:* fully cloud (GPT/Gemini) or fully on-prem OCR such as Tesseract/PaddleOCR (analyst suggestion).
6. **Bilingual regulatory output (business).** *Solution:* GoogleTranslator derivation of five field families with a 3-second timeout and `max_retries=0` so translation cannot stall the request. *Alternatives:* LLM emits Arabic in the same pass, or Azure Translator (analyst suggestion).
7. **Credential hygiene (business).** *Solution:* hvac KV v2 reads from Vault (`retail-dev`); no plaintext secrets in the repository. *Alternatives:* deploy-time environment variables or a cloud KMS such as Azure Key Vault (analyst suggestion).
8. **Memory growth from image matrices (technical).** *Solution:* `gc.collect()` per request. *Alternative:* scoped context managers / explicit array deletion (analyst suggestion).
9. **Latency against the CPU-bound design intent (technical).** The document positions the SLM as one that "adapts to CPU bound" hardware with a "within 10 secs" intent, yet observed latencies in the March 31, 2026 test log reach 19.09 s on `residence_back` (the hardware used for that test run is not stated). Unresolved in the document; GPU serving, quantized Qwen variants or batched Azure calls are candidate fixes (analyst suggestion).

**Documented limitations and defects (honest talking points):**
- `python main.py` fails with `TypeError` because `main.py:81` evaluates `int(os.getenv('PORT'))` while `.env` defines lowercase `port = 5055`; the Uvicorn CLI command is the verified path.
- `utils/output_keys.py` is an orphaned schema module (superseded by `OUTPUT_DICT`) and contains a duplicated `'issue_place|issue_place'` field on `license_front`.
- NumPy version drift: 2.2.6 declared vs 2.5.2 installed in `.venv`.
- CORS is `allow_origins=['*']`, `allow_credentials=True`, `allow_methods=['*']`, `allow_headers=['*']` — permissive for a Basic-Auth service (analyst observation).
- One file per request (`len(file) == 1`); Azure rejects out-of-range image dimensions; the Ollama endpoint is an internal IP address (`http://172.20.132.97:11434/v1`; whether hard-coded or configured is not stated); Swagger UI and Ollama versions are "Not specified in the project" and the Qwen variant/size is not given.
- No Jupyter notebooks (`.ipynb`), scratch experiment scripts (`poc_*`) or draft prototypes were found in the repository; the only experimentation artifact is the orphaned `utils/output_keys.py`.

## 8. Impact Analysis

- **Business impact:** Provides a single standardized API for six UAE document faces consumed by underwriting, claims, onboarding and fleet systems; prevents misclassified documents from entering core systems; delegates credential lifecycle to Vault for compliance.
- **Productivity improvements:** Removes manual rekeying and manual PDF splitting/cropping; bilingual values generated without translators. No quantified productivity figure is given.
- **Accuracy improvements:** The document states extraction "may happen with good accuracy" but explicitly says overall extraction accuracy rates are "Not specified in the project." Qualitatively, determinism (`temperature=0`, JSON schema) and the anchor gate reduce format and misclassification errors.
- **Cost savings:** Percentage clerical cost savings and ROI are "Not specified in the project." Qualitatively, self-hosting the Qwen SLM reduces cloud token consumption relative to a cloud LLM.
- **Time savings:** Verbatim observed latencies per endpoint (March 31, 2026 test log): license_front 7.00–9.98 s; license_back 5.07–10.54 s; residence_front 8.27–13.66 s; residence_back 6.62–19.09 s; vehicle_front 7.71–8.86 s; vehicle_back 5.41–8.05 s. Throughput (documents per minute) is not specified.
- **User benefits:** Consuming systems receive normalized snake_case, bilingual JSON with descriptive, machine-readable error payloads (`messageType: "E"`), plus auto-generated Swagger/ReDoc documentation.

## 9. Interview Discussion Points

- Why split perception across a local YOLOv8 detector and Azure `prebuilt-read` rather than one OCR pass: cropping raises OCR quality, filters non-documents, and scopes Azure calls to card regions.
- How `combine_text_lines` works (10-px vertical agglomeration, then horizontal sort) and why polygon-based reading order matters before LLM extraction.
- The anchor-validation gate: how `DOCUMENT_ANCHORS` scoring works and why it runs before the LLM.
- Constrained decoding: `json_schema` strict mode, `temperature=0`, `/no_think`, pipe-delimited schemas per endpoint, and what happens when the SLM still hallucinates a value.
- Hybrid cloud/edge rationale: PII sovereignty and token cost drove a self-hosted Qwen via Ollama; the trade-off versus a stronger cloud LLM.
- Latency profile: 5.07–19.09 s observed in the March 31, 2026 test log against a design framed as CPU-bound with a 10-second intent (test hardware not stated); what dominates (Azure round-trip vs LLM decode vs translation) and how to get under 10 s consistently.
- The "Open-source approach": why a small open-weight model (Qwen via Ollama) and an open-source detector (YOLOv8) were chosen over a larger cloud LLM, and what accuracy you would expect to give up on CPU-bound hardware.
- The single-file-per-request contract (`len(file) == 1`) and multi-card PDF handling: how you would evolve the API to batch uploads or async job processing without breaking existing clients (analyst discussion point).
- Failure isolation for translation: 3-second thread timeout, `max_retries=0`, and why a non-critical enrichment step must never fail the request.
- Security posture: Vault KV v2 for Basic-Auth credentials versus wildcard CORS with credentials.
- The `PORT`/`.env` case mismatch bug and the orphaned `output_keys.py` with the duplicated `issue_place` field, and how you would fix them.
- Production hardening gaps (tests, CI/CD, containerization, model versioning, accuracy benchmarking) and the undocumented provenance of the YOLOv8 weights.

## 10. Architecture Explanation Points

Start with the contract: one image or PDF posted with Basic Auth to `/uae/<document_face>` returns normalized bilingual JSON or a structured error. Draw five boxes left to right. **FastAPI service** — middleware handles CORS, Vault-backed auth, per-request timing to a daily-rotating log, and `gc.collect()` on exit; one `create_route` factory produces all six endpoints. **YOLO splitter** — pypdfium2 renders PDF pages at 2x, OpenCV decodes images, a custom YOLOv8 model crops each card and dumps non-documents to `skipped_images/`. **Azure Document Intelligence** — `prebuilt-read` returns text lines with polygons; only cropped cards are sent. **Validation + ordering** — anchor tokens confirm the document type before any LLM cost, then 10-pixel vertical clustering rebuilds reading order. **Qwen via Ollama + mapper** — an OpenAI-compatible call at temperature 0 with a strict JSON schema from pipe-delimited field lists; keys normalized to snake_case, five field families translated to Arabic under a 3-second timeout.

Key decisions (stated in the document): perception on cloud, understanding on-prem (the document cites reduced cloud token consumption and PII data sovereignty); an "Open-source approach" with a small language model suited to CPU-bound hardware; determinism wherever the LLM touches output; document types as data, not code; every external call bounded or gated. Trade-offs (analyst assessment): observed 5.07–19.09 s latency versus a 10-second intent, dependence on Azure availability, a general-purpose translator instead of domain glossaries, no persistent store or queue, and permissive wildcard CORS. Improve next (analyst suggestions): GPU or quantized Qwen serving, async batching of Azure calls, a field-level accuracy benchmark, containerization plus CI/CD, tightened CORS, the `PORT`/`.env` config-key fix, retiring the orphaned schema file, and a YOLO-class-based document classifier where anchor heuristics prove brittle.


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


# Project 9: P&C Clause Recommendation Engine

Source: `PNC_Recommendation_documentation.md` (Document no. AT/PNC/V1, dated 01-Sep-2026). Client: Qatar Insurance Group (QIG).

## 1. Project Overview

- **Project name:** P&C Clause Recommendation Engine — an underwriting decision-support system for commercial property submissions.
- **Business domain:** Insurance — Property & Casualty commercial property underwriting at Qatar Insurance Group. Target users are commercial property underwriters (new submissions and renewals) and underwriting managers monitoring clause consistency and portfolio quality.
- **Problem statement:** Underwriters manually decide which risk clauses to attach to each commercial property policy. The review is time-intensive, inconsistent across underwriters, and error-prone under information overload or on unfamiliar risk classes. Omitted clauses create underinsurance and uninsured exposure gaps.
- **Project objective:** Automate clause gap analysis by combining (a) historical portfolio patterns retrieved via semantic vector search (SentenceTransformers `all-MiniLM-L6-v2` + FAISS `IndexFlatL2`) and ranked by a hybrid actuarial score (Recency Score x Frequency Score), with (b) contextual reasoning from a Google Gemini LLM constrained to a JSON schema — delivering recommendations in three tiers (Highly Recommended / Review Required / Optional), each with a 0.0-1.0 confidence and a plain-English underwriting justification.
- **Expected business outcome:** Faster submission processing (the document claims gap analysis moves "from hours to seconds"), standardised coverage terms across the property book, an auditable rationale per recommendation for compliance and peer review, and proactive identification of underinsured risks at inception. Strategic benefits cited: portfolio intelligence from aggregated recommendation patterns, improved loss-ratio performance, horizontal scalability ("without requiring changes to the recommendation logic"), and REST-based integration with underwriting platforms. The document also frames consistency as standardising coverage terms "aligned with underwriting guidelines", and asserts "measurable improvements across the underwriting lifecycle, from initial submission review through to policy issuance" — while supplying no measurements.
- **Key capabilities as named by the document:** *Automated Gap Analysis* (proactively detects omitted policy clauses, preventing underinsurance and uninsured exposure gaps); *Dual Intelligence (ML + LLM)* (combines empirical historical portfolio patterns with generative AI contextual underwriting reasoning); *Underwriting Efficiency* (turns time-intensive manual clause review into instant, ranked recommendations with transparent rationales); *Consistency & Compliance* (standardises coverage terms across property portfolios aligned with underwriting guidelines). The consistency claim is elaborated in the benefits section as "reducing inter-underwriter variability and improving portfolio quality metrics".
- **Document scope:** the source is technical system documentation (introduction, architecture, API reference, Streamlit guide, business value). It names no author, no team, no delivery timeline and no evaluation results.

## 2. Role and Responsibilities

The document does not name the candidate's role; responsibilities are inferred from what was built.

**Core responsibilities (inferred)**

- Designed the five-stage architecture (Client/Streamlit UI -> FastAPI Gateway -> NLP + FAISS Search -> Gemini AI + Scoring -> JSON Recommendations); each stage is described as independently deployable and REST-connected.
- Built the FastAPI microservice (Uvicorn, port 8085) with four endpoints — `GET /health`, `POST /load_data`, `POST /historical_response`, `POST /AI_response` — using Pydantic validation and auto-generated OpenAPI/Swagger docs.
- Implemented semantic retrieval: `all-MiniLM-L6-v2` embeddings of business activity descriptions (`AC_DESC`) indexed in FAISS `IndexFlatL2` for nearest-neighbour lookup of similar historical activities.
- Designed the hybrid scoring model (Recency Score x Frequency Score) over `TPI_UW_YEAR` and occurrence counts, normalised to a 0.0-1.0 confidence, plus the three-tier classification.
- Integrated Google Gemini (e.g. `gemini-2.0-flash`) for contextual gap analysis with responses constrained to an `AIRecommendation` JSON schema.
- Built the Streamlit underwriter UI (`streamlit/streamlit_app.py`) with activity selection, fallback Top-N slider, sort controls and tabbed results.

**Supporting tasks (inferred)**

- Defined the historical data contract: `Data/PnC_Clauses_df.csv`, cp1252 encoding, columns `TPI_UW_YEAR`, `CLAUSE_DESC`, `AC_DESC`; loaded and aggregated it with Pandas/NumPy.
- Designed the `/load_data` configuration schema (`FILENAME`, `BUSINESS_DESCRIPTION`, `PROPERTY_LOCATION`, `EXISTING_CLAUSES`, `LLM_MODEL`, `TOP_N_RECOMMENDATIONS`).
- Prompt and output-schema design for Gemini; exact-match-first with semantic fallback logic (fallback when fewer than 50 exact matches); rationale text generation citing year and occurrence count.
- Authored the technical documentation (architecture, API reference, UI guide, data requirements, business value) — authorship is inferred; the document carries no author name.

**Estimated ownership level: Senior AI Engineer (inferred)**

Rationale: the candidate owned the whole stack of a multi-component AI system — architecture, a four-endpoint REST service, an embedding + vector-search layer, a bespoke actuarial scoring model, a schema-constrained LLM integration, a data contract and a UI. The visible decisions (dual ML + LLM pipelines behind one response contract, structured output for parseability, exact-then-semantic fallback) are solution-level rather than ticket-level. Production readiness is partial — Uvicorn, Pydantic, `/health` and OpenAPI docs exist, but authentication, deployment, monitoring, tests and evaluation are undocumented — which keeps this below Solution Architect / Technical Lead. No team size is stated; a single-owner build is inferred.

## 3. End-to-End Workflow

**Processing stages**

1. **Configuration and data load (`POST /load_data`).** The client uploads a JSON configuration as a multipart file: `FILENAME` (absolute or relative path to the historical clause CSV), `BUSINESS_DESCRIPTION`, `PROPERTY_LOCATION`, `EXISTING_CLAUSES` (clauses already on the policy), `LLM_MODEL` (e.g. `gemini-2.0-flash`) and `TOP_N_RECOMMENDATIONS`. The service ingests the dataset and initialises parameters.
2. **Validation and routing.** Pydantic validates payloads; FastAPI orchestrates calls into the two intelligence pipelines.
3. **Historical retrieval (`POST /historical_response`).** Activity descriptions are embedded with `all-MiniLM-L6-v2`; FAISS `IndexFlatL2` returns nearest-neighbour activities. Exact activity matches are used first; when fewer than 50 exact matches exist, the Top-N similar-activity fallback is triggered (rule stated in the UI guide).
4. **Hybrid scoring.** Clauses attached to matched activities are scored as Recency x Frequency using `TPI_UW_YEAR` (most recent year the clause appeared) and occurrence frequency (submissions including the clause in the reference year), normalised to 0.0-1.0 and bucketed into Highly_Recommended / Review_Required / Optional.
5. **LLM gap analysis (`POST /AI_response`).** Gemini receives the business description and existing clauses (documented) and identifies missing coverages contextually; whether `PROPERTY_LOCATION` is passed to the model is not stated (inferred from the config schema only). The technology-stack section also credits Gemini with "underwriting rationale generation", from which it is *inferred* that the `reason` text on this leg is LLM-written rather than computed from year/count — the document does not say so explicitly. Output is constrained to the `AIRecommendation` schema and mapped to the same three keys.
6. **Response assembly.** Both endpoints return `{Highly_Recommended, Review_Required, Optional}` lists of `{clause, confidence, reason}`. The Streamlit UI renders three tabs with columns Sr, Clause Description, Underwriting Year, Occurrence Frequency, Score, Summary.

**Data flow**

- Inbound: the client/request layer is an "underwriter portal or Streamlit UI submitting risk profile parameters (business activity, property location, and existing policy clauses)", forwarded as "structured JSON payloads"; the JSON configuration itself arrives at `/load_data` as a multipart file upload.
- Internal: CSV (`Data/PnC_Clauses_df.csv`, cp1252) -> Pandas DataFrame (Pandas as the reader is inferred; the document only lists NumPy/Pandas for "data preprocessing, score computation, and result aggregation") -> `AC_DESC` text -> dense embeddings from `all-MiniLM-L6-v2` (dimension not stated) -> FAISS index -> matched activity rows -> NumPy/Pandas score computation -> tiered dictionaries.
- LLM leg: config fields -> Gemini prompt -> schema-constrained JSON -> tiered dictionaries.
- Outbound: standardised JSON to the client — "categorised clause recommendations with confidence ratings and underwriting rationale are assembled and returned as a standardised JSON object". The Streamlit UI also reads the CSV directly for its activity dropdown (and, inferred, its result tables); the UI guide describes it as letting underwriters "interactively query the recommendation engine without using the raw REST API".
- Index granularity is ambiguous in the source: Figure 1 labels stage 3 "Finds similar historical clauses" and the stage description says FAISS searches "against historical clause records", while the same paragraph says the SentenceTransformer "encodes business activity descriptions" and the UI guide describes the fallback as returning "similar-activity" results. Whether vectors represent activities or individual clause records is therefore not resolved by the document (analyst observation).

**Integrations and dependencies:** Google Gemini LLM (selectable via `LLM_MODEL`); SentenceTransformers `all-MiniLM-L6-v2`; FAISS `IndexFlatL2`; FastAPI, Uvicorn, Pydantic; NumPy, Pandas; Streamlit.

**Flow line**

`Streamlit UI / Portal -> FastAPI :8085 (Pydantic) -> /load_data (CSV + config) -> /historical_response (MiniLM embeddings -> FAISS IndexFlatL2 -> Recency x Frequency -> 3 tiers) | /AI_response (Gemini structured output, AIRecommendation schema -> 3 tiers) -> JSON {Highly_Recommended, Review_Required, Optional}`

**Architecture Summary**

The engine is a Python microservice fronting two parallel recommendation strategies behind one response contract. The empirical leg treats the historical clause book as a retrieval problem: activity descriptions are embedded, indexed in FAISS and queried per request; clauses attached to matching activities are ranked by a transparent score rewarding recency and prevalence. The generative leg asks Gemini what a given business, with a given set of existing clauses (and, inferred, the configured property location), is still missing, and forces the answer into a JSON schema. Both legs emit the same three tiers with per-clause confidence and rationale, so the UI and downstream systems consume them interchangeably. Streamlit gives underwriters interactive access; the REST/OpenAPI surface is described as enabling integration with underwriting platforms, portals and policy management systems.

## 4. Technologies and Tools Used

| Category | Tools (as stated in the document) |
|---|---|
| Programming languages | Python |
| Frameworks | FastAPI ("high-performance Python REST framework with automatic OpenAPI documentation"); Streamlit ("interactive Python-native web interface for underwriters" providing activity selection, sort controls and tabbed recommendation display) |
| Libraries | SentenceTransformers; FAISS; NumPy / Pandas (stated role: data preprocessing, score computation, result aggregation); Pydantic (validation and response schema enforcement); Uvicorn (ASGI server, "production-grade async request handling") |
| AI/ML models | `all-MiniLM-L6-v2` (SentenceTransformer embeddings); Google Gemini LLM (`gemini-2.0-flash` as example, set via `LLM_MODEL`) |
| OCR tools | Not stated in documentation |
| Databases | Not stated in documentation (data is a CSV, `Data/PnC_Clauses_df.csv`; FAISS `IndexFlatL2` is the vector index — in-memory by nature of that index type, not stated in the document) |
| Cloud platforms | Not stated in documentation (Gemini consumed as an external service; hosting not described) |
| APIs | Google Gemini API (structured output / JSON schema); own REST API at `http://<host>:8085` — `GET /health`, `POST /load_data`, `POST /historical_response`, `POST /AI_response` |
| DevOps tools | Not stated in documentation |
| Version control | Not stated in documentation |
| Deployment tools | Uvicorn (ASGI serving); `streamlit run streamlit/streamlit_app.py`. No container/orchestration tooling stated |
| Document processing tools | Not stated in documentation (inputs are JSON and CSV) |
| Automation tools | Not stated in documentation |

No pinned library versions are given.

## 5. Technical Skills Demonstrated

- **Software Engineering:** Multi-stage Python system split into independently deployable REST components; explicit CSV data contract (`TPI_UW_YEAR`, `CLAUSE_DESC`, `AC_DESC`) and a stable JSON response contract shared by two pipelines; handling of `cp1252` legacy encoding.
- **AI Engineering:** Retrieval (embeddings + FAISS) combined with generative reasoning (Gemini); parseable LLM output enforced via the `AIRecommendation` JSON schema; model selectable via `LLM_MODEL`.
- **Machine Learning:** Hybrid Recency x Frequency scoring normalised to 0.0-1.0 and used for three-tier bucketing; use of a pre-trained transformer for similarity rather than training.
- **NLP:** Dense sentence embeddings with `all-MiniLM-L6-v2`; semantic nearest-neighbour matching of activities with no exact string match; natural-language rationale text.
- **Computer Vision:** Not demonstrated in this project.
- **Data Engineering:** CSV ingestion with Pandas/NumPy (the document assigns them "data preprocessing, score computation, and result aggregation"); a mandatory input-schema contract — the CSV "must contain" `TPI_UW_YEAR` (underwriting year, integer), `CLAUSE_DESC` (clause description text) and `AC_DESC` (business activity description) — with a configurable source path (`FILENAME`, "absolute or relative"); aggregation of clause occurrence counts per underwriting year and derivation of the most recent year a clause appeared; exact-match-first retrieval with semantic fallback below 50 matches; handling of `cp1252` encoding on read.
- **Cloud:** Not demonstrated beyond consuming the Google Gemini API; hosting undocumented.
- **MLOps:** Limited — `/health` and `/load_data` initialisation endpoints; no registry, monitoring or evaluation pipeline described.
- **API Development:** FastAPI REST design with four endpoints, multipart JSON config upload, Pydantic validation, Uvicorn on port 8085, Swagger UI.
- **Prompt Engineering:** Structured-output prompting of Gemini over `BUSINESS_DESCRIPTION` and `EXISTING_CLAUSES` (the document names only the business description and existing clauses as Gemini inputs; use of `PROPERTY_LOCATION` is inferred from the config schema), constrained to `AIRecommendation`; the document also tasks Gemini with "underwriting rationale generation", i.e. the LLM writes the per-clause `reason` on its leg. The prompt text itself is not in the document.
- **System Design:** Gateway pattern over a five-stage pipeline; parallel empirical and generative paths behind a common schema; explicit tiering and confidence for auditability; UI decoupled from the API.

## 6. Detailed Technical Contributions

**Features implemented**

All four endpoints are served from the base URL `http://<host>:8085`, with a Swagger UI page referenced in the document (screenshot only).

- `GET /health` — "health check endpoint confirming API availability and operational connectivity".
- `POST /load_data` — accepts a **multipart file upload** carrying the JSON configuration (`FILENAME` = absolute or relative path to the historical clause CSV, `BUSINESS_DESCRIPTION` = commercial activity description for the submission, `PROPERTY_LOCATION` = location of the insured property, `EXISTING_CLAUSES` = list of clause descriptions already on the policy, `LLM_MODEL` = Gemini model identifier e.g. `gemini-2.0-flash`, `TOP_N_RECOMMENDATIONS` = maximum number of fallback similar-activity recommendations); "ingests and initialises the historical policy clause dataset and configuration parameters".
- `POST /historical_response` — "performs FAISS semantic search and hybrid scoring on historical clause utilisation data"; returns `Highly_Recommended`, `Review_Required`, `Optional` lists of `{clause, confidence, reason}`, where `confidence` is the hybrid score (0.0-1.0) "representing historical recency and frequency" and `reason` is a "plain-English explanation citing year and occurrence count".
- `POST /AI_response` — "leverages Google Gemini LLM to analyse business exposures and recommend missing property clauses", returning the identical three-key structure.
- Streamlit UI: sidebar "Select Search Activity" dropdown (all unique activities, default "Industries - Cement Plants"), "Top N Recommendations (Fallback)" slider (1-20), "Sort Clauses By" (`HYBRID_SCORE` default, `OCCURENCY`, `TPI_UW_YEAR`); main panel tabs Highly Recommended / Review Required / Optional with columns Sr, Clause Description, Underwriting Year, Occurrence Frequency, Score, Summary. Launched with `streamlit run streamlit/streamlit_app.py` from the project root; described as a way to query the engine "without using the raw REST API".

**Models used:** `all-MiniLM-L6-v2` for embeddings; FAISS `IndexFlatL2` for L2 nearest-neighbour search (the document cites sub-millisecond / millisecond-scale lookups; "exact" search is a property of the flat index type, not a statement in the document); Google Gemini via `LLM_MODEL` (example `gemini-2.0-flash`) with `AIRecommendation` structured output, used for both gap analysis and "underwriting rationale generation".

**Pipelines built:** Historical — CSV load -> embed `AC_DESC` -> FAISS index -> exact match, falling back to Top-N nearest activities under 50 exact matches -> aggregate by `TPI_UW_YEAR` and count -> Recency x Frequency score -> tier -> reason string. AI — config fields -> Gemini prompt -> schema-constrained JSON -> tier -> response.

**APIs integrated:** Google Gemini API (structured output). The engine's own REST API is designed for consumption by underwriting portals and policy management systems.

**Data extraction methods:** Read of `Data/PnC_Clauses_df.csv` (`cp1252`) — Pandas as the reader is inferred from the stated stack; required columns `TPI_UW_YEAR` (integer), `CLAUSE_DESC`, `AC_DESC`; derivation of "Underwriting Year" (most recent year a clause appeared) and "Occurrence Frequency" (submissions including the clause in the reference year).

**Validation logic:** Pydantic request validation and response schema enforcement (the gateway layer is described as "managing request validation, data loading, and orchestration across intelligence pipelines"); a documented input-data contract requiring `TPI_UW_YEAR` (integer), `CLAUSE_DESC` and `AC_DESC` in the CSV; a documented routing rule (fall back to Top-N similar activities only when fewer than 50 exact matches are found); Gemini responses constrained to the `AIRecommendation` JSON schema, which the document says ensures parseable output (whether any post-hoc parsing or re-validation also occurs is not stated). No explicit error handling is documented — nothing on missing/malformed columns, empty match sets, an activity absent from the dataset, Gemini API failures, timeouts or quota errors, or what `/historical_response` and `/AI_response` do if `/load_data` has not been called.

**Automation workflows:** On-request automated clause gap detection with rationale replaces manual clause-by-clause review.

**Optimization techniques (inferred from the stated stack; the document does not describe optimisation work):** Pre-trained embedding model plus a flat FAISS index, which the document credits with millisecond-scale / sub-millisecond matching; async ASGI serving via Uvicorn; exact-match-first, which would avoid the semantic fallback when the activity is well represented (inferred from the 50-match rule).

**Performance improvements:** The document claims sub-millisecond nearest-neighbour search and gap analysis "from hours to seconds"; no measured benchmarks, accuracy figures or before/after timings are provided.

## 7. Challenges and Solutions

The document does not describe problems encountered during development; the challenge framings below are inferred from the design choices it documents. Solutions cite documented mechanisms; alternatives are analyst suggestions.

1. **Rare or differently phrased risk classes (technical, inferred).** Exact matching on activity descriptions fails for paraphrased or sparse activities. *Solution:* embed `AC_DESC` with `all-MiniLM-L6-v2` and use FAISS `IndexFlatL2` to retrieve similar activities when fewer than 50 exact matches exist, sized by `TOP_N_RECOMMENDATIONS` (1-20). *Alternatives (analyst suggestion):* BM25/TF-IDF lexical matching; a larger embedding model such as `all-mpnet-base-v2`; hybrid lexical + dense retrieval with rank fusion.
2. **Ranking clauses underwriters trust (business + technical, inferred).** Pure frequency over-weights stale clauses; pure recency ignores prevalence. *Solution:* Recency x Frequency hybrid score exposed as `confidence`, with UI re-sorting by `OCCURENCY` or `TPI_UW_YEAR`. *Alternatives (analyst suggestion):* exponentially decayed frequency; learned ranking from underwriter acceptance; Bayesian shrinkage for low-count clauses. Tier thresholds and exact formulas are not stated.
3. **History cannot recommend what the portfolio never contained (business, inferred).** *Solution:* an independent Gemini pipeline reasoning over `BUSINESS_DESCRIPTION` and `EXISTING_CLAUSES` (plus `PROPERTY_LOCATION`, inferred). *Alternatives (analyst suggestion):* underwriting-maintained clause matrices; retrieval-augmented prompting that feeds the historical shortlist to the LLM for one fused answer.
4. **Free-text LLM output is unreliable to parse (technical, inferred from the documented rationale "ensuring parseable output").** *Solution:* structured output constrained to the `AIRecommendation` JSON schema, matching the historical endpoint's shape. *Alternatives (analyst suggestion):* few-shot prompting with regex post-processing and retries; Pydantic re-validation with automatic re-prompting.
5. **Auditability and compliance (business).** *Solution:* every recommendation carries `confidence` and a `reason` citing year and occurrence count; tier semantics are explicit (Highly Recommended = statistically dominant, omission is significant risk; Review Required = moderate support, judgement needed; Optional = low-frequency, specialist or atypical risks, with inclusion "driven by risk-specific factors identified during survey"). *Alternative (analyst suggestion):* persist recommendation logs with request context.
6. **Legacy data encoding (technical, inferred).** The CSV is read in `cp1252` encoding, which suggests a legacy export. *Solution:* explicit encoding on load. *Alternative (analyst suggestion):* one-time UTF-8 normalisation in an ingestion job.
7. **Statefulness of `/load_data` (technical, analyst observation).** `/load_data` "ingests and initialises the historical policy clause dataset and configuration parameters" while also carrying per-submission fields (`BUSINESS_DESCRIPTION`, `PROPERTY_LOCATION`, `EXISTING_CLAUSES`), implying the two response endpoints read process state set by an earlier call — which is in tension with the documented claim that the microservice "scales horizontally … without requiring changes to the recommendation logic". *Solution:* not addressed in the documentation; the current shape simplifies single-user operation. *Alternative (analyst suggestion):* stateless requests carrying the submission payload in each call, with the dataset and FAISS index loaded once at startup or from a shared cache.
8. **Serving two consumers with one engine (business, inferred).** The document targets both underwriters (interactive Streamlit) and "existing underwriting platforms, portals, and policy management systems" (REST). *Solution:* a single standardised JSON contract (`Highly_Recommended` / `Review_Required` / `Optional` of `{clause, confidence, reason}`) shared by both legs and both consumers. *Alternative (analyst suggestion):* keep the UI on the REST API rather than reading the CSV directly, so scoring logic exists in exactly one place.

**Documented limitations and open issues**

(The document states no limitations itself; the items below are analyst observations drawn from what it documents and omits.)

- `/load_data` appears to set state consumed by the two response endpoints (inferred), making the service stateful per process — in tension with the horizontal-scalability claim.
- The Streamlit UI "reads directly from `Data/PnC_Clauses_df.csv`" and exposes only historical controls; whether it calls the REST API or surfaces the Gemini pipeline is unclear.
- No authentication, rate limiting, error handling, tests, evaluation methodology or deployment target is documented.
- Tier thresholds, scoring formulas, FAISS distance cut-off and the Gemini prompt are not given.
- `/load_data` couples per-submission fields with dataset loading — an awkward shape for multi-user use.
- The UI selects the activity from the dataset's unique `AC_DESC` values (so an exact match always exists and the fallback fires only on the under-50 rule), whereas the API takes free-text `BUSINESS_DESCRIPTION`; the semantic fallback therefore matters most on the API path (inferred).
- The document says each of the five stages is "independently deployable" and REST-connected, yet all four endpoints live in one FastAPI service; how the stages are actually separated is not described (analyst observation).
- The text references Figure 1 (architecture), a Swagger UI screenshot and a Streamlit UI screenshot that are not present in the extracted text, so any detail visible only in those images is unavailable.
- `EXISTING_CLAUSES` is only named as an input to the Gemini leg; the `/historical_response` description never mentions it, so whether clauses already on the policy are excluded from the historical recommendations is undocumented (analyst observation).
- `PROPERTY_LOCATION` is collected in the `/load_data` configuration but no endpoint description states a consumer for it; location-based reasoning is therefore unverified.
- `TOP_N_RECOMMENDATIONS` exists both as a configuration field and as the UI's "Top N Recommendations (Fallback)" slider (1-20); which takes precedence, and whether the API caps the value, is not stated.
- Only one run instruction is documented — `streamlit run streamlit/streamlit_app.py` from the project root. No Uvicorn/API start command, host binding, worker count, environment variables or Gemini API-key handling is described; the API is only located by the base URL `http://<host>:8085`.
- Gemini cost, latency, rate limits and any fallback when the LLM is unavailable are not addressed, despite `/AI_response` being one of the two headline pipelines.
- Whether the FAISS index is built once at startup or rebuilt per `/load_data` call — and whether it is persisted — is not described.

## 8. Impact Analysis

The document provides no measured metrics — no accuracy, precision/recall, latency benchmarks, adoption or cost figures. The points below are its qualitative claims.

- **Business impact:** Proactive detection of omitted clauses to prevent underinsurance and uninsured exposure gaps; improved loss-ratio performance from adequate terms at inception; aggregated recommendation patterns as portfolio intelligence for product development.
- **Productivity improvements:** Gap analysis described as moving "from hours to seconds", enabling higher submission volume per underwriter (claim, not measured).
- **Accuracy improvements:** the document claims machine-learning-driven pattern matching "eliminates human oversight errors caused by information overload or unfamiliar risk classes" — an absolute claim with no supporting figure, evaluation set or error-rate baseline.
- **Consistency:** claimed to standardise coverage recommendations across the team, "reducing inter-underwriter variability and improving portfolio quality metrics" (no variability or quality metric is reported).
- **Cost savings:** Not stated.
- **Time savings:** Only the qualitative "hours to seconds" claim.
- **User benefits:** Ranked, tiered recommendations with rationale and confidence; consistency across underwriters, "aligned with underwriting guidelines"; an audit trail for compliance and peer review; interactive Streamlit UI plus a REST API for platform integration; underwriting managers gain a view of clause consistency and portfolio quality.
- **Scalability claim:** The FastAPI microservice is said to scale horizontally for peak submission volumes "without requiring changes to the recommendation logic" (claim, not demonstrated).
- **Scope of claimed value:** The document asserts "measurable improvements across the underwriting lifecycle, from initial submission review through to policy issuance" but supplies no measurements.

## 9. Interview Discussion Points

- Why two independent pipelines (historical + LLM) rather than one RAG pipeline? Explain the value of a transparent actuarial score alongside LLM reasoning, and what a fused design would look like.
- Walk through the hybrid score: how recency and frequency are normalised, why a product rather than a weighted sum, and how tier boundaries were set. Formulas are unstated in the document, so bring concrete numbers.
- Why `all-MiniLM-L6-v2` and `IndexFlatL2`? Size of the clause set, why a flat exact index suffices, and when you would move to IVF/HNSW or a larger model.
- The exact-match-first / semantic-fallback rule (under 50 exact matches -> Top-N, slider 1-20): how 50 was chosen and what happens for wholly novel activities.
- Gemini structured output: how `AIRecommendation` is defined, behaviour on schema violations or empty responses, and handling of hallucinated clause names.
- Statefulness of `/load_data`: how state persists between calls, implications for concurrent underwriters, and refactoring to stateless per-request payloads to make horizontal scaling real.
- Evaluation: how you would measure quality (underwriter acceptance rate, precision@k against subsequently bound policies) given no metrics were captured.
- Data quality: `cp1252` encoding, the `OCCURENCY` field naming, and (analyst speculation, not in the document) whether clause descriptions were inconsistently worded and needed normalisation.
- Auditability: how `reason` strings are produced for the historical versus LLM legs, and how you would log recommendations.
- Productionisation gaps: authentication, Gemini key handling, containerisation, monitoring — own them as next steps.
- Prompt design: what context goes to Gemini and how it is stopped from re-recommending clauses already on the policy.
- Rationale provenance: on the historical leg the `reason` is computed from year and occurrence count, on the AI leg Gemini writes it ("underwriting rationale generation") — how do you keep LLM-written rationales factual and reviewable for compliance?
- "Independently deployable" stages versus one FastAPI service holding all four endpoints: what independent deployment would actually require (separate embedding/index service, separate LLM service) and whether it is worth it at QIG's volumes.
- Two input paths: the UI picks an activity from the dataset's `AC_DESC` values while the API takes free-text `BUSINESS_DESCRIPTION` — how the under-50 fallback behaves on each, and how you would handle a wholly novel description.
- Renewals versus new submissions: the target users include renewal reviews; how `EXISTING_CLAUSES` is populated for a renewal and whether the engine should flag clauses to remove as well as add (not addressed in the document).
- Deduplication: `EXISTING_CLAUSES` is documented only as a Gemini input — does the historical leg filter out clauses already on the policy, and how would you guarantee a "gap analysis" never recommends something already attached?
- API ergonomics: `/load_data` takes the configuration as a *multipart file upload* rather than a JSON request body. Justify the choice (file-based config re-use, drag-and-drop from the UI) versus the cost (harder to call, no Pydantic body model in the OpenAPI schema, awkward for machine-to-machine integration with the underwriting platform the doc targets).
- Index granularity: the doc says the transformer "encodes business activity descriptions" but that FAISS searches "against historical clause records" — be ready to say precisely what one vector represents, how many vectors there are, and how clause rows are grouped per activity.
- The "sub-millisecond" and "millisecond-scale" search claims: how you would actually measure them (index size, batch vs single query, embedding time excluded or included), and why embedding latency, not search, usually dominates.
- Gemini operations: API-key handling, per-request cost, rate limits, timeout/retry policy, and what `/AI_response` returns when the LLM is unavailable — none of this is in the document, so present it as a designed next step.
- Data freshness: the CSV is loaded on demand via `/load_data`; how often the historical clause book is refreshed, whether the FAISS index is rebuilt or cached, and how you would move to an incremental refresh.
- `PROPERTY_LOCATION` is captured but has no documented consumer — explain what location-aware recommendation would add (nat-cat, flood, seismic clauses) and how you would evidence it.

## 10. Architecture Explanation Points

Start with the user: an underwriter has a commercial property submission — activity, location, and the clauses already on the policy — and needs to know what is missing. Draw five boxes: Client (Streamlit UI or portal) -> FastAPI gateway (port 8085, Pydantic, four endpoints) -> NLP + FAISS retrieval -> Gemini + scoring -> JSON out.

Explain the two legs. The empirical leg embeds activity descriptions with `all-MiniLM-L6-v2`, stores them in a FAISS `IndexFlatL2`, finds the same or nearest activities, gathers the clauses historically attached to them, and ranks each by Recency x Frequency from `TPI_UW_YEAR` and occurrence counts, normalised to a 0-1 confidence. The generative leg sends the submission context to Gemini and forces a schema-constrained `AIRecommendation` response. Both produce the same three tiers with `clause`, `confidence`, `reason`, so consumers see one contract.

Key decisions: separate historical and AI endpoints so underwriters can compare a transparent statistical view with contextual reasoning; structured output (`AIRecommendation` JSON schema) to make the LLM safe to integrate — the document's own justification is "ensuring parseable output"; exact-match-first with a Top-N semantic fallback under 50 matches for rare activities; a runtime-selectable model via `LLM_MODEL` so Gemini versions can be swapped without a code change; a shared three-tier response contract so the UI and downstream systems consume either leg identically; and per-clause `confidence` + `reason` for auditability, with the same three tiers surfaced in the UI as tabs and re-sortable by `HYBRID_SCORE`, `OCCURENCY` or `TPI_UW_YEAR` so an underwriter can interrogate the ranking rather than accept it.

Why a hybrid score at all: the document's tier semantics carry the underwriting logic — Highly Recommended = "statistically dominant clauses supported by high recency and frequency … omission poses significant underwriting risk"; Review Required = "moderate historical support … underwriter judgement is required"; Optional = "low-frequency clauses appropriate for specialist or atypical risks … driven by risk-specific factors identified during survey". Recency alone would recommend one-off clauses from last year; frequency alone would keep recommending clauses the book has moved away from. The product of the two is the cheapest interpretable way to require both, which matters more than raw ranking quality because an underwriter has to defend the recommendation in peer review.

Trade-offs (analyst inferences; the document states none): a flat FAISS index (`IndexFlatL2`) gives exact results and zero tuning but scales linearly with the corpus, so it is the right call only while the clause book is small — IVF/HNSW would trade recall for sub-linear search later; `all-MiniLM-L6-v2` is small and fast (and keeps the whole thing CPU-friendly) at the cost of the semantic precision a larger model such as `all-mpnet-base-v2` would give on paraphrased activity names; the hybrid score is interpretable, though its formula, normalisation and tier boundaries are undocumented, so nothing shows the tiers were calibrated; the LLM covers novel risks but nothing in the document says its `confidence` is calibrated against portfolio data, and the two legs' confidences are therefore not comparable despite sharing a field name; `/load_data` couples configuration with submission state, which simplifies single-user use but complicates concurrency and undercuts the horizontal-scaling claim; taking that configuration as a multipart file upload rather than a JSON body is convenient for a file-driven UI but awkward for the underwriting-platform integration the document targets; the "independently deployable stages" claim is aspirational relative to the single four-endpoint service described; and the Streamlit UI reads `Data/PnC_Clauses_df.csv` directly rather than going through the API, which is convenient for underwriters but risks duplicating retrieval/scoring logic in two places and means the UI never exercises the Gemini leg.

Next improvements: stateless requests carrying the submission in each call, with the dataset and FAISS index loaded once at startup; fusing the legs (feed the historical shortlist to Gemini to explain or extend it, which also constrains it to real clause names); filtering `EXISTING_CLAUSES` out of the historical leg so a "gap analysis" never returns a clause already attached; giving `PROPERTY_LOCATION` a real consumer (nat-cat/flood clause rules); calibrating tiers against underwriter acceptance; and adding authentication, Gemini key management with retry/fallback, containerised deployment, recommendation logging and an offline evaluation set built from bound policies.


# Project 10: Multilanguage Speech-to-Text Voice Data Capture

Source: `SpeechToText_Documentation.md` (Document no. AT/STT/V1, dated 01-Sep-2026). Client: Qatar Insurance Group.

## 1. Project Overview

- **Project name:** Multilanguage - Speech to text (internal name "SpeechToText"; the Streamlit page titles are "Voice Extractor" and "Conversational Voice Assistant").
- **Business domain:** Insurance — motor claims. The document lists "Motor claim products" as the target users; the fields captured are vehicle-insurance attributes.
- **Problem statement (inferred from the Business Benefits section):** Motor-claim data entry means typing vehicle details into a form field by field, in a fixed language, in one pass. The project replaces this with voice capture: a browser recording is sent to a multimodal LLM with a JSON-schema prompt, and the returned values populate the form automatically in whatever language the speaker uses.
- **Project objective:** Build lightweight voice-to-form prototypes that (a) extract thirteen vehicle-insurance attributes from one recording into an editable table, (b) collect a smaller set of required fields through a multi-turn spoken conversation that asks only for what is missing, and (c) export a machine-readable `result.json`. A stated strategic aim is fast evaluation of three voice UX patterns — table-based, status-display, and conversational extraction — before committing to one.
- **Expected business outcome:** Hands-free, incremental, correctable data entry for motor-claim products with immediate visual feedback; multilanguage coverage; near-zero infrastructure footprint (no database, broker, or external state store — a single `streamlit run` command). The document frames the target as "pilot or internal use" and states that quantified figures (throughput, accuracy, cost savings, processing time) are not available, and that "no metrics, benchmarks, log files, or test results are present in the project".

## 2. Role and Responsibilities

The document never names the candidate's role; everything below is inferred from what was built and how it is described. The closest thing to a documented scope statement is the Architecture Component Descriptions table, which assigns the Streamlit Application responsibility for "session state management, deduplication of audio submissions, prompt construction, calling the Gemini SDK, parsing the JSON response, and rendering the editable table or chat history" — the responsibilities below map onto that list plus the prompt/model choices recorded in the Technology Stack table.

**Core responsibilities (inferred)**

- Designed the end-to-end audio-to-JSON pipeline: browser capture to in-memory WAV bytes, multimodal prompt with an explicit JSON schema, Gemini call with forced JSON response MIME type, parse-and-merge into session state, re-render.
- Built two independent Streamlit applications (`app.py`, `chat_interaction.py`) plus one standalone utility, each in a single file under 110 lines.
- Selected and wired the LLM models per use case: `gemini-3.1-flash-lite-preview` for one-shot 13-field extraction, `gemini-2.0-flash-lite` for the conversational assistant, both via the `google-genai` SDK.
- Designed the prompts: a schema-constrained extraction prompt for the vehicle fields, and a stateful conversational system prompt that embeds the full current `memory` dict as JSON so the model asks only for missing fields and returns `{response, field values, completed}`.
- Implemented session-state logic: deduplication of audio submissions (`audio_bytes != st.session_state.last_audio`), incremental merge (only truthy extracted values overwrite; implemented in all three apps), completion detection (`data.get('completed', False)`), and immediate refresh via `st.rerun()` (documented for the extractor and the status-display utility).
- Built the UI layer with Streamlit primitives: `st.audio_input`, `st.data_editor`, `st.chat_message`, `st.json`, `st.spinner`, status alerts, and `st.download_button` producing `result.json`.
- Delivered multilanguage behaviour: the system adapts to the consumer's spoken language; not restricted to English.

**Supporting tasks (inferred)**

- Dependency management across two `requirements.txt` files (root, unpinned; `4-6-2026/requirements.txt` with Streamlit 1.57.0 and pandas 3.0.3).
- Migrated microphone capture from the third-party `streamlit-mic-recorder` package to Streamlit's built-in `st.audio_input` (old import commented out; package still listed in root requirements).
- Wrote the system documentation (AT/STT/V1): architecture diagram, component and stack tables, UI guide, behavioural notes, and an explicit statement that no metrics, logs, or tests exist.
- Error-handling decisions per app (detailed in Section 6, Validation logic).

**Estimated ownership level: AI Engineer (inferred)**

Rationale: the candidate appears to own the entire project — architecture, prompt design, model selection, session-state logic, UI, and documentation — with no evidence of a wider team. The scope is deliberately small: three single-file applications under 110 lines each, no HTTP API, no database, no deployment pipeline, no tests, one external service (Gemini), and the document positions the work as "rapid prototyping of voice-to-form workflows" for "pilot or internal use". This is complete AI-engineering work on a multimodal LLM prototype, but it lacks the multi-service orchestration, production hardening, or leadership evidence needed for Senior AI Engineer, Solution Architect, or Technical Lead on this project alone.

## 3. End-to-End Workflow

**Processing stages**

1. **Audio capture.** The user speaks into the system microphone through the browser. Streamlit's built-in `st.audio_input` widget (`'Record your details'` in the extractor, `'Speak...'` in the assistant) returns the recording to the Streamlit server as WAV bytes held in memory. No client-side JavaScript was written.
2. **Deduplication gate.** Because Streamlit re-runs the script on every interaction, the app compares the new bytes against `st.session_state.last_audio`; only a changed recording proceeds.
3. **Prompt construction.** The application builds a text prompt specifying the required JSON schema. In the extractor this is the fixed list of thirteen vehicle fields. In the assistant the system prompt also embeds the full current `memory` dict as JSON, so the model knows which fields are already collected.
4. **Gemini call.** Audio bytes plus the text prompt are sent through the `google-genai` SDK to `gemini-3.1-flash-lite-preview` (extractor) or `gemini-2.0-flash-lite` (assistant), with the response MIME type forced to JSON. A spinner (`'Extracting info...'` / `'Understanding your voice...'`) is shown during the call.
5. **Parse and merge.** The JSON payload is parsed. Only non-empty (truthy) extracted values overwrite existing session-state values, so fields not mentioned in this recording are preserved; the document states this incremental merge is implemented in all three audio applications. In the assistant, the reply additionally contains a natural-language `response` string, updated field values merged into `memory`, and a boolean `completed` flag read with `data.get('completed', False)` (defaults to False if the key is absent).
6. **Re-render.** In the extractor and the status-display utility, `st.rerun()` is documented to trigger an immediate UI refresh after a successful extraction: the extractor shows the editable `st.data_editor` table and the utility its markdown field display. The assistant re-renders `chat_history`, the status alert (`st.warning` / `st.success` / `st.info`), and `st.json(st.session_state.memory)`; the document does not state whether the assistant also calls `st.rerun()`.
7. **Export.** All three applications expose a download button producing `result.json` (`st.download_button('Submit & Download JSON')` in the extractor, `st.download_button('⬇️  Submit')` in the assistant; for the standalone utility the document says only "download button" — neither its label nor its widget call is documented). The document calls `result.json` "a machine-readable JSON file usable by downstream systems"; no downstream consumer is named.

**Data flow**

- Browser → Streamlit server: WAV audio bytes (in memory, no file written).
- Streamlit → Gemini: audio bytes + text prompt (with JSON schema; in the assistant, also the serialized `memory` state).
- Gemini → Streamlit: JSON object (field values; for the assistant, `response`, field values, `completed`).
- Session state keys named in the document: `last_audio`, `memory`, `chat_history`, `completed` (plus the extractor's field values). A `status` value is set to `'idle'` on exceptions in the assistant; the document does not say where it is stored. Streamlit → user: `result.json` download.

**Integrations and dependencies**

- Google Gemini API via the `google-genai` SDK (unpinned); Streamlit (1.57.0 in `4-6-2026/requirements.txt`) for UI, audio capture, and session state; pandas 3.0.3 listed under "Data handling" (its use for the data-editor table is inferred); `streamlit-mic-recorder` listed but superseded. The document's "Backend & API" section lists only "Streamlit" — the Streamlit process is the entire backend.
- No database, message broker, external state store, or HTTP API. The document is explicit on the last point: "This project does not expose an HTTP API. Both audio applications are Streamlit UIs."
- **Run instructions (document, verbatim):** `streamlit run app.py` for the Voice Extractor (13 vehicle fields) and `streamlit run chat_interaction.py` for the Conversational Voice Assistant. No launch command, file name, or entry point is documented for the third (standalone utility) application. No environment variables, API-key configuration, container image, service manager, or hosting target is described anywhere in AT/STT/V1.

**ASCII flow**

`Browser mic (st.audio_input) -> WAV bytes -> Streamlit app (dedup gate, prompt + JSON schema [+ memory]) -> google-genai SDK -> Gemini flash-lite (forced JSON MIME) -> JSON -> parse/merge into st.session_state -> st.rerun() -> data_editor / chat + st.json -> result.json download`

**Architecture Summary**

A deliberately minimal three-tier pipeline: a browser microphone front end provided entirely by Streamlit, a Python process that owns session state and prompt construction, and a remote multimodal LLM that does audio understanding and structured extraction in one call. No separate ASR step is described — the audio bytes are "posted directly" to Gemini with a JSON-schema prompt, so transcription and extraction are collapsed into a single JSON-constrained request; that this single-call design is what makes the documented multilanguage behaviour essentially free is an analyst inference (the document asserts multilanguage support but does not explain its mechanism). State lives only in Streamlit session state and is exported as JSON. The design trades production robustness (no persistence, API, tests, or thorough error handling) for simplicity: one file under 110 lines per UX variant, the model as a single string literal, one command to start.

## 4. Technologies and Tools Used

| Category | Technology / detail |
| --- | --- |
| Programming languages | Python |
| Frameworks | Streamlit (unpinned in root `requirements.txt`; pinned 1.57.0 in `4-6-2026/requirements.txt`) — supplies the entire stack: the browser microphone widget `st.audio_input` (returns WAV bytes to the server, "no client-side JavaScript is written by the project"), session state, all UI widgets, and the process the document's "Backend & API" section names as the whole backend |
| Libraries | `google-genai` SDK (unpinned); pandas (pinned 3.0.3 in `4-6-2026/requirements.txt`); `streamlit-mic-recorder` (listed in root requirements, superseded, import commented out) |
| AI/ML models | `gemini-3.1-flash-lite-preview` (app.py); `gemini-2.0-flash-lite` (chat_interaction.py) |
| OCR tools | Not stated in documentation |
| Databases | None — document explicitly states no database or external state store |
| Cloud platforms | Google Gemini API — described as a "remote cloud service accessed via the google-genai SDK"; hosting platform for the Streamlit apps not stated in documentation, and no API-key management, authentication, data-residency or audio-retention detail is given |
| APIs | Google Gemini API (consumed). No HTTP API exposed by the project; the document's "Backend & API" entry reads only "Streamlit" |
| DevOps tools | Not stated in documentation |
| Version control | Not stated in documentation |
| Deployment tools | `streamlit run app.py` / `streamlit run chat_interaction.py` (the only two documented entry points; none given for the third utility); no container, CI/CD, process manager, or other deployment tooling stated in documentation |
| Document processing tools | Not stated in documentation |
| Automation tools | Not stated in documentation |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Single-file Python applications separating capture, prompt, call, merge, and render; dependency pinning in a dated requirements file (Streamlit 1.57.0, pandas 3.0.3) alongside an unpinned root file; removing a third-party dependency in favour of a framework built-in (`streamlit-mic-recorder` → `st.audio_input`, import commented out in all current application files); idempotency guard against Streamlit re-runs; the same linear skeleton reused across three applications.
- **AI Engineering:** Multimodal LLM integration (audio + text in one request) via `google-genai`; forced JSON response MIME type; per-task model selection (`gemini-3.1-flash-lite-preview` vs `gemini-2.0-flash-lite`); model string as a single literal for model-agnostic design.
- **Machine Learning:** Not demonstrated in this project — no training, fine-tuning, or evaluation; pre-trained hosted models only.
- **NLP:** Speech understanding and slot-filling for thirteen vehicle-insurance fields; multi-turn dialogue management with memory-aware prompting; multilanguage adaptation to the speaker's language; completion detection via a boolean flag.
- **Computer Vision:** Not demonstrated in this project.
- **Data Engineering:** In-memory WAV byte handling; incremental merge semantics (truthy values overwrite, missing fields preserved); structured `result.json` export for downstream systems (all three apps); pandas for data handling (its backing of the editable table is inferred).
- **Cloud:** Consumption of a hosted generative-AI API (Google Gemini). No cloud deployment or infrastructure described.
- **MLOps:** Not demonstrated in this project — the document states no metrics, benchmarks, logs, or test results exist.
- **API Development:** Not demonstrated in this project — the project exposes no HTTP API; the skill shown is API consumption (Gemini SDK).
- **Prompt Engineering:** Schema-specifying extraction prompt; stateful system prompt embedding the current `memory` JSON so the model asks only for missing fields; structured reply contract `{response, fields, completed}`; prompt-plus-audio pattern "not tied to Gemini-specific features beyond the SDK call", with the model name as a single string literal per file so a model swap is a one-string change.
- **System Design:** Zero-infrastructure design (no DB, broker, or state store); Streamlit session state as the single source of truth; three UX patterns built as interchangeable prototypes for evaluation; explicit deduplication and rerun handling for Streamlit's execution model.

## 6. Detailed Technical Contributions

**Features implemented**

- *Voice Extractor (`app.py`):* page title "Voice Extractor" with 🎙️ icon; single-panel centered layout, no sidebar; `st.data_editor` two-column table (Field | Value) for 13 vehicle fields with the Field column disabled and Values editable; `st.audio_input('Record your details')`; `st.spinner('Extracting info...')`; `st.download_button('Submit & Download JSON')` producing `result.json`.
- *Conversational Voice Assistant (`chat_interaction.py`):* `st.title('🎙️ Conversational Voice Assistant')`; `st.caption('Speak naturally. The AI will collect your details conversationally')`; chat history rendered via `st.chat_message` for each entry in `st.session_state.chat_history`; status alert via `st.warning` / `st.success` / `st.info`; `st.audio_input('Speak...')`; `st.spinner('Understanding your voice...')`; `st.subheader('Collected Information:')` followed by `st.json(st.session_state.memory)`; `st.download_button('⬇️  Submit')` producing `result.json`. Initial assistant greeting listing the three required fields is appended once when `chat_history` is empty.
- *Third UX pattern (inferred mapping):* the document's "status-display extraction" with a "markdown field display" appears to be the standalone utility; its file name is not given. The document does state that the incremental merge, the `st.rerun()`-driven refresh of the markdown field display, and a `result.json` download button are implemented in it as well ("all three audio applications").
- *Multilanguage:* "It's able to adapt the consumer's language and fill the form. Not restricted to English only." No supported-language list, tested languages, or language-detection step is documented.
- *Delivery/run surface:* two documented entry points, `streamlit run app.py` and `streamlit run chat_interaction.py`; "This project does not expose an HTTP API. Both audio applications are Streamlit UIs." Both apps use a single-panel centered layout with no sidebar and the 🎙️ icon.

**Models used**

- `gemini-3.1-flash-lite-preview` — one-shot extraction of 13 vehicle-insurance attributes, forced JSON response MIME type.
- `gemini-2.0-flash-lite` — multi-turn conversational collection, forced JSON response MIME type.

**Pipelines built**

- Linear audio-to-JSON pipeline (Section 3 ASCII flow); identical skeleton across all applications.

**APIs integrated**

- Google Gemini API through the `google-genai` SDK (audio bytes + text prompt in, JSON out). No HTTP API is exposed.

**Data extraction methods**

- Direct multimodal extraction: audio bytes are "posted directly" to Gemini together with the text prompt; no separate transcription step is described. Gemini performs audio understanding and field extraction in a single call constrained to JSON.
- Conversational extraction: each turn returns `response` (natural-language string), updated field values, and `completed` (bool). The system prompt carries the full current `memory` JSON so the model targets only missing fields.

**Validation logic**

- Deduplication: `audio_bytes != st.session_state.last_audio` prevents reprocessing on Streamlit re-runs.
- Incremental merge: only non-empty (truthy) extracted values overwrite existing session-state values; unmentioned fields are preserved. Documented as implemented in all three audio applications.
- Completion detection: `st.session_state.completed = data.get('completed', False)`; defaults to False when the key is absent.
- Human-in-the-loop: the extractor's Value column is editable before download; the Field column is disabled so "user cannot edit field names" (document), which keeps the 13-field output schema fixed for downstream consumers (inferred).
- Not documented: any schema validation of the returned JSON, type checking or normalisation of extracted values, required-field enforcement before download, retry on a malformed response, or a confidence/uncertainty signal. The only checks recorded in AT/STT/V1 are the four above plus the model-side constraint of the forced JSON response MIME type.
- Error handling: `app.py` swallows JSON-parsing exceptions with a bare `except: pass` (line 34); `chat_interaction.py` sets status to `'idle'` and displays `st.error`.

**Automation workflows**

- `st.rerun()` after each successful extraction refreshes the UI without user action (documented for the extractor's data table and the utility's markdown field display).
- Automatic greeting injection on first load of the assistant: a welcome message listing the three required fields is appended once when `chat_history` is empty.
- Assistant status alert (`st.warning` / `st.success` / `st.info`) reflects the current turn state; on exception the status is set to `'idle'` and `st.error` is shown.

**Optimization techniques**

- Flash-lite variants for both apps (low-latency/low-cost tier, inferred from naming; no benchmark given).
- Forced JSON MIME type constrains the model to JSON output (analyst inference: this removes the need for regex/text extraction from prose, although JSON parsing can still fail — hence the `except: pass` note).
- Dedup gate "prevents reprocessing on Streamlit re-runs" (document); that this avoids redundant paid API calls is inferred.
- Single-string model configuration allows model swaps without other code changes.

**Performance improvements**

- None quantified; the document states no throughput, accuracy, processing-time, or cost figures exist.

## 7. Challenges and Solutions

**1. Streamlit re-execution causing duplicate LLM calls (technical).**
Streamlit re-runs the script on every widget interaction, so one recording could be sent to Gemini repeatedly. *Solution:* compare new bytes to `st.session_state.last_audio` and process only on change. *Alternatives (analyst suggestion):* hash the audio and cache the response with `st.cache_data`; use `st.form` or fragments to isolate re-runs.

**2. Capturing a complete record from partial or corrected multi-utterance speech (technical).**
Users rarely state all 13 fields cleanly at once (analyst framing — the document records the capability, "a user can speak additional details or corrections across multiple submissions", not the underlying problem). *Solution:* persist session state across recordings and merge only truthy values, so later recordings add or correct without wiping earlier values; the extractor table is editable for manual fix-ups. *Alternatives (analyst suggestion):* per-field confidence with a review flag; a diff view per recording; a "clear field" voice intent.

**3. Guiding the user to supply missing information without a rigid form (business).**
*Solution:* the assistant embeds the current `memory` JSON in the system prompt so the model asks only for what is missing, returns a spoken-style `response`, and signals `completed`. *Alternatives (analyst suggestion):* rule-based dialogue manager with slot tracking outside the LLM; function calling to update fields deterministically.

**4. Serving customers in multiple languages (business).**
*Solution:* one multimodal Gemini call handles audio in the speaker's language and returns the same JSON schema; no per-language ASR or translation step was built. *Alternatives (analyst suggestion):* separate ASR (e.g., Whisper) plus translation plus text extraction — more components and latency, but auditable.

**5. Getting reliably parseable structured output from an LLM (technical).**
*Solution:* force the JSON response MIME type and specify the schema in the prompt. *Alternatives (analyst suggestion):* Pydantic/JSON-schema validation with retry; Gemini structured-output schema objects instead of a prose schema.

**6. Keeping the pilot deployable with no infrastructure (business).**
*Solution:* no DB, broker, or state store; single `streamlit run`; export to `result.json`. *Trade-off (analyst observation):* nothing is persisted server-side; no audit trail. *Alternatives (analyst suggestion):* SQLite or a file-backed session store for durability with the same single-command start; a thin FastAPI wrapper if downstream systems need to pull results rather than receive a downloaded file.

**Documented limitations and defects (interview talking points)**

- `app.py` line 34 silently swallows JSON-parsing exceptions with a bare `except: pass`, so a malformed model response yields no user feedback — inconsistent with `chat_interaction.py`, which surfaces `st.error`.
- Root `requirements.txt` is unpinned; the superseded `streamlit-mic-recorder` is still listed although its import is commented out.
- No metrics, benchmarks, logs, or tests exist in the project.
- No HTTP API, so downstream integration is only via the downloaded file.
- Two model strings across the two apps (one a preview model) with no stated rationale; preview models carry deprecation risk (analyst observation).
- The document does not state whether `chat_interaction.py` calls `st.rerun()`; immediate refresh via `st.rerun()` is documented only for the extractor's data table and the utility's markdown field display.
- Two divergent dependency files: the root `requirements.txt` pins nothing, while `4-6-2026/requirements.txt` pins Streamlit 1.57.0 and pandas 3.0.3. The `google-genai` SDK is unpinned in the root file and no version is given anywhere, so the runtime environment is not reproducible from the document alone.
- The third application is never named. It is referred to only as "one standalone utility" / the "status-display extraction" pattern with a "markdown field display"; no file name and no launch command are documented for it, although the document counts it in "all three audio applications" for the merge, the `st.rerun()` refresh and the `result.json` download.
- The document's two "### Streamlit UI" subsections are empty headings — no screenshots or UI walkthrough are included in AT/STT/V1, so the interface is described only in prose widget lists.
- Nothing is documented about API-key handling, authentication in front of the Streamlit apps, audio retention, consent capture, or data residency, even though customer voice recordings are sent to a third-party cloud service (analyst observation: the document is silent on this, so it is a documentation gap rather than a stated defect).
- No validation of the model's JSON beyond the forced response MIME type: no schema check, type normalisation, required-field enforcement, retry, or confidence signal is documented.
- Neither the 13 vehicle-insurance field names, the three required conversational fields, the prompt text, nor the JSON schema is reproduced in the document.
- Figure 1 draws four stages (Browser → Streamlit Application → LLM → JSON) while the Architecture Component Descriptions table names only three components; the figure is the only architecture artefact provided.

## 8. Impact Analysis

The document states explicitly: "Quantified figures (throughput, accuracy rates, cost savings, processing time) are not available. No metrics, benchmarks, log files, or test results are present in the project." All impact below is qualitative and taken from the document's Business Benefits section.

- **Business impact:** Hands-free, multilanguage data entry for motor-claim products; three voice UX patterns for rapid evaluation; zero-infrastructure deployment suited to pilots and internal use.
- **Productivity improvements:** Users speak instead of typing; values are extracted "without manual transcription"; the assistant does not require all information in one utterance.
- **Accuracy improvements:** Not measured. Immediate visual feedback and an editable table let users verify and correct before download.
- **Cost savings:** Not measured. No database/broker/state store reduces deployment complexity; flash-lite models and the dedup gate limit API spend (inferred).
- **Time savings:** Not measured. `st.rerun()` gives immediate feedback; one command starts the stack.
- **User benefits:** Language-flexible interaction; add or correct details across recordings; machine-readable `result.json` for downstream systems.
- **Strategic benefits (document, Section 4):** *Rapid prototyping of voice-to-form workflows* — three UX patterns (table-based, status-display, conversational), each "in a single file of under 110 lines", supporting "fast evaluation of different interaction models before committing to one approach". *Model-agnostic prompt architecture* — "the Gemini model string is a single literal in each file. Switching to a different model variant requires changing one string", and "the prompt-plus-audio pattern used is not tied to Gemini-specific features beyond the SDK call". *No infrastructure dependency* — "there is no database, message broker, or external state store. The entire stack runs from a single streamlit run command. This reduces deployment complexity for pilot or internal use."
- **Operational benefits (document, Section 4), stated as directly implemented:** hands-free data entry ("audio bytes are sent to Gemini and the returned JSON populates predefined fields without manual transcription"); incremental and correctable input across submissions, "implemented in all three audio applications"; immediate visual feedback via `st.rerun()` "letting the user verify the result before downloading"; exportable structured output from all three applications; and conversation-based collection where "users are not required to provide all information in a single utterance".
- **Deployment/adoption status:** the document positions the work for "pilot or internal use"; no production rollout, user count, claim volume, or go-live date is stated.

## 9. Interview Discussion Points

- Why collapse ASR and extraction into one multimodal Gemini call instead of a Whisper-then-LLM pipeline? Fewer components, native multilanguage handling, one round trip; the trade-off is less observability and no intermediate transcript to audit.
- Why two models (`gemini-3.1-flash-lite-preview` vs `gemini-2.0-flash-lite`)? Explain latency/cost fit for a 13-field one-shot versus a multi-turn loop, and acknowledge preview-model deprecation risk.
- How does the app avoid double-billing on Streamlit re-runs? Explain the `last_audio` dedup gate and Streamlit's execution model.
- How does the incremental merge work, and what if the model returns an empty string for a field the user corrected? Discuss the truthy-overwrite rule and its edge cases.
- Explain the conversational memory design: `memory` is serialized into the system prompt each turn; the model returns `response`, field values, and `completed`. Why not function calling?
- What is wrong with `except: pass` on line 34 of `app.py`, and how would you fix it? (Surface `st.error`, log the raw response, retry with a repair prompt, validate against a schema.)
- How would you evaluate accuracy across languages given no metrics exist? Propose a labelled audio set per language, per-field F1, and latency percentiles.
- How would you productionise this? Gemini calls behind a FastAPI service, persisted sessions, auth, pinned dependencies, tests, structured logging.
- Why was `streamlit-mic-recorder` replaced by `st.audio_input`, and why is the old dependency still listed?
- Security/privacy: customer audio goes to a third-party cloud API — what are the data-residency implications for a Qatar insurer?
- Which of the three UX patterns would you recommend for motor claims, and why?
- Walk through the assistant's UI state handling: the status alert (`st.warning` / `st.success` / `st.info`), the `'idle'` status plus `st.error` on exception, and the `completed` flag read with `data.get('completed', False)` — why a safe default matters when the model omits a key.
- Why is the Field column disabled in `st.data_editor`? Keeping the 13-field schema fixed so `result.json` stays consistent for downstream systems while values remain user-correctable.
- The document's "Backend & API" is just "Streamlit": what does it mean that the Streamlit process is the whole backend, and what changes (concurrency, session isolation, Gemini call timeouts) once more than a handful of users hit it?
- How does the truthy-only merge (implemented in all three apps) interact with the assistant's memory-in-prompt design, where the model already sees which fields are filled?
- Dependency reproducibility: the root `requirements.txt` pins nothing while `4-6-2026/requirements.txt` pins Streamlit 1.57.0 and pandas 3.0.3, and the `google-genai` version is never stated. Which file is authoritative, how would you consolidate them, and what breaks first when Streamlit or the SDK moves (`st.audio_input` and `st.data_editor` are both comparatively new APIs)?
- Handling real customer voice at a Qatari insurer: API-key/secret management, an authentication layer in front of a Streamlit app that currently has none, consent capture before recording, an audio-retention policy, and a data-processing agreement with the model provider — none of which appears in AT/STT/V1. What is the minimum set before a pilot with real claimants?
- The 13 vehicle fields, the three required conversational fields, the prompt text, and the JSON schema are not in the document. How would you version that schema so the prompt, the `st.data_editor` table, and `result.json` cannot drift apart — and what does that imply about the "one string literal" model-agnostic claim?
- Why validate nothing beyond the forced JSON MIME type? Walk through what you would add (schema validation, type/format normalisation, required-field enforcement before the download button enables, a repair retry) and where each belongs in the current single-file design.
- Integration reality check: with no HTTP API and no persistence, the only handoff to downstream claims systems is a manually downloaded `result.json`. What is the smallest change that makes this consumable by a policy-admin or claims system, and why was it deliberately left out of a prototype?
- Documentation and handover: the two "Streamlit UI" sections in AT/STT/V1 are empty headings and the third application is never named. What would a V2 of this document contain, and what does that say about how you document prototypes?
- The document says "no client-side JavaScript is written by the project" — what did switching from `streamlit-mic-recorder` to the built-in `st.audio_input` buy you in supply-chain and maintenance terms, and what did it cost in control over the recording (format, sample rate, chunking, streaming)?
- Latency and UX: the whole pipeline is one blocking Gemini call behind an `st.spinner`. How would you measure and then improve perceived latency (streaming partial results, chunked audio, a smaller model for the conversational turn) with no benchmarks in place today?

## 10. Architecture Explanation Points

- **Components:** the document's Figure 1 ("SpeechToText: End-to-End Architecture") shows four stages — Browser (records audio) → Streamlit Application (reads audio bytes and builds prompt) → LLM (calls Gemini) → JSON (returned as structured data) — described as three components. **Browser** — `st.audio_input` grabs the mic and hands the server WAV bytes, no custom JS. **Streamlit app** — one Python process that owns session state, builds the prompt, calls Gemini, merges JSON, re-renders. **Gemini** — a hosted multimodal model doing speech understanding and field extraction in one shot, returning JSON because the response MIME type is forced.
- **Data flow:** record → dedup check against `last_audio` → prompt with the JSON schema (plus current `memory` in the assistant) → `google-genai` call to a flash-lite model → JSON → truthy-only merge into session state (all three apps) → `st.rerun()` (documented for the extractor and the utility) → editable table, markdown field display, or chat view + `st.json(memory)` → `result.json` download.
- **Key decisions:** one multimodal call instead of ASR + LLM (multilanguage "for free" — analyst inference; the document asserts multilanguage support without explaining the mechanism); forced JSON output; session state as the only state, with incremental merge so users can speak in pieces; model name as one string literal; three UX variants under 110 lines each to compare interaction patterns cheaply.
- **Key decision — no service boundary:** Streamlit is both the frontend and, per the document's "Backend & API" section, the entire backend. UI rendering, session state, prompt construction and the paid model call all live in one re-executing script, which is why the dedup gate and `st.rerun()` are load-bearing rather than incidental. Cheap and correct for a prototype; it means concurrency, per-user session isolation, timeout and retry behaviour, and API-key custody are all properties of a single process (analyst observation — the document describes the arrangement but not its consequences).
- **Key decision — two model variants:** `gemini-3.1-flash-lite-preview` for the one-shot 13-field extraction and `gemini-2.0-flash-lite` for the multi-turn loop. The document records the split but gives no rationale; the flash-lite tier reads as a latency/cost choice (inferred from naming, no benchmark given) and the preview model carries deprecation risk (analyst observation).
- **Trade-offs:** no persistence or audit trail, no API surface, the bare `except: pass`, unpinned root dependencies alongside a separately pinned dated requirements file, no tests or metrics, reliance on a preview model, no validation of the model's JSON beyond the forced MIME type, and no documented authentication, secret handling or audio-retention policy for recordings that leave the network.
- **What to improve next:** wrap the Gemini call in a small service with schema validation and retries, surface parse failures instead of swallowing them, persist sessions and raw audio references for audit, consolidate and pin the two requirements files, add authentication and a retention/consent policy before real claimants use it, document the 13-field schema and the three required conversational fields, give the third application a name and a launch command, build a per-language evaluation set to produce accuracy and latency numbers, and pick one UX pattern on that evidence.


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


# Project 12: Trade License Extractor

## 1. Project Overview

- **Project name:** Trade License Extractor (document no. AT/classification/V1, dated 01-Sep-2026 in the document header; the header contains a typo "20262").
- **Business domain:** Insurance (client: Qatar Insurance Group) — trade-licence document intake for the FINANCE Department and the General Insurance (GI) team (the document names only these two target users; the corporate-onboarding / KYC framing is inferred), using UAE trade-license PDFs issued by Abu Dhabi, Dubai (mainland and JAFZA free zone), Sharjah, Ajman, and Umm Al Quwain authorities.
- **Problem statement:** Trade licences arrive as PDFs of highly varied provenance — born-digital, rasterized print-to-PDF, true flatbed scans, and vector-outline PDFs with zero embedded fonts — often as multi-document bundles (licence + commercial register + chamber certificate). Finance and GI staff otherwise re-key licence number, company name, dates, registration number, and contact details by hand (inferred from the "zero manual field entry" benefit; the document does not describe the prior manual process). Layouts differ per issuing authority (9 known templates), text is bilingual Arabic/English with bidi and mojibake hazards (named as corpus hazards in the experimentation table), and look-alike fields (DCCI/Chamber No vs. Commercial Register No; Fax vs. Mobile — the confusions the vision-LLM path actually produced) make naive extraction unreliable.
- **Project objective:** Build a CPU-only, open-source Python pipeline that reads any UAE trade-license PDF and emits structured, validated field data — with per-field confidence and provenance — in under 10 seconds per document and inside a 20-second operational constraint, without paid APIs, cloud services, or GPU hardware.
- **Expected business outcome:** Zero manual field entry for licence intake, two role-specific views (Finance and GI) delivered as JSON, absent fields returned as null rather than "stolen" adjacent values, and an automatic `review_flags` queue that prioritises low-confidence and LLM-sourced fields for human review. The document splits these into **Operational Benefits** — which it states are "directly traceable to implemented and measured behaviour in the project" (zero manual entry on the corpus, processing inside the 20-second constraint, nulls for absent values, per-field confidence and provenance making every value auditable without re-reading the PDF, and `review_flags` as a prioritised review queue) — and **Strategic Benefits**: a template-agnostic expansion path that "operates without template maps and handles unseen authority layouts" and is positioned as the route to future GPU or served-LLM deployment "where it could meet the same latency budget without code changes", plus an open-source CPU-only stack needing no paid APIs, cloud services or GPU, with the Ollama fallback local and optional.

## 2. Role and Responsibilities

**Core responsibilities (inferred from the system as documented):**

- Designed the end-to-end architecture: PDF input → document classifier (scanned vs. digital via `fitz`) → text extraction (PyMuPDF / OCR) → coordinate-aware line conversion → regex/label-anchor field extraction with LLM fallback → validation and JSON output.
- Implemented deterministic label-anchor extraction for 9 known UAE authority templates (`fields.py` anchor matching) with a template router (`router.py`) that derives `issuing_authority` and `issuing_emirate`.
- Built the selective OCR strategy: PyMuPDF word extraction first, OCR invoked only for pages failing a character-count sufficiency threshold or an Arabic-sanity lexicon check; `ensure_ocr()` detects zero-character (vector-outline) pages.
- Designed the guard-railed local LLM fallback (`llm_fallback.py`) calling Ollama `/api/chat` directly via `httpx` with `format='json'`, restricted to unknown templates and low-confidence fields only.
- Defined the output schema (`extractor/schema.py`, `TradeLicenseRecord`) including `finance_view()` and `gi_view()` role-specific projections, per-field confidence, provenance (page number, extraction method), and `review_flags`. Pydantic >=2.6 is the stack table's declared data-validation library; that `TradeLicenseRecord` is itself a Pydantic model is inferred. The document records that the Finance/GI view pattern from the intermediate `unified_extractor` pipeline was "carried directly into extractor/schema.py TradeLicenseRecord", and that the two views expose deliberate preference rules (`company_name_en` over `company_name_ar`, `phone` over `mobile`, `registration_no` never DCCI/Chamber No).
- Ran the experimentation programme across five approaches (regex/OCR precursor, intermediate hybrid with remote VLM, Ollama JSON-mode script, production POC, RAG extractor) and made the go/no-go decisions recorded in `COMPLETE_SUMMARY.md` and `PROJECT_DESIGN.md`.
- Built the evaluation harness (`eval/run_eval.py`, `values_match()`) and ground-truth corpus of 16 documents, and measured per-document latency against the 20-second budget.
- Exposed the pipeline through a FastAPI backend (`python app.py`); a "Swagger UI" heading exists under section 3 of the document but the endpoint reference beneath it is empty.

**Supporting tasks (inferred):**

- Assembled and annotated the evaluation corpus (Dubai Samples 1, 2, 6 and others across the five emirates) with ground truth, including null-field verification (e.g., Email null where no label is present).
- Wrote Jupyter notebooks (`trade_license.ipynb`, `TradeLicense_POC.ipynb`, `rag_extractor/RAG_POC.ipynb`) as experiment logs and a demonstration entry point (`extractor.extract()`, `extractor.extract_folder()`).
- Built the secondary RAG extractor sub-package (`rag_extract()` orchestrator, `linearise.py`, `chunking.py`, `semantic_retrieval.py` with RRF fusion, `llm.py` with grounding check) as the template-agnostic expansion path.
- Handled Arabic text processing (python-bidi, arabic-reshaper), fuzzy label matching (RapidFuzz), date parsing to ISO strings (dateparser), and OpenCV image pre-processing for OCR.
- Maintained `requirements.txt` with pinned minimum versions and authored the design/summary documentation (`PROJECT_DESIGN.md`) and engineering diary (`COMPLETE_SUMMARY.md`).
- Trialled the earlier `sample_code.py` design (Gemma-4 via Ollama with JSON structured outputs, keyword-list page routing, 250 DPI rendering, phonenumbers normalisation) — the document records it as a standalone script not imported or referenced by any other project file.

**Estimated ownership level:** Senior AI Engineer (inferred).

Rationale: the document reads as a single engineer's engineering diary (`COMPLETE_SUMMARY.md`) and design record (`PROJECT_DESIGN.md`) — single-author status is inferred; the document never names the author or a team — covering the full lifecycle: five trialled approaches with explicit "carried into production / not carried" decisions, a production package (`extractor/`) with router, fields, schema, OCR (`ensure_ocr()`) and LLM-fallback modules, a second sub-package (`rag_extractor/`), an evaluation harness with ground truth, a FastAPI service (endpoint reference left empty), and latency measurements against a 20-second operational constraint. The scope (classification, OCR, bilingual text normalisation, deterministic extraction, LLM guard-rails, RAG, validation, API) and the evidence-driven rejection of the VLM path (DCCI/Register and Fax/Mobile confusion) indicate senior-level independent technical judgement. There is no evidence of managing other engineers or of multi-system enterprise integration, so Technical Lead or Solution Architect is not supported by the document.

## 3. End-to-End Workflow

**Processing stages:**

1. **PDF input.** Any UAE trade-licence PDF: born-digital, rasterized print-to-PDF, flatbed scan, or vector-outline (zero fonts). Multi-document bundles (licence + register + chamber certificate) are accepted; the intermediate pipeline capped pages at `MAX_PAGES=2`, while the production pipeline handles 11- and 14-page raster bundles.
2. **Document classification.** `fitz` (PyMuPDF) "finds the pdf is scanned or digital and routes" (document wording, Architecture Component Descriptions). The document states the classification at PDF level; that the digital/scanned decision is effectively taken per page is inferred from the Key Capabilities wording that OCR is invoked "only for pages that fail a character-count sufficiency threshold".
3. **Text extraction.** PyMuPDF word extraction runs first. Pages that fail a character-count sufficiency threshold, or whose Arabic text fails an Arabic-sanity lexicon check, are sent to OCR (the Key Capabilities section names "Paddle v3 medium"; the experimentation table states PaddleOCR was replaced by Tesseract + RapidOCR, and `ensure_ocr()` applies Tesseract to zero-character pages — see gaps). OpenCV (`opencv-python-headless`) handles image pre-processing.
4. **Line conversion.** Word coordinates are mapped with text to reconstruct line alignment as it appears on the page ("tried to map the line alignment in document"). python-bidi and arabic-reshaper are in the stack for bidi text and Arabic reshaping, and bidi/mojibake are named as corpus hazards the intermediate pipeline did not handle; that this normalisation is applied at the line-conversion stage is inferred (the document does not state where it runs).
5. **Template routing and field extraction.** A template router identifies one of 9 known authority templates and derives `issuing_authority` / `issuing_emirate`; regex and label-anchor matching (`fields.py`; RapidFuzz is the stack's fuzzy-matching library, its use inside `fields.py` is inferred) extract the field set. Negative-label guards — whose design the document attributes to the confirmed VLM failures (DCCI No returned as Register No, Fax as Mobile) — prevent DCCI/Chamber No being taken as `registration_no` and Fax being taken as `mobile`. The regex and line-alignment concepts from the early `trade_license.ipynb` cells are recorded as the foundation of `fields.py` anchor matching.
6. **LLM fallback (optional).** For unknown templates or low-confidence fields only, `llm_fallback.py` calls local Ollama (`qwen2.5:3b-instruct-q4_K_M`) via `httpx` on `/api/chat` with `format='json'` and an explicit output schema. The Architecture Component Descriptions table states the trigger more loosely ("For some layout complication fileds the system fallbacks to this LL method"); the Key Capabilities section is the stricter and more specific statement (unknown templates and low-confidence fields only). The Strategic Benefits section states the fallback "runs locally and is optional", so the deterministic path must stand alone when Ollama is absent (how the system behaves in that case is not documented).
7. **Validation and output.** `TradeLicenseRecord` validates the record ("Validation & JSON — Data checks & return in json output" per Figure 1), stores per-field confidence and provenance (page, method), populates `review_flags`, and returns JSON; `finance_view()` and `gi_view()` project role-specific dictionaries. Issue and expiry dates are ISO `YYYY-MM-DD` strings per the view tables and dateparser >=1.2 is the stack's date-parsing library; that dateparser is what produces the ISO normalisation is inferred. pandas >=2.0 is the stack's "Data frames / export" library and supports batch export.

**Data flow:** PDF bytes → per-page text/word spans with coordinates (PyMuPDF) or OCR output → normalised text lines → field dictionary with confidence/provenance → validated `TradeLicenseRecord` → JSON (full record, Finance view, GI view). Batch runs use `extractor.extract_folder()`; pandas is the stack's data-frame/export library.

**RAG expansion path (`rag_extractor/`):** load → linearise (`linearise.py`) → chunk (`chunking.py`) → semantic / BM25 / hybrid retrieval with RRF fusion (`semantic_retrieval.py`, `paraphrase-multilingual-MiniLM-L12-v2` via fastembed/ONNX) → full extraction with a grounding check (`llm.py`), orchestrated by `rag_extract()` with defaults `k_per_query=3`, `max_chars=2400`. The document positions it as operating without template maps, handling unseen authority layouts, and as the path for future GPU or served-LLM deployment "where it could meet the same latency budget without code changes".

**Integrations and dependencies:** PyMuPDF >=1.24, Pydantic >=2.6, RapidFuzz >=3.5, python-bidi >=0.4.2, arabic-reshaper >=3.0, dateparser >=1.2, PaddleOCR, opencv-python-headless >=4.9, Ollama (host-installed) with `qwen2.5:3b-instruct-q4_K_M`, pandas >=2.0, FastAPI, httpx; RAG sub-package uses `paraphrase-multilingual-MiniLM-L12-v2` via fastembed/ONNX with BM25 and RRF fusion.

**Flow line:**
`Client -> FastAPI (app.py) -> fitz classifier (digital/scanned) -> PyMuPDF words | OCR (threshold / Arabic-sanity gate) -> line alignment + bidi/reshape -> template router (9 templates) -> regex/label-anchor fields -> [Ollama qwen2.5:3b JSON fallback if unknown/low-confidence] -> Pydantic TradeLicenseRecord (confidence, provenance, review_flags) -> finance_view()/gi_view() -> JSON`

The document supports every stage of this line except the first hop: it states only that the backend framework is FastAPI and that the app launches with `python app.py`. That the FastAPI layer is what invokes the extractor, and the shape of the request/response, is inferred — section 3 ("API & Endpoint Reference") and its "Swagger UI" sub-heading are empty.

**Architecture Summary:** The system is a deterministic-first hybrid document extractor. Cheap, exact methods (native PDF text, regex anchors, template routing) handle the known 9 templates, and expensive or fallible methods (OCR, local LLM) are gated behind explicit sufficiency and confidence checks so they run only when needed. Every value carries confidence and provenance, and uncertain or LLM-sourced values are flagged for human review rather than silently trusted. The design is CPU-only and dependency-free of paid services, with a separate RAG extractor positioned as the template-agnostic path for future GPU or served-LLM deployment.

## 4. Technologies and Tools Used

| Category | Technologies (as stated in the document) |
| --- | --- |
| Programming languages | Python |
| Frameworks | FastAPI (backend/API; named in the "Backend & API" section only, no version stated and absent from the stack table); Pydantic >=2.6 (data validation) |
| Libraries | PyMuPDF (fitz) >=1.24; RapidFuzz >=3.5; python-bidi >=0.4.2; arabic-reshaper >=3.0; dateparser >=1.2; opencv-python-headless >=4.9; pandas >=2.0; httpx (Ollama client in `llm_fallback.py`); fastembed/ONNX (RAG sub-package); BM25 retrieval in the RAG sub-package (library not named); phonenumbers (experimental `sample_code.py` only, not in production); an `OpenAI` client object appears in a recorded `TypeError` in `trade_license.ipynb` (experimental only; the client library is otherwise not named) |
| AI/ML models | qwen2.5:3b-instruct-q4_K_M (production LLM fallback via Ollama; exact release version not specified); paraphrase-multilingual-MiniLM-L12-v2 (RAG embeddings); experimental only: Gemma-4 (remote llama.cpp and Ollama `gemma4:e4b`), qwen2.5vl:3b (vision-LLM) |
| OCR tools | PaddleOCR ("Paddle v3 medium" per Key Capabilities; "Paddle ocr" as OCR-primary in the stack table with no version; noted as not in `requirements.txt` for the intermediate pipeline); Tesseract and RapidOCR (named in the experimentation table as replacing PaddleOCR; `ensure_ocr()` applies Tesseract) |
| Databases | Not stated in documentation |
| Cloud platforms | None — explicitly CPU-only, no cloud services or paid APIs |
| APIs | FastAPI endpoints (a "Swagger UI" heading exists; paths and contracts not documented); Ollama `/api/chat` (local, `format='json'`); Ollama at http://localhost:11434 in experimental `sample_code.py`; experimental remote llama.cpp server at http://172.20.132.97:8000 (APITimeoutError recorded) |
| DevOps tools | Not stated in documentation |
| Version control | Not stated in documentation |
| Deployment tools | Not stated in documentation (launched with `python app.py`; Ollama installed on host) |
| Document processing tools | PyMuPDF (fitz) for parsing/classification; OpenCV for OCR image pre-processing; python-bidi and arabic-reshaper for Arabic text; Jupyter notebooks for experimentation |
| Automation tools | Not stated in documentation (batch processing via `extractor.extract_folder()`; evaluation via `eval/run_eval.py`) |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Modular Python package design (`extractor/` with `router.py`, `fields.py`, `schema.py`, `llm_fallback.py`, `ensure_ocr()`; `rag_extractor/` with `__init__.py` defaults, `linearise.py`, `chunking.py`, `semantic_retrieval.py`, `llm.py`); pinned minimum versions in `requirements.txt`; evolving the concepts of a 599-line intermediate script (`unified_extractor/trade_license_extractor.py`) and a 1,272-line experimental notebook into a production package (the document records which concepts were carried and which were not); engineering diary and design documentation.
- **AI Engineering:** Guard-railed LLM fallback restricted to unknown templates and low-confidence fields; JSON-mode structured outputs with explicit schema; local quantised model (q4_K_M) — the rationale of CPU latency is inferred, the document only states the quantisation; evidence-based rejection of the vision-LLM path after field-confusion failures; replacement of a remote VLM dependency with a local text model.
- **Machine Learning:** Model evaluation against ground truth (`values_match()` harness in `eval/run_eval.py`, reused by both pipelines, over 16 documents); Plan B (full-LLM via `OllamaClient.full_extract()`) and Plan E (VLM, `RUN_VLM=False`) comparison cells in `TradeLicense_POC.ipynb` — recorded as demonstration stubs with no stored outputs; multilingual embedding model (`paraphrase-multilingual-MiniLM-L12-v2`) for retrieval.
- **NLP:** Bilingual Arabic/English text handling (python-bidi, arabic-reshaper; mojibake is named as a corpus hazard, the handling mechanism is not described); Arabic-sanity lexicon check gating OCR; fuzzy label matching with RapidFuzz; date normalisation to ISO `YYYY-MM-DD`; hybrid semantic/BM25 retrieval with RRF fusion in the RAG extractor.
- **Computer Vision:** Selective OCR with sufficiency gating; OpenCV pre-processing; DPI tuning for OCR and VLM rendering (OCR_DPI=200, VLM_DPI=120, 250 DPI in experiments); handling of vector-outline (zero-font) PDFs.
- **Data Engineering:** Coordinate-aware line reconstruction from word spans; per-field provenance capture; batch folder processing and pandas export; ground-truth corpus construction.
- **Cloud:** Not demonstrated in this project (deliberately CPU-only, no cloud services).
- **MLOps:** Evaluation harness reused across pipelines; latency measurement per document class against a 20-second budget; notebook gating via `RUN_VLM=False`; local model discovery through Ollama.
- **API Development:** FastAPI backend exposing extraction, launched with `python app.py` (a "Swagger UI" sub-heading exists in section 3 but its content — endpoint paths, request/response contracts, auth — is empty, so the API surface is undemonstrated in the document); library-level public API (`extract()`, `extract_folder()`, `finance_view()`, `gi_view()`) documented in section 4 as the "Library Public API Reference" with a per-key source-field mapping for each role view; direct `httpx` client to Ollama `/api/chat`.
- **Prompt Engineering:** JSON-schema-constrained prompts to Ollama (`format='json'`), grounding check in `rag_extractor/llm.py`, and LLM recovery calls for missing fields (9 recovery calls observed in the RAG run; prompt contents are not shown in the document).
- **System Design:** Deterministic-first hybrid with cost-gated fallbacks; template router with negative-label guards; confidence/provenance/review-flag contract; separation of production extractor from template-agnostic RAG expansion path.

## 6. Detailed Technical Contributions

**Features implemented**
- Support for five emirates and 9 authority templates (Abu Dhabi, Dubai mainland and JAFZA, Sharjah, Ajman, Umm Al Quwain).
- Acceptance of born-digital, rasterized, flatbed-scanned, and vector-outline PDFs and multi-document bundles.
- Two role views: `finance_view()` (License No, Company Name, License Type [Commercial/Professional/Industrial], License Issue Date, License Expiry Date, License Issuing Authority, Issuing Emirate, Nature of Business [activities joined with '; '], optional Address/Email/Telephone) and `gi_view()` (Company Id [Trade Licence No], Company Name [Trade Name], Issue Date, Expiry Date, Registration No, Email Id, Telephone No). Both views are projections of the same underlying fields (`license_number`, `company_name_en`/`company_name_ar`, `license_type`, `issue_date`, `expiry_date`, `issuing_authority`, `issuing_emirate`, `nature_of_business`, `address`, `email`, `phone`/`mobile`, `registration_no`).
- Field preference rules: `company_name_en` preferred over `company_name_ar`; `phone` preferred over `mobile`; `registration_no` must be the Commercial Register number, never DCCI/Chamber No.
- Per-field confidence, provenance (page number, extraction method), and `review_flags` on every record.

**Models used**
- Production: `qwen2.5:3b-instruct-q4_K_M` via Ollama (text-only fallback).
- RAG sub-package: `paraphrase-multilingual-MiniLM-L12-v2` via fastembed/ONNX.
- Experimental, not carried: Gemma-4 via remote llama.cpp (http://172.20.132.97:8000) and via Ollama (`gemma4:e4b`), `qwen2.5vl:3b` vision-LLM.

**Pipelines built**
- Production `extractor/`: classify → extract → line-align → route → anchor-extract → LLM fallback → validate.
- Early `trade_license.ipynb` (root, 90 KB, 1,272 lines): Plans A/B precursor cells (regex + PaddleOCR) and Plan E vision-LLM cells that render PDF pages as images and send them to (a) Gemma-4 on the remote llama.cpp server and (b) `qwen2.5vl:3b` via Ollama; it also contains a cell calling `unified_extractor.extract()` whose stored output shows the Finance and GI views for Dubai Sample 2. Stored regression output on Dubai Sample 1: correct `license_no` (120278), `registration_no` (1014241), `issue_date`, `expiry_date`, `email`, and phone numbers. VLM/LLM cells have no stored output or recorded errors (`APITimeoutError`; `TypeError 'OpenAI object is not iterable'`). Not carried into `extractor/`.
- Intermediate `unified_extractor/trade_license_extractor.py` (599 lines): PyMuPDF text spans + PaddleOCR for scanned pages + remote Gemma-4 VLM for missing fields only; produces Finance and GI view dicts; `MAX_PAGES=2`, `OCR_DPI=200`, `VLM_DPI=120`. Documented shortcomings: PaddleOCR not in `requirements.txt`, dependence on a remote VLM server, and no handling of bidi, mojibake, or multi-page bundles. Carried forward: the Finance/GI view pattern into `extractor/schema.py`, the page-type logic and VLM-as-fallback concept into `router.py` and `llm_fallback.py`; PaddleOCR replaced by Tesseract + RapidOCR; remote VLM replaced by the local Ollama text model.
- `sample_code.py` (root, 291 lines): Gemma-4 via Ollama (http://localhost:11434, `gemma4:e4b`) with JSON structured outputs (Ollama `format` parameter), page routing by keyword list, 250 DPI rendering, phonenumbers normalisation. Not imported or referenced elsewhere; only the JSON-mode/explicit-schema idea was carried into `llm_fallback.py`.
- `TradeLicense_POC.ipynb` (root, 7.9 KB, 235 lines): production demonstration notebook — (1) single-doc JSON with confidence + provenance; (2) Finance/GI views; (3) batch run over all 16 documents; (4) accuracy evaluation against ground truth; (5) Plan B cell (text → full LLM extract via `OllamaClient.full_extract()`); (6) Plan E cell (vision-LLM `qwen2.5vl`, gated by `RUN_VLM=False`). All `execution_count` values are null (never run and saved).
- `rag_extractor/` with `RAG_POC.ipynb` (7.2 KB, 221 lines): load → linearise → chunk → semantic/BM25/hybrid retrieval (RRF) → full extraction with grounding check; defaults `k_per_query=3`, `max_chars=2400`. Stored outputs: `'LLM model: qwen2.5:3b-instruct-q4_K_M'` (model discovery succeeds); stages cell `'pages kept: [1] | aligned lines: 0 / 0 chunks -> 0 selected (0 chars)'` on the hard-coded vector-outline Dubai Sample 6; all three retrieval modes return empty lists; full extraction returns all-null fields after 9 LLM recovery calls in 44,902 ms. The full hybrid eval output is not stored.
- `eval/run_eval.py` with `values_match()` reused across both pipelines.

**APIs integrated**
- FastAPI service launched with `python app.py`; a "Swagger UI" heading exists but endpoint paths and request/response contracts are not documented.
- Ollama `/api/chat` via `httpx` with `format='json'`; Ollama at http://localhost:11434 in `sample_code.py`; remote llama.cpp server at http://172.20.132.97:8000 in experiments (timed out).

**Data extraction methods**
- PyMuPDF word extraction with coordinates; OCR only when the character-count threshold or Arabic-sanity check fails; regex and RapidFuzz label anchors; dateparser to ISO `YYYY-MM-DD`; Arabic bidi/reshaping normalisation.

**Validation logic**
- Pydantic `TradeLicenseRecord` schema; nulls for absent values (Email null verified in ground truth); negative-label guards against DCCI-as-Register and Fax-as-Mobile; LLM output restricted to JSON schema (`format='json'` with an explicit output schema); grounding check in RAG `llm.py`; `review_flags` for low-confidence and LLM-sourced fields; field preference/fallback rules (`company_name_en` → `company_name_ar`, `phone` → `mobile`); `license_type` constrained to Commercial / Professional / Industrial; dates emitted as ISO `YYYY-MM-DD`; `nature_of_business` joined with '; '.
- **Error-handling behaviour actually documented** is thin: absent values return null rather than an adjacent value; low-confidence and LLM-sourced values are flagged for review; `ensure_ocr()` detects zero-character (vector-outline) pages and applies OCR before field extraction. Failure modes are recorded only as experiment outcomes (`APITimeoutError` when the remote llama.cpp server was unreachable; `TypeError 'OpenAI object is not iterable'`). No exception handling, timeout, retry, or degraded-mode behaviour is documented for the production Ollama call, for OCR failure, or for the FastAPI layer.

**Automation workflows**
- `extractor.extract_folder()` batch runs over all 16 corpus documents; accuracy evaluation against ground truth in `TradeLicense_POC.ipynb` section 4.

**Optimization techniques**
- Deterministic-first ordering so OCR and LLM calls run only when the sufficiency/confidence gates fail; q4_K_M quantised local model; DPI settings (`OCR_DPI=200`, `VLM_DPI=120`, 250 DPI) in the experimental pipelines; `MAX_PAGES=2` page-cap in the intermediate pipeline; retrieval limits (`k_per_query=3`, `max_chars=2400`) in RAG. Which of these were tuned versus set once is not stated.

**Performance improvements (documented figures)**
- Born-digital pages: 65–130 ms; single-page OCR: 6–10.7 s; 11- and 14-page raster bundles: 12–13.5 s; all 16 corpus documents within the 20-second budget; intermediate pipeline on Dubai Sample 2: 10.9 s. RAG run on the vector-outline Dubai Sample 6: 44,902 ms with 9 LLM recovery calls and all-null output (edge case, handled in production by `ensure_ocr()`).

## 7. Challenges and Solutions

1. **Heterogeneous PDF provenance (technical).** Born-digital, rasterized, scanned, and zero-font vector-outline PDFs. Solved by a fitz-based classifier plus a character-count sufficiency threshold that triggers OCR only when native text is missing; `ensure_ocr()` catches zero-character pages. Alternative considered: OCR every page (rejected implicitly by the latency figures — a single-page OCR document costs 6–10.7 s vs. 65–130 ms for a born-digital page) (analyst suggestion).
2. **Arabic/English bidi and mojibake (technical).** Solved with python-bidi, arabic-reshaper, and an Arabic-sanity lexicon check that routes garbled Arabic to OCR. Alternative: rely on an LLM to "read through" corruption (analyst suggestion) — rejected in favour of deterministic normalisation given the CPU budget.
3. **Look-alike field confusion (technical/business).** The VLM path returned DCCI No as Register No and Fax as Mobile (documented in PROJECT_DESIGN.md §1.2.8). Solved with negative-label guards in the anchor extractor and an explicit rule that `registration_no` is never the Chamber number. Alternative considered and rejected: vision-LLM extraction (Plan E).
4. **Unknown authority layouts (business).** 9 templates cover the corpus, but new authorities will appear. Solved by the guard-railed Ollama fallback and the template-agnostic RAG extractor as the expansion path. Alternative: hand-author a template per new authority (analyst suggestion).
5. **Latency on CPU (technical).** Remote VLM server timeouts (APITimeoutError at 172.20.132.97:8000) and 45-second RAG recovery runs. Solved by a local quantised 3B text model called only for low-confidence fields, keeping all 16 documents under 20 seconds. Alternative: GPU or served LLM, explicitly deferred as a future path.
6. **Auditability (business).** Finance/GI reviewers need to trust values without re-opening PDFs. Solved with per-field confidence, provenance (page, method), and `review_flags`.
7. **Documented limitations and inconsistencies.** The OCR engine is named inconsistently (Paddle v3 medium in Key Capabilities and "Paddle ocr" in the stack table vs. Tesseract + RapidOCR in the experimentation table and Tesseract in `ensure_ocr()`); PaddleOCR is also noted as absent from `requirements.txt`. The corpus size is stated as both 15 and 16 documents. `TradeLicense_POC.ipynb` has no stored outputs, so the accuracy evaluation result is not recorded. The stored output demonstrating the VLM's DCCI/Fax confusion is not in `trade_license.ipynb`'s saved cells (the document cites `PROJECT_DESIGN.md` §1.2.8 and `COMPLETE_SUMMARY.md` §2 instead). The RAG_POC run against Dubai Sample 6 produced all-null fields after 9 costly LLM calls because OCR fallback was not triggered in that run, and the full hybrid eval output is not stored. Swagger/endpoint paths are missing from section 3. The notebook count is described as "three Jupyter notebooks ... and one notebook in the rag_extractor sub-package" while the table lists three notebooks in total. The header date contains a typo. Further documented gaps: the stack table gives no version for "Paddle ocr" ("-") and records Ollama's version as "Not specified in the project (installed on host)" and the qwen2.5 release version as unspecified; section 3 ("API & Endpoint Reference" / "Swagger UI") is entirely empty, so no request/response contract, authentication, authorisation, rate-limiting, logging, PII-handling or data-retention behaviour is documented for a service that ingests company identity documents (this is an absence in the documentation, not a claim that the system lacks them); no error-handling, timeout or retry behaviour is documented for the production Ollama call or for OCR failure; the RAG path has no documented OCR trigger, which is exactly why RAG_POC returned all nulls on the vector-outline Dubai Sample 6 while the production path handles it via `ensure_ocr()`; and the Strategic Benefits claim that the RAG path "could meet the same latency budget without code changes" on GPU or a served LLM is a forward projection with no measurement behind it — the only recorded RAG run took 44,902 ms on CPU and produced nothing.

## 8. Impact Analysis

- **Business impact:** Zero manual field entry is claimed for the 15 documents in the evaluation corpus (the document frames its operational benefits as "directly traceable to implemented and measured behaviour"); Finance and GI receive ready-to-consume role views in JSON. Strategically, the template-agnostic RAG path is positioned to handle unseen authority layouts and to move to GPU or served-LLM deployment "without code changes". No cost-saving or accuracy percentage is stated.
- **Productivity improvements:** Automated extraction replaces manual re-keying; `review_flags` give reviewers a prioritised queue instead of full re-checks.
- **Accuracy improvements:** An accuracy evaluation against ground truth exists (`values_match()`), but no accuracy figure is recorded in the document. Qualitative evidence: correct extraction of license_no 120278 and registration_no 1014241 on Dubai Sample 1 and registration_no 2490012 on Dubai Sample 2; absent fields return null rather than adjacent values.
- **Cost savings:** Qualitative only — no paid APIs, cloud services, or GPU required; the LLM fallback is optional and local.
- **Time savings:** Documented latency: 65–130 ms (born-digital), 6–10.7 s (single-page OCR), 12–13.5 s (11- and 14-page raster bundles); all 16 corpus documents within the 20-second operational budget; headline claim of under 10 seconds per document. The headline conflicts with the document's own worst case (12–13.5 s on the 11- and 14-page bundles); the 20-second operational constraint is the figure the measurements actually support, and the "under 10 seconds" line holds only for single-document, non-bundle inputs (inferred). No baseline manual handling time is recorded, so the saving per document cannot be quantified from the document.
- **User benefits:** Every value is auditable via page and method provenance; role views match Finance and GI vocabulary; `registration_no` semantics are enforced for GI.

## 9. Interview Discussion Points

- Why deterministic-first rather than LLM-first? Cite the 65–130 ms native path, the 20-second budget, and the VLM's DCCI/Fax confusion.
- How does the OCR gate work — character-count sufficiency threshold plus Arabic-sanity lexicon — and why is it cheaper than OCR-everything?
- Explain the negative-label guard: how you prevent `registration_no` from capturing the Chamber/DCCI number and `mobile` from capturing Fax.
- How is the LLM fallback guard-railed? JSON schema via Ollama `format='json'`, only for unknown templates or low-confidence fields, flagged in `review_flags`.
- Why `qwen2.5:3b-instruct-q4_K_M` on CPU rather than a served model or Gemma-4? Discuss the remote llama.cpp timeouts and quantisation trade-offs.
- Walk through the experimentation history (Plans A/B/E, intermediate hybrid, RAG) and what each contributed to production.
- What does provenance look like on a record and how do Finance/GI use it?
- How did you handle bidi Arabic, reshaping, and mojibake in line reconstruction?
- The RAG_POC produced all nulls on Dubai Sample 6 in 45 seconds — what went wrong and how does production avoid it (`ensure_ocr()`)?
- What accuracy did the eval harness report? Be candid that the POC notebook has no stored outputs and describe how `values_match()` works.
- Clarify the OCR engine story (PaddleOCR vs. Tesseract + RapidOCR) and why the documentation is inconsistent; also why PaddleOCR was used by the intermediate pipeline without being in `requirements.txt`.
- How would you add a tenth authority template versus relying on the RAG path?
- How does the RAG path work — linearise, chunk, semantic vs. BM25 vs. hybrid retrieval with RRF fusion, `k_per_query=3` / `max_chars=2400`, and the grounding check in `llm.py` — and why is it the right expansion path for GPU or served-LLM deployment "without code changes"?
- Why did the intermediate `unified_extractor` pipeline (10.9 s on Dubai Sample 2, correct `registration_no` 2490012) still get superseded? Cite its documented shortcomings: PaddleOCR outside `requirements.txt`, remote VLM dependency, no bidi/mojibake/multi-page-bundle handling.
- What did each abandoned experiment contribute? `sample_code.py` gave the JSON-mode/explicit-schema idea now in `llm_fallback.py`; the notebook's regex/line-alignment cells became `fields.py`; the VLM failures produced the negative-label guards.
- Plan B (`OllamaClient.full_extract()`) and Plan E cells exist as stubs with no stored outputs — be candid that the full-LLM comparison was set up but its results are not recorded.
- Why the output contract is one validated record with two projections (`finance_view()`, `gi_view()`) rather than two extractors: one extraction, one audit trail, two vocabularies (GI's "Company Id (Trade Licence No)" vs. Finance's "License No"), plus the preference rules baked into the views — English company name over Arabic, `phone` over `mobile`, and `registration_no` strictly the Commercial Register number.
- How the per-field confidence score is computed and what threshold flips a field into the LLM fallback — be candid that the document specifies neither, nor the character-count sufficiency threshold, nor the contents of the Arabic-sanity lexicon.
- The fallback is documented as *optional* ("runs locally and is optional"): what the system returns when Ollama is not installed — deterministic-only output with more review flags is the expected behaviour (inferred; the degraded path is not documented).
- What is still missing before this is production-facing: no documented endpoints or request/response contract, no auth, no logging, no timeouts/retries, no containerisation, CI/CD or version control in the document — and trade licences carry company contact PII, so retention and access control are fair questions to raise yourself.
- Reconciling the two corpus counts (15 in the "zero manual field entry" benefit, 16 in the latency claim and batch run) and stating precisely what the zero-manual-entry claim covers.
- Why the headline "under 10 seconds per document" does not match the documented 12–13.5 s worst case, and how you would restate the SLA around the 20-second operational constraint.
- Why `MAX_PAGES=2` in the intermediate pipeline was dropped in production — multi-document bundles (licence + register + chamber certificate) mean the licence page is not always in the first two pages (inferred rationale; the document records the cap and the production 11-/14-page handling but not the reasoning).

## 10. Architecture Explanation Points

Start with the contract: a PDF in, a validated `TradeLicenseRecord` out with confidence, provenance, and review flags, projected into Finance and GI views. Draw five boxes left to right: classifier (fitz decides digital vs. scanned per page), text extraction (PyMuPDF words first; OCR only if the character-count or Arabic-sanity gate fails), line reconstruction (coordinates → aligned lines, bidi/reshape normalisation), template router and anchor extraction (9 templates, regex + RapidFuzz, negative-label guards), and validation (Pydantic, ISO dates, nulls for absent values). Add a dashed side box for the Ollama `qwen2.5:3b` JSON fallback and explain it only fires for unknown templates or low-confidence fields and always sets a review flag. Key design decisions: deterministic-first to hit 65–130 ms on native PDFs and stay under 20 s on 14-page scans; CPU-only, no paid services, with the local Ollama fallback optional; explicit guards derived from observed VLM failures; a local quantised text model replacing the remote VLM after the llama.cpp server timed out; a separate template-agnostic RAG path (no template maps) kept as the expansion route. Trade-offs (documented evidence): OCR dominates latency (6–10.7 s per single-page OCR document vs. 65–130 ms native); LLM recovery calls are expensive on CPU (9 calls cost ~45 s in the RAG run and produced nothing when the page had no text); the intermediate pipeline's remote VLM was the fastest route to full field coverage but was rejected for its dependency and hazard-handling gaps. Trade-offs (analyst view): the 9-template deterministic path presumably needs maintenance as authorities change layouts, which is why the RAG path exists. Name the output contract as a design decision in its own right: one validated `TradeLicenseRecord` with two projections rather than two extraction paths — one extraction, one audit trail, and each team keeps its own vocabulary (`Company Id (Trade Licence No)` for GI, `License No` for Finance) and preference rules (English name over Arabic, `phone` over `mobile`, `registration_no` never the DCCI/Chamber number). Two further documented trade-offs worth stating aloud: the strategic "same latency budget without code changes" claim for a GPU or served-LLM RAG deployment is a projection, not a measured result; and the fallback LLM is declared optional, so the deterministic path has to stand alone when Ollama is absent (the degraded behaviour is not documented). Next improvements (analyst suggestion): pin down and document the single OCR engine, record the ground-truth accuracy from the eval harness, document the FastAPI endpoints, add OCR triggering to the RAG path, and evaluate a served LLM for the template-agnostic route once GPU is available (the document itself names GPU/served-LLM deployment as the future path). Also (analyst suggestion): publish the concrete gate values — character-count sufficiency threshold, the confidence threshold that triggers the fallback, and the Arabic-sanity lexicon — since the whole cost model rests on them; document authentication, logging, timeouts/retries and data retention for the FastAPI service given the company PII in trade licences; reconcile the 15-vs-16 corpus count and the "under 10 seconds" headline against the 12–13.5 s worst case; and pin PaddleOCR/Ollama versions, which the stack table currently leaves as "-" and "Not specified in the project".


# Project 13: UAE OCR AI (FastAPI / Azure Document Intelligence)

## 1. Project Overview

- **Project name:** UAE OCR AI, exposed as the "UAE OCR API" microservice (document no. AT/uae-ocr-api/V2, app version `2.0.0`, dated 01-Sep-2026; FastAPI app description string `'UAE OCR powered by Azure Document Intelligence Service'`). Client: Qatar Insurance Group (QIC / Anoud).
- **Business domain:** Insurance — UAE motor claims (intimation, registration, liability determination, subrogation recovery, underwriting).
- **Target users (as listed in the document):** (1) internal claims and policy administration systems that submit scans and consume the JSON to automate motor claim registration, intimation, and underwriting; (2) motor claims operations teams within QIC / Anoud who need liable-party, affected-party, driver-license, and vehicle-registration data to determine liability and subrogation recovery; (3) executive and technical management evaluating extraction efficiency, service reliability, and auditability; (4) external customer-facing web or mobile intimation portals uploading documents on behalf of policyholders.
- **Problem statement:** UAE motor claims arrive with police accident reports from three jurisdictions (Sharjah/Rafid, Abu Dhabi/Saaed, Dubai Police) — each with its own layout, some with legacy/modern variants or editable/scanned forms — plus six card layouts (UAE Driving License, Emirates Residence ID, Vehicle Registration "Mulkiya", front and back). Content is bilingual Arabic/English, dates are inconsistent, insurer names are free text, and uploads may be blank, corrupted, upside down, or the wrong document type. Claims handlers transcribed accident numbers, dates, vehicle specs, damage descriptions, and driver identities by hand, and bad uploads risked propagating corrupt records into policy and claims administration platforms.
- **Project objective:** Build an internal AI backend that accepts a PDF or image, verifies it is the expected document class, extracts fields via a cost-optimized hybrid of local text-layer parsing and custom-trained Azure Document Intelligence models, normalizes dates to `DD/MM/YYYY`, translates Arabic to English while preserving reshaped Arabic, resolves insurer names to canonical internal codes, and returns one standardized JSON envelope — or a structured error envelope when the input is unusable.
- **Expected business outcome (the document's own "Business Benefits" section):** *Operational* — elimination of manual data transcription (accident numbers, dates, coordinates, vehicle specs, damage descriptions, driver identities); zero-cloud-cost local extraction bypass for digital Abu Dhabi and Dubai reports; intelligent layout auto-classification removing manual pre-sorting of Sharjah documents; deterministic input quality enforcement protecting downstream core insurance databases; automated subrogation entity resolution; seamless bilingual reconciliation for legal audits and regulatory compliance. *Strategic* — a standardized multi-emirate claims pipeline (one unified JSON schema across three police jurisdictions); a decoupled, maintainable microservice architecture in which changes to one emirate's reporting template or Azure model "do not risk introducing regressions in other document classes"; turnkey containerized cloud and on-premise portability; regulatory compliance through strict `DD/MM/YYYY` standardization and right-to-left word reconstruction that "prevent transcription errors in policy enforcement and statutory reporting". No quantified accuracy, throughput, or cost figures are given.

## 2. Role and Responsibilities

**Core responsibilities (inferred from scope and build):**

- Designed the star-shaped architecture: `main.py` entry point, `utils.py` shared core, emirate handlers (`sharjah.py`, `abudhabi.py`, `dubai.py`) and `id_card_parser.py`, each importing only from `utils.py`.
- Built the FastAPI service: `GET /health` plus nine `POST` extraction endpoints, multipart `UploadFile` handling and integrity validation, and two global exception handlers (`OCRError` -> 400/404/422/502 envelopes; generic `Exception` -> HTTP 500 with `logger.critical` stack traces).
- Integrated ten custom Azure Document Intelligence models (four police-report, six card, all `anoud_ocr_*`); training/labeling is implied by "custom-trained" but not described (inferred).
- Engineered the hybrid extraction engine: `is_scanned_pdf` detection, PyMuPDF local text/table parsing for digital PDFs, Azure delegation for raster scans.
- Implemented pre-cloud layout classification: Tesseract (`--psm 6`) on the top third of page 1 at 150 DPI, keyword voting between `anoud_ocr_uae_PR_sharjahnew` and `anoud_ocr_uae_PR_sharjahold`.
- Implemented deterministic quality gates: anchor-keyword verification (>= 50% hit rate) for cards and the `> 3 non-empty fields` gate (`ensure_report_parsed`) for police reports, both HTTP 422.
- Built dual-language insurer resolution with RapidFuzz token-sort / token-set (threshold 85) against 42 `insurers.json` records.
- Integrated Azure Translator; handled Arabic reshaping and bidi display with `arabic-reshaper` and `python-bidi`.
- Containerized the service (Python 3.11-slim, bundled Tesseract, OpenCV/PyMuPDF runtime deps, non-root uvicorn) for on-premise, Azure Container Apps, or Kubernetes.

**Supporting tasks:**

- Defined the response envelope (`message`, `messageType: "S"`, `response`) and per-document field schemas.
- Wrote numeric-identifier extractors: 13+ digit Saaed incident numbers via regex (stated); Dubai 5-6 digit security code and 8+ digit report number (extraction method not stated; regex inferred).
- Parallelized Dubai vector-PDF page and party extraction with `ThreadPoolExecutor(max_workers=6)`.
- Implemented right-to-left polygon sorting to rebuild Arabic `makeModelColorYear` strings.
- License-back post-processing: `SequenceMatcher` (threshold 0.6) normalization of `transmission_type` and permitted-vehicle mapping to English/Arabic lists.
- Date normalization via `python-dateutil`; `RotatingFileHandler` on `logs/app.log` (10,485,760 bytes / 10 MB, 5 backup files); three-path Tesseract discovery (`TESSERACT_PATH` env var, `shutil.which('tesseract')`, AppData local install path); `.env` configuration (its contents — presumably Azure endpoints and keys — are never specified in the document); `/health` probe returning status, service name, and version string.
- Exposed the framework-default documentation surface: `GET /docs` (Swagger UI), `GET /redoc` (ReDoc), `GET /openapi.json` (OpenAPI 3.1 schema); tagged `/health` under `monitoring` and the nine extraction endpoints under `UAE`.
- Authored the technical documentation (endpoint reference, verified constants/thresholds table, business benefits); the document states its content was derived "directly from project source files, configuration manifests, and architectural documentation" (inferred that the candidate authored it).

**Estimated ownership level: Senior AI Engineer (inferred).**
Rationale: the document describes end-to-end build of an "enterprise document intelligence microservice" (its production status and traffic are not stated) — ten endpoints, ten custom cloud OCR models, three police-report pipelines with editable and scanned branches, a local OCR classifier, fuzzy entity resolution, machine translation, structured error contracts, log rotation, and a hardened Docker image targeting Kubernetes/Azure Container Apps. Design decisions (star-shaped dependencies, sync endpoints on the AnyIO pool, cost-avoiding local bypass) are deliberate and justified. No evidence of managing engineers or of architecture beyond this service, so Solution Architect / Technical Lead is not supported; the breadth of AI, backend, and deployment work exceeds Developer or ML Engineer. Note that the document describes itself as derived "directly from project source files, configuration manifests, and architectural documentation" and refers to "verified" constants and launch commands, so it reads as a code-derived write-up; it never names the author or the author's title.

## 3. End-to-End Workflow

**Processing stages:**

1. **Upload and validation.** Client POSTs `multipart/form-data` with a required binary `file` field (FastAPI `UploadFile`). Police-report endpoints are documented as accepting PDF document bytes; card endpoints accept image files (JPEG, PNG). FastAPI performs stream reading and validates file integrity before handing the payload to the domain dispatcher. Empty files or invalid PDF streams raise HTTP 400.
2. **Dispatch.** Police reports route to the emirate handler; cards go to `id_card_parser.py`, which selects the Azure model and runs type validation.
3. **Layout / scan detection.** Sharjah: `page_to_image` rasterizes the top third of page 1 at 150 DPI, Tesseract `--psm 6` counts layout keywords, a vote picks new or old model (ties -> new), and the whole document is then submitted to Azure — the local text-layer bypass is documented only for Abu Dhabi and Dubai reports. Abu Dhabi and Dubai: `is_scanned_pdf(doc)` branches between local and cloud. Dubai scanned: Tesseract on 150 DPI page crops keeps only pages scoring `> 2` of 7 keywords before any page is submitted to Azure (the constants table states `> 2 keywords`, the endpoint text `>= 3 keywords` — equivalent for integer counts, so the two phrasings agree).
4. **Extraction.** Editable: PyMuPDF parses JSON text blocks and table cells locally (Saaed `_parse_editable`: `FAULTY PARTY`, `NON FAULTY PARTY`, `PROPERTY DAMAGES` sliced into indexed coordinate dictionaries, Arabic descriptions shaped, claims translated; Dubai: text spans extracted across all pages, incident-details table plus affected/liable party tables in parallel threads, trailing image-coordinate artifacts popped). Scanned: bytes go to the matching custom Azure model (Saaed `_parse_scanned` parses the returned parties array into driver, insurance, and vehicle profiles; Dubai submits only the pages that passed the keyword filter).
5. **Type verification (cards).** Raw OCR lines are checked against per-card anchor phrases; below 50% the request is rejected with HTTP 422 (e.g. `'Provided image is not a license front'`).
6. **Quality gate (police).** `ensure_report_parsed` rejects `<= 3` non-empty fields with HTTP 422 ("document quality is low"). The document states this gate generically for police reports in its Key Capabilities and constants table, and lists it explicitly for the Saaed `_parse_scanned` branch and the Dubai endpoint; the Rafid endpoint's documented error list names only 400 / 422 (Tesseract missing) / 502, so whether the field-count gate also runs for Sharjah is not stated.
7. **Normalization.** Party fields grouped by prefix (`liable/affected` for Sharjah new, `faulty_/nonfaulty_` for old); dates to `DD/MM/YYYY`; Arabic reshaped for display while unshaped forms are kept for translation and lookup; Azure word polygons sorted right-to-left.
8. **Translation and entity resolution.** Azure Translator produces `*_english` fields; insurer names are fuzzy-matched against `insurers.json` to emit `insurance_company_code` (police documents only, per the architecture diagram).
9. **Response.** `{ "message": "success", "messageType": "S", "response": {...} }`, or an `OCRError` envelope (400/404/422/502); unhandled faults return 500.

**Data flow:** PDF/JPEG/PNG bytes -> PyMuPDF document (handled as a byte stream; the document never mentions writing files to disk, so "in-memory" is inferred) / 150 DPI NumPy raster via `page_to_image` -> Tesseract text (classification and page filtering only) -> either PyMuPDF JSON text blocks and coordinate-indexed table cells (local) or Azure field/word-polygon results (cloud) -> Python dicts -> Azure Translator for Arabic strings -> RapidFuzz lookup against `insurers.json` -> JSON envelope.

**Documented architecture diagram (Figure 1, "End-to-End Architecture"), five stages:** FastAPI service (validation and routing) -> routing endpoints (validation and routing) -> Azure Document Intelligence (extracts fields) -> Normaliser (insurer code, **only police docs**) -> JSON conversion.

**Integrations and dependencies:** Azure Document Intelligence (ten custom models) and Azure Translator via the `azure-ai-*` SDKs; local Tesseract via pytesseract; PyMuPDF, NumPy, RapidFuzz, python-dateutil, arabic-reshaper, python-bidi; `insurers.json` (42 records); `.env` configuration. Pinned versions are listed in Section 4.

```
Client -> FastAPI (validate, dispatch)
       -> [text layer? yes -> PyMuPDF local parse | no -> Tesseract classify/filter -> Azure DI custom model]
       -> anchor / field-count gate -> date normalize + Arabic reshape/bidi
       -> Azure Translator -> RapidFuzz insurer resolution -> JSON envelope
```

**Architecture Summary:** A flat, star-shaped FastAPI microservice: `main.py` declares the ten endpoints and two global exception handlers; every domain module depends only on `utils.py`. Endpoints are plain `def` functions, so FastAPI runs them on the AnyIO worker thread pool and blocking Azure calls, PyMuPDF rendering, and Tesseract stay off the event loop. Extraction is hybrid: digital PDFs are parsed locally at zero cloud cost; raster scans go to one of ten custom Azure models chosen statically (cards) or by a local Tesseract classifier (Sharjah old/new, Dubai page filtering). Deterministic post-processing — anchor verification, field-count gating, date normalization, reshaping, translation, fuzzy insurer resolution — yields one JSON schema across three police jurisdictions and six card types. Delivery is a hardened Docker image (Python 3.11-slim, bundled Tesseract, non-root uvicorn on port 8000).

## 4. Technologies and Tools Used

| Category | Technologies (pinned versions where given) |
|---|---|
| Programming languages | Python (Docker base image Python 3.11-slim) |
| Frameworks | FastAPI `>=0.110.0`; Uvicorn `>=0.28.0` (ASGI); AnyIO worker thread pool (via FastAPI) |
| Libraries | PyMuPDF (fitz) `>=1.24.0`; NumPy `>=1.26.0`; pytesseract `>=0.3.10`; RapidFuzz `>=3.6.0`; azure-ai-documentintelligence `>=1.0.0b1`; azure-ai-translation-text `>=1.0.0b1`; azure-core `>=1.30.0`; python-dateutil `>=2.9.0`; arabic-reshaper `>=3.0.0`; python-bidi `>=0.4.2`; stdlib `difflib.SequenceMatcher`, `concurrent.futures.ThreadPoolExecutor`, `logging.handlers.RotatingFileHandler`, `re`, `shutil`; OpenCV listed as a Docker runtime dependency |
| AI/ML models | Ten custom Azure Document Intelligence models: `anoud_ocr_uae_PR_sharjahnew`, `anoud_ocr_uae_PR_sharjahold`, `anoud_ocr_uae_PR_abudhabi`, `anoud_ocr_uae_PR_dubai`, `anoud_ocr_custom_uae_df`, `anoud_ocr_custom_uaedb`, `anoud_ocr_custom_uaerf`, `anoud_ocr_custom_uaerb`, `anoud_ocr_custom_uae_vf`, `anoud_ocr_custom_uae_vb`; Azure Cognitive Services Translator |
| OCR tools | Azure Document Intelligence (cloud, custom models); Tesseract OCR via pytesseract (local classification, `--psm 6`, 150 DPI) |
| Databases | Not stated in documentation (master data is a flat file, `insurers.json`, 42 records) |
| Cloud platforms | Microsoft Azure (Document Intelligence, Translator); Azure Container Apps and Kubernetes named as deployment targets |
| APIs | Exposed: REST with OpenAPI 3.1 (`/docs` Swagger UI, `/redoc`, `/openapi.json`); `/health` under the `monitoring` tag, the nine POST endpoints under the `UAE` tag. Consumed: Azure Document Intelligence API, Azure Translator Text API |
| DevOps tools | Docker (hardened Dockerfile, non-root uvicorn, `--env-file .env`); RotatingFileHandler logging (`logs/app.log`, 10 MB, 5 backups); `/health` probe |
| Version control | Not stated in documentation |
| Deployment tools | Docker (`docker build -t uae-ocr .`, `docker run -p 8000:8000 --env-file .env uae-ocr`); Uvicorn (`uvicorn main:app --host 0.0.0.0 --port 8000` / `python main.py`); targets: on-premise, Azure Container Apps, Kubernetes |
| Document processing tools | PyMuPDF (text-layer/table extraction, `page_to_image` rasterization); arabic-reshaper and python-bidi; python-dateutil |
| Automation tools | Not stated in documentation (no CI/CD or workflow tooling described) |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Star-shaped modular design with single-responsibility modules importing exclusively from `utils.py`; custom `OCRError` mapped to dynamic HTTP status codes; global exception handlers with `logger.critical` stack traces; regex extractors for numeric identifiers; rotating file logging (10 MB, 5 backups); environment-based configuration; three-path native-binary discovery for cross-platform portability.
- **AI Engineering:** Integrating ten custom Azure Document Intelligence models; hybrid local/cloud routing that calls cloud AI only when a text layer is absent; using a cheap local OCR pass to select the right neural model before paying for the cloud call.
- **Machine Learning:** Custom document-model training on Azure Document Intelligence is implied by "custom-trained" (inferred); no training procedure, evaluation, or metrics are described. Approximate string matching with tuned thresholds (RapidFuzz 85, `SequenceMatcher` 0.6).
- **NLP:** Arabic/English fuzzy entity resolution (token-sort/token-set); Arabic reshaping and bidi; neural MT integration; right-to-left word reconstruction from OCR polygons; keyword-vote document classification.
- **Computer Vision:** PDF rasterization to NumPy arrays at 150 DPI; header-region cropping for OCR; geometric sorting of word polygons; coordinate-based table slicing. No CNN/YOLO training here.
- **Data Engineering:** Normalizing heterogeneous report formats into one JSON schema; date standardization; party-prefix grouping; flat-file master-data lookup; stripping image-coordinate artifacts.
- **Cloud:** Azure Document Intelligence and Azure Translator SDK integration (`azure-ai-documentintelligence`, `azure-ai-translation-text`, `azure-core`); container packaging for on-premise / Azure Container Apps / Kubernetes; `.env`-file configuration passed via `--env-file` (the document does not say what the file contains; Azure endpoint and key are inferred).
- **MLOps:** Model-per-document-type routing by model ID; deterministic guardrails around model output; health endpoint and rotating logs. No monitoring, retraining, or experiment tracking described.
- **API Development:** FastAPI multipart endpoints, OpenAPI 3.1 docs, consistent success/error envelopes, blocking-work isolation on the AnyIO pool, HTTP semantics 400/404/422/502/500.
- **Prompt Engineering:** Not demonstrated in this project (no LLM prompting; extraction uses trained document models and deterministic logic).
- **System Design:** Cost-aware routing, pre-cloud classification, fail-fast validation protecting downstream systems, concurrency choices, containerized portability.

## 6. Detailed Technical Contributions

**Features implemented**

- `GET /health` -> `{"status": "healthy", "service": "AI OCR API", "version": "2.0.0"}`.
- `POST /police_report/rafid`, `/police_report/saaed`, `/police_report/dubai` — police-report extraction. The one full sample envelope given (Rafid) returns `nature_of_loss, reason_of_accident, report_number, report_date, incident_time, road_condition, is_unknown_damage, incident_number, street_name, incident_date, weather_condition, is_legal_claim, accident_location, region, emirate, incident_type, vehicle` (vehicle count, e.g. `"2"`), `primary_reason, visiblity_condition, damage_description`, plus `vehicle_party[]` with `driver_detail` (`driver_name`, `driver_name_english`, `license_expiry_date`, `license_source`, `license_no`), `insurance_detail` (`insurance_type`, `insurance_policy_no`, `insurance_company`, `insurance_company_english`, `insurance_company_code`, `insurance_validity`), `vehicle_detail` (`owner_name`, `plate_no`, `vehicle_model`, `vehicle_make_and_model`, `is_blamed_party`, `chasiss_no`), and `injured_party[]`, `property_damage_party[]`, `claim_description{}`. Saaed is described in prose only, as "Abu Dhabi claim, property damage, and party structures". Dubai is described in prose as containing primary/secondary causes, visibility, `incident_number`, `security_code`, `vehicle_party` profiles, and `important_notes`. (`visiblity_condition` and `chasiss_no` spellings are as documented.)
- `POST /uae/license_front` -> `date_of_birth, expiry_date, issue_date, license_number, name, nationality, place_of_issue, traffic_code_number`.
- `POST /uae/license_back` -> `traffic_code_number, transmission_type` (`automatic gear` / `manual gear`), `permitted_vehicle.{english, arabic}` (documented sample: `english: ["light vehicle", "motorcycle"]` with the parallel Arabic list; category mapping example `'light vehicle'` -> `مركبة خفيفة`).
- `POST /uae/residence_front` -> `id_number, name, nationality, issue_date, expiry_date`.
- `POST /uae/residence_back` -> `card_number, occupation, employer, issuing_place, date_of_birth, sex`.
- `POST /uae/vehicle_front` -> `traffic_plate_no, place_of_issue, policy_no, insurance_company, insurance_expiry, mortgage_by, owner_name`.
- `POST /uae/vehicle_back` -> `chassis_number, engine_number, model_year, vehicle_make, vehicle_model, vehicle_type, empty_weight, gross_vehicle_weight (g_v_w), number_of_passengers`.

**Models used:** police — `anoud_ocr_uae_PR_sharjahnew`, `_sharjahold`, `_abudhabi`, `_dubai`; cards — `anoud_ocr_custom_uae_df` (license front), `_uaedb` (license back), `_uaerf` (residence front), `_uaerb` (residence back), `_uae_vf` (vehicle front), `_uae_vb` (vehicle back); Azure Translator for Arabic -> English; Tesseract for local classification only.

**Pipelines built:** the Sharjah classify-then-Azure pipeline; the Saaed `_parse_editable` / `_parse_scanned` split; the Dubai parallel-editable / filtered-scanned split (scanned branch reconstructs Arabic `makeModelColorYear` by sorting word polygons right-to-left); and the card select-model -> Azure -> anchor-verify -> normalize pipeline (see Section 3).

**APIs integrated:** Azure Document Intelligence custom-model analysis; Azure Translator Text; Tesseract via pytesseract. The service publishes OpenAPI 3.1 with Swagger UI and ReDoc.

**Data extraction methods:** native PDF text-layer and table extraction (PyMuPDF); cloud layout-model field extraction; regex for numeric identifiers; coordinate-indexed table slicing; polygon-order reconstruction; keyword scoring for classification.

**Validation logic**

- HTTP 400: empty upload or invalid PDF stream. HTTP 404: unknown card type.
- HTTP 422: anchor hit rate below `ANCHOR_MATCH_THRESHOLD = 0.5`; police report with `<= 3` non-empty fields ("document quality is low"); Tesseract binary missing (documented for the Rafid and Dubai endpoints).
- HTTP 502: Azure client uninitialized or analysis call failed. HTTP 500: unhandled exception, logged via `logger.critical` with stack trace.
- Errors are delivered as structured JSON envelopes by `@app.exception_handler(OCRError)` (dynamic status code) and `@app.exception_handler(Exception)` (500).
- Anchor sets, e.g. license front: `'driving license', 'License No.', 'Place of Issue'` plus an Arabic ID-number phrase; license back: `'Traffic Code No.', 'Permitted Vehicles', 'Police Vehicles', 'Give way to', 'This license'`; residence front: `'United Arab Emirates', 'Resident Identity Card', 'Port Security', 'Federal Authority', 'ID Number', 'Signature', 'nationality'`; residence back: `'Card Number', 'if you find this card', 'please return it to', 'Organization', 'nearest police station'`; vehicle front: `'vehicle License', 'Place of Issue', 'Traffic Plate No', 'Policy No', 'Mortgage  By', 'Ins. Exp.'`; vehicle back: `'vehicle Information', 'Model', 'changes to vehicle', 'owner Infomation must be notified', 'G. V. W', 'Veh. Type', 'Empty Weight'`.

**Configuration constants and thresholds (the document's "Implementation Constants & Thresholds" table, presented as verified from code):** `ANCHOR_MATCH_THRESHOLD` 0.5 (50%); `REPORT_QUALITY_MIN_FIELDS` `> 3` non-empty; `INSURER_FUZZY_THRESHOLD` 85 (RapidFuzz token score, 0-100); `CATEGORY_FUZZY_THRESHOLD` 0.6 (SequenceMatcher ratio for license-back vehicle categories); `TESSERACT_RASTER_DPI` 150; `TESSERACT_CANDIDATE_PATHS` 3 (env var, `shutil.which`, AppData); `LOG_ROTATION_MAX_BYTES` 10,485,760; `LOG_BACKUP_COUNT` 5; `DUBAI_WORKER_THREADS` 6; `DUBAI_SCANNED_PAGE_SCORE` `> 2` of 7 keywords; `INSURERS_MASTER_COUNT` 42 records.

**Automation workflows:** automatic Sharjah old/new selection (no manual pre-sorting); automatic scanned-vs-editable branching; automatic insurer-code resolution and translation; container startup via `python main.py` (which invokes `uvicorn.run('main:app', host='0.0.0.0', port=8000)`) or `uvicorn main:app --host 0.0.0.0 --port 8000`; Docker build/run (`docker build -t uae-ocr .`, `docker run -p 8000:8000 --env-file .env uae-ocr`).

**Optimization techniques:** PyMuPDF bypass avoiding per-page Azure fees and network latency; Tesseract only on a cropped header region; Dubai page pre-filtering so only qualifying pages are billed; `ThreadPoolExecutor(max_workers=6)` for Dubai parallelism; sync endpoints offloaded to the AnyIO pool.

**Performance improvements:** The document states local extraction runs "in milliseconds" and that parallelism "accelerates vector PDF table slicing"; no measurements are provided.

## 7. Challenges and Solutions

1. **Three jurisdictions, multiple layouts (technical).** Sharjah has old and new templates; Abu Dhabi and Dubai arrive as editable PDFs or raster scans. *Solution:* one Azure custom model per layout, a Tesseract keyword vote on the page-1 header for Sharjah (ties -> new), and `is_scanned_pdf` to branch local vs cloud. *Alternatives:* a composed Azure model with built-in classification, or a small image classifier (analyst suggestion); the local vote is free and fast.

2. **Cloud OCR cost and latency (business/technical).** Each page sent to Azure incurs a fee and a round-trip. *Solution:* parse native text layers locally with PyMuPDF; pre-filter Dubai scan pages (`> 2` of 7 keywords). *Alternatives:* send everything to Azure, or run a full local OCR stack end-to-end (analyst suggestion) at the cost of accuracy on Arabic scans.

3. **Wrong, blank, or upside-down uploads (business).** *Solution:* anchor-phrase verification at a 50% hit rate for cards and a `> 3` non-empty-field gate for police reports, both HTTP 422. *Alternatives:* Azure confidence-score thresholds or a dedicated document-type classifier (analyst suggestion); anchors are deterministic and explainable to auditors.

4. **Free-text insurer names in two languages (technical).** *Solution:* RapidFuzz token-sort and token-set scoring (threshold 85) against Arabic and English aliases for 42 `insurers.json` records, emitting codes such as ADNIC -> `2600040`. *Alternatives:* embedding-based or LLM matching (analyst suggestion); deterministic matching was preferred for auditability.

5. **Arabic handling (technical).** Arabic must render, translate, and match correctly, and Azure returns Dubai vehicle strings as unordered word polygons. *Solution:* keep unshaped Arabic for translation/lookup and reshaped Arabic (arabic-reshaper + python-bidi) for display; sort polygons right-to-left; Azure Translator for `*_english` fields.

6. **Blocking work in an async framework (technical).** *Solution:* plain `def` endpoints dispatched to the AnyIO worker pool, plus `ThreadPoolExecutor(max_workers=6)` inside the Dubai parser. *Alternative:* async Azure SDK clients (analyst suggestion), which would still require executor jobs for CPU-bound PDF/Tesseract work.

7. **Tesseract portability (technical).** Tesseract is a native binary that must exist on every developer machine and in every container. *Solution:* three-path discovery (`TESSERACT_PATH` env var -> `shutil.which('tesseract')` -> AppData local install path) and a Docker image that bundles Tesseract OCR alongside the OpenCV/PyMuPDF runtime dependencies. *Analyst observation (the document states the behavior but does not flag it as a problem):* a missing binary surfaces as HTTP 422, conflating a server configuration fault with a client error; 500/503 would be more accurate.

8. **Contract-hygiene and hardening gaps (analyst observations drawn from documented facts; the document does not itself flag them):** `/health` reports `"service": "AI OCR API"` while the app title is `'UAE OCR API'`; response fields carry typos (`visiblity_condition`, `chasiss_no`) that are now part of the published contract; Azure SDKs are pinned to beta (`>=1.0.0b1`); insurer-code resolution applies only to police documents (per the architecture diagram), so `vehicle_front` returns `insurance_company` without a code; the full response schema is shown only for Rafid and license-back, with Saaed and Dubai described in prose; the Rafid error list omits the field-count quality gate; no authentication, rate limiting, tests, or CI/CD are described anywhere in the document.

## 8. Impact Analysis

- **Business impact:** Unified multi-emirate claims pipeline — disparate, non-standard accident report formats from three police jurisdictions normalized into a single JSON schema "across the organization"; automated liable-carrier identification for inter-company subrogation recovery; protection of core policy/claims databases from corrupt or truncated records; bilingual output preserving formatted Arabic presentation strings for legal audits and regulatory compliance.
- **Maintainability impact:** The flat, star-shaped architecture isolates domain logic into single-responsibility modules importing exclusively from `utils.py`, so (per the document) changes to one emirate's reporting template or Azure model do not risk introducing regressions in other document classes.
- **Portability impact:** A hardened Dockerfile bundling Python 3.11-slim, Tesseract OCR, OpenCV/PyMuPDF runtime dependencies, and a non-root uvicorn configuration is stated to allow "instant deployment" across on-premise servers, Azure Container Apps, or Kubernetes clusters.
- **Compliance impact:** Strict `DD/MM/YYYY` date standardization and right-to-left word reconstruction are stated to prevent transcription errors in policy enforcement and statutory reporting.
- **Productivity improvements:** Removes manual transcription of accident numbers, dates, coordinates, vehicle specs, damage descriptions, and driver identities, and manual pre-sorting of Sharjah reports.
- **Accuracy improvements:** No figures provided. Qualitatively, date normalization, RTL reconstruction, anchor gating, and insurer-code resolution are stated to prevent transcription errors and invalid data propagation.
- **Cost savings:** No monetary figures provided. The PyMuPDF bypass is stated to eliminate per-page cloud transaction costs for digital Abu Dhabi and Dubai reports; Dubai page filtering limits Azure submissions.
- **Time savings:** No measured latencies. Local extraction is stated to complete "in milliseconds"; parallel Dubai parsing accelerates table slicing.
- **User benefits:** Claims processors get structured, bilingual, code-resolved party data; integrating systems get a stable JSON contract with actionable HTTP errors; management gets a health probe and rotating logs.

The document provides no quantified accuracy, throughput, or cost-savings metrics; its figures are configuration constants and scope counts (10 endpoints, 10 custom Azure models, 3 emirates, 6 card layouts, 42 insurer records, 4 documented target-user groups, and the thresholds 50%, `> 3` fields, 85, 0.6, 150 DPI, 6 worker threads, `> 2` of 7 keywords, 10 MB / 5 log backups, 3 Tesseract lookup paths).

## 9. Interview Discussion Points

- Why route on the presence of a PDF text layer rather than always calling Azure: cost per page, latency, native-text fidelity; the risk of hybrid PDFs with partial text layers.
- How the Sharjah old/new classifier works and why Tesseract at 150 DPI on only the top third of page 1; keyword voting and the tie-break to new.
- Anchor verification: why a 50% hit rate, how anchor sets were chosen per card, how to tune false rejects vs false accepts.
- The `> 3 non-empty fields` gate: what it catches (blank, upside-down, wrong-type reports) and what it misses (partially wrong extractions).
- RapidFuzz token-sort vs token-set, why threshold 85 across Arabic and English aliases, and handling a new insurer or a near-tie.
- Arabic handling: reshaped vs unshaped strings, bidi layout, right-to-left polygon sorting for `makeModelColorYear`, translating the unshaped form.
- Concurrency: why endpoints are `def` not `async def`, what AnyIO does with them, and why `ThreadPoolExecutor(max_workers=6)` sits inside the Dubai parser.
- Error contract: `OCRError` -> 400/404/422/502, the generic 500 handler, and the debatable 422 for a missing Tesseract binary.
- Ten custom Azure models: how they were labeled/trained (be candid that the document does not say), how model IDs are managed, how you would version or retrain them.
- Deployment: Python 3.11-slim, bundled Tesseract, non-root uvicorn, `.env` secrets, on-prem/Azure Container Apps/Kubernetes; what is missing (auth, rate limiting, CI/CD, tests).
- What to measure next: field-level accuracy per document type, Azure calls avoided, p95 latency local vs cloud, rejection rates by reason.
- Schema hygiene: `visiblity_condition` / `chasiss_no` and the `/health` name mismatch, and evolving the contract without breaking consumers.
- Why one custom Azure model per layout (ten of them) rather than one composed/classifier model: per-layout labeling effort and model sprawl versus higher per-template accuracy and the ability to change one emirate's template without regressing others — tie this to the documented "no regressions in other document classes" benefit of the star-shaped module layout.
- Why Sharjah has no local text-layer bypass: the document documents the PyMuPDF zero-cost path only for Abu Dhabi and Dubai, and always sends Rafid documents to Azure after classification — was that a data reality (Rafid reports arrive as scans) or unfinished work?
- Why insurer-code resolution runs only on police documents: `vehicle_front` extracts `insurance_company` but emits no `insurance_company_code`, so a downstream consumer gets an unresolved free-text carrier name from the Mulkiya path — deliberate scoping to the subrogation use case, or a gap?
- Where the anchor check gets its text: the document says anchors are matched against "extracted OCR lines" but does not say whether those come from the Azure response or a separate local Tesseract pass — a meaningful cost difference, since anchor verification after the Azure call still pays for the call it rejects.
- The four documented consumer classes — internal claims/policy systems, claims operations teams, executive/technical management, and **external customer-facing intimation portals** — and what it means that an externally reachable upload path has no documented authentication or rate limiting.
- Error-envelope design: one `OCRError` type carrying a dynamic status code versus per-condition exception classes, and why the API returns a structured actionable envelope rather than a bare error, given the stated goal of preventing invalid data propagation into downstream policy and claims platforms.
- Operability story and its limits: `/health` (status, service, version), `RotatingFileHandler` at 10 MB with 5 backups, and what is absent — request tracing, per-model call metrics, alerting, and any record of Azure spend actually avoided.
- Beta Azure SDK pins (`azure-ai-documentintelligence`, `azure-ai-translation-text` at `>=1.0.0b1`) plus open-ended `>=` constraints across the whole stack: reproducibility risk and how you would move to pinned or hash-locked builds.

## 10. Architecture Explanation Points

The document's own Figure 1 draws five boxes — FastAPI service (validation and routing) -> routing endpoints -> Azure Document Intelligence (extracts fields) -> Normaliser (insurer code, only police docs) -> JSON conversion — which is the skeleton to expand on when whiteboarding.

Start with the client: an internal claims system or an external intimation portal POSTs a PDF or image to one of the nine POST extraction endpoints (ten routes in total including `GET /health`). Draw `main.py` as the hub that validates the upload, with `sharjah.py`, `abudhabi.py`, `dubai.py`, and `id_card_parser.py` as spokes sharing only `utils.py`. Next draw the decision diamond: does the PDF have a native text layer? Yes -> PyMuPDF parses tables locally in milliseconds with no cloud cost. No -> a cheap Tesseract pass either picks the right Azure model (Sharjah old vs new) or filters which pages are worth sending (Dubai), then the scan goes to one of ten custom Azure Document Intelligence models. Below that, draw the normalization strip: anchor/field-count gates returning HTTP 422, `DD/MM/YYYY` dates, Arabic reshape/bidi plus RTL polygon sorting, Azure Translator for English fields, and RapidFuzz resolution of insurer names to internal codes from `insurers.json`. Finish with the JSON envelope and the handler mapping `OCRError` to 400/404/422/502 and everything else to 500.

Key decisions, each with the reason to state out loud: (1) **cost-first hybrid routing** — check for a native text layer before spending a cloud call, because per-page Azure Document Intelligence charges and a network round-trip are avoidable whenever the PDF already carries high-fidelity text; (2) **cheap local OCR as a router, not an extractor** — Tesseract runs only on a 150 DPI crop of the page-1 header (Sharjah) or on page crops (Dubai), so it decides *which* model or *which* pages get paid for, and its own accuracy on Arabic never reaches the output; (3) **deterministic, auditable validation instead of model confidence scores** — anchor keyword hit rates and non-empty-field counts are explainable to auditors and stable across model retrains; (4) **one custom model per layout** rather than one generalized model, so each of the ten templates can be retrained independently; (5) **sync `def` endpoints on the AnyIO pool** because every heavy step (Azure I/O, PyMuPDF rendering, Tesseract) is blocking, plus a nested `ThreadPoolExecutor(max_workers=6)` where Dubai page and party extraction can genuinely run in parallel; (6) **a self-contained Docker image** with Tesseract and the OpenCV/PyMuPDF runtime baked in and a non-root uvicorn user, so the same artifact runs on-premise, on Azure Container Apps, or on Kubernetes; (7) **fail fast at the edge** — a 422 at the boundary is cheaper than a corrupt record in the core claims database.

Trade-offs to concede: Tesseract is a native dependency with a startup failure mode that currently surfaces as a client-side 422; the modular one-model-per-layout design multiplies labeling and lifecycle work across ten Azure models with no documented retraining or versioning process; the insurer master is a 42-record flat file rather than a database, so adding a carrier is a redeploy; insurer resolution is wired only into the police path; the Azure SDKs are beta pins and every dependency is an open `>=` range; and there is no auth, rate limiting, test suite, or CI/CD at the service layer even though one documented consumer class is an external customer-facing portal. Next improvements: per-field accuracy telemetry and Azure-call-avoidance metrics, async Azure clients with bounded concurrency, confidence-score fallbacks alongside anchors, an API-key/JWT layer, contract tests pinning the response schema (typos included, then a versioned migration off them), and a CI pipeline that builds, scans, and pushes the image.


# Project 14: LLM Fine-tuning Platform (Unsloth / Gemma / Qwen / vLLM / llama.cpp / Streamlit)

Source: "Fine tune LLM", document no. AT/UNSLOTH/V1, dated 01-Sep-2026, client Qatar Insurance Group. Items marked "(inferred)" are analyst inferences; all other statements come from the document.

## 1. Project Overview

- **Project name:** Fine tune LLM (AT/UNSLOTH/V1; labeled "unsloth_version" in the architecture figure).
- **Business domain:** Insurance. KYC document verification for underwriting and onboarding at Qatar Insurance Group (QIC / Anoud), traceable from corporate network paths and the API key prefix 'QIC-Anoud-'.
- **Problem statement:** KYC staff manually transcribe identity and vehicle cards (mobile photos .jpg/.jpeg, scans .png) covering six UAE categories (Driving License Front/Back, Resident ID Front/Back, Vehicle Registration "Mulkiya" Front/Back) and three Doha categories (Driving License, Resident Permit, Vehicle License) with bilingual Arabic-English layouts. Commercial OCR APIs cost per call and expose PII to external vendors, while raw LLM output is unreliable: the Phase 1 audit found 12 of 30 flagged field labels wrong, including a day/month swap ('1984-11-03' vs actual '11-03-1984').
- **Project objective:** (a) Use a remote teacher VLM (Qwen3.5-122B-A10B on vLLM) with Pydantic-guided decoding to produce candidate labels; (b) verify and correct them in a Streamlit tool with audit provenance; (c) restructure the audited corpus into Unsloth conversational format and version it on Hugging Face Hub; (d) fine-tune Gemma 4 E2B with 4-bit BitsAndBytes QLoRA; (e) quantize to GGUF ('gemma-4-E2B-it-Q4_K_S.gguf') and serve privately on llama.cpp.
- **Expected business outcome:** Schema-compliant KYC JSON at zero per-call API cost, on-premise data residency compliant with UAE PDPL and Qatar financial data standards, exception-only human review, complete lineage from raw capture to training split, and a proprietary Document-AI asset trained on 664 verified UAE and 296 Doha documents.
- **Target users (as listed in the document):** KYC operations and document verification staff at QIC / Anoud; human reviewers and annotators ('Afsal', 'waris', 'peter'); Document-AI and ML engineers maintaining curation, fine-tuning, quantization and inference endpoints; and downstream insurance core applications (internal microservices and policy administration platforms) consuming validated KYC JSON over REST.

## 2. Role and Responsibilities

The document never names the candidate's title. The Hugging Face namespace 'Peterleo5/uae-ocr-test' and the reviewer identity 'peter' in the annotation logs indicate the candidate personally owned dataset staging and took part in annotation alongside 'Afsal' and 'waris' (inferred).

**Core responsibilities (inferred from the system as built):**

- Designed the five-stage architecture: Document Capture -> Teacher VLM -> Streamlit Audit -> Unsloth Fine-Tune -> llama.cpp Host.
- Built the teacher-distillation pipeline in 'OCR training.ipynb' against Qwen3.5-122B-A10B on vLLM 0.27.1 with Pydantic 'response_format' guided decoding, temperature 0.0, top_p 1.0 (0.1 in the LangChain client), enable_thinking=False, cache_prompt=True, 60 s timeout / 2 retries (120 s under LangChain).
- Authored the six Pydantic 2.13.4 schemas (ResidentFrontSchema, ResidentBackSchema, DrivingLicenseFrontSchema, DrivingLicenseBackSchema, VehicleFrontSchema, VehicleBackSchema) and the per-document field catalog.
- Engineered the Streamlit 1.60.0 review dashboard 'demo.py': login gate, document filter, side-by-side image/field layout, real-time diffs, 'edited_by' / 'edited_fields' audit stamps, auto-save on navigation.
- Ran the forensic dataset audit ('UAE_Card_Extraction_Audit.md') proving unaudited teacher labels would corrupt the student.
- Designed the Unsloth 'messages' dataset format, the stratified 85/15 split (564 train / 100 validation), and the Parquet push to Hugging Face Hub.
- Configured and iterated the QLoRA recipe (Run 1 YAML, then 'unsloth_config_v2.yaml'): r=16, alpha=16, dropout 0.0, lr 1e-4, adamw_8bit, batch 2 x accumulation 4, target modules q/k/v/o/gate/up/down_proj, frozen vision layers, train_on_completions.
- Led root-cause analysis of the Run 1 memorization failure ('FINETUNE_REPORT.md') and redesigned the recipe.
- Exported merged weights to 4-bit GGUF and stood up llama.cpp on 'http://172.20.132.97:8000/' with multimodal 'mmproj', verified via /v1/models (notebook cell 237).

**Supporting tasks:**

- Image ingestion with aspect-preserving scaling and RGB conversion (Pillow 12.3.0); base64 data-URL encoding.
- Dual client integration: OpenAI Python SDK 2.52.0 and LangChain Core/OpenAI 1.5.4 / 1.5.0.
- Dual-pass audit dataset ('dataset_600C.json') and lineage across 'doha_teacher_dataset.json', 'annotated_dataset.json', and Hub splits.
- Documentation builders ('build_all_artifacts.py', 'generate_doc.py') using python-docx 1.2.0 and openpyxl 3.1.5.
- Held-out evaluation of Run 1 (e.g., 'dba_10.jpeg'); local inference client to llama.cpp (temperature 0.0, max_tokens 1500).
- Maintained the three documented execution entry points and their verified launch commands: 'streamlit run demo.py', 'python build_all_artifacts.py', and 'jupyter notebook "OCR training.ipynb"'.
- Curated the initial 29-image PoC corpus in 'docs/UAE_OCR/dataset_30/' and the scaled 664-record corpus ('uae_dataset_600').

**Estimated ownership level: Senior AI / Machine Learning Engineer (inferred).**
Rationale (inferred): the candidate worked across every layer of the pipeline — remote teacher integration, an annotation tool used by three reviewers across 664 UAE + 296 Doha records, dataset versioning on Hugging Face Hub, a training recipe with two documented iterations, a forensic failure investigation that changed the recipe, and a quantized on-premise deployment verified live via /v1/models. The document explicitly notes there is no custom FastAPI/Flask middleware, and the pipeline is notebook-centric with no documented CI/CD, version control or monitoring, so architect- or lead-level titles are not claimed here. Title, team size and reporting lines are not stated in the document.

## 3. End-to-End Workflow

**Processing stages:**

1. **Capture and ingest.** Card images (.jpg/.jpeg/.png) are scaled with aspect preservation, converted to RGB, and classified by document category (classification method not detailed).
2. **Teacher extraction (Stage 1).** Base64 'image_url' messages go to POST http://78.100.128.116:8001/v1/chat/completions on vLLM 0.27.1 serving 'Qwen/Qwen3.5-122B-A10B-GPTQ-Int4' (max_model_len 100,000 tokens) with 'response_format' set to the Pydantic class; the parsed dict is read from 'choices[0].message.parsed'. Logged throughput: 553 images in 18 min 41 s (2.03 s/image, cell 127).
3. **Human review (Stage 2).** JSONL records ('image_path', 'document_type', 'target.fields') are opened in 'demo.py'; 'save_current' computes changed fields, stamps 'edited_by' and 'edited_fields', and serializes to disk.
4. **Restructuring and staging (Stage 3).** Audited records become Unsloth 'messages' with explicit image token references and strict-JSON assistant turns; stratified 85/15 split (564/100); Arrow tables written as Parquet shards ('messages', 'images' columns) and pushed to https://huggingface.co/api/datasets/Peterleo5/uae-ocr-test with Bearer HF_TOKEN.
5. **QLoRA fine-tuning (Stage 4).** Gemma-4-E2B-it loaded in 4-bit BitsAndBytes through Unsloth 2026.8.18 / TRL 0.24.0 / Transformers 5.5.0 / PyTorch 2.11.0; adapters applied per 'unsloth_config_v2.yaml'.
6. **Merge and quantize.** Merged LoRA weights exported to 'gemma-4-E2B-it-Q4_K_S.gguf' (3,028,113,548 bytes; 4,647,450,147 params).
7. **On-premise serving.** llama.cpp at http://172.20.132.97:8000/ with 'mmproj' (n_ctx 65,536); downstream systems call POST /v1/chat/completions and receive a JSON string in 'choices[0].message.content'.

**Data flow:** image files -> base64 data URLs over HTTP -> Pydantic-validated dicts -> JSONL on disk -> edited JSONL with audit fields -> Unsloth 'messages' + PIL images -> Parquet on Hugging Face Hub -> LoRA adapters -> merged GGUF -> JSON responses to insurance core applications.

**Integrations and dependencies:** remote vLLM teacher (bearer key prefixed 'QIC-Anoud-'); Hugging Face Hub (datasets 4.3.0, huggingface_hub 1.26.0); llama.cpp server (version "not specified in the project"); OpenAI SDK and LangChain clients; Unsloth / Unsloth Zoo; Streamlit on port 8501.

**Execution entry points:** the system has three, not a monolithic web app — 'demo.py' (Streamlit HITL dashboard), 'OCR training.ipynb' (distillation, forensic validation, Hub staging) and 'build_all_artifacts.py' / 'generate_doc.py' (technical document builders). Verified launch commands: 'streamlit run demo.py', 'python build_all_artifacts.py', 'jupyter notebook "OCR training.ipynb"'. No custom FastAPI/Flask middleware routes exist.

**Documented project phases:** Phase 1, 29-sample PoC and forensic audit (13-14 Aug 2026, 'docs/UAE_OCR/dataset_30/'); Phase 2, 'demo.py' HITL tool development; Phase 3, scaling to 664 records ('uae_dataset_600') and QLoRA Run 1 (18 Aug 2026, failed); Phase 4, recipe redesign ('unsloth_config_v2.yaml') and local GGUF edge serving.

**Flow line:**
`Card image -> PIL (scale/RGB) -> base64 -> vLLM Qwen3.5-122B (Pydantic guided JSON) -> JSONL -> Streamlit demo.py (edited_by/edited_fields) -> Unsloth messages -> Parquet -> HF Hub -> Unsloth QLoRA Gemma-4-E2B -> merge -> GGUF Q4_K_S -> llama.cpp :8000 -> JSON`

**Architecture Summary:** A teacher-student distillation loop wrapped in governance. A large quantized teacher produces schema-constrained candidates quickly; a purpose-built review UI turns them into audited ground truth with provenance; the corpus is versioned on the Hub; a small 4-bit student is adapted with LoRA and shipped as one GGUF file to an internal llama.cpp host. Every hop uses OpenAI-compatible REST, so the same client code targets teacher and student; no bespoke middleware exists.

## 4. Technologies and Tools Used

| Category | Technologies (pinned versions where documented) |
|---|---|
| Programming languages | Python 3.11.15; YAML training recipes ('unsloth_config_v2.yaml') |
| Frameworks | Streamlit 1.60.0; PyTorch 2.11.0; Transformers 5.5.0; Unsloth 2026.8.18 / Unsloth Zoo 2026.8.12; TRL 0.24.0; LangChain Core 1.5.4 / LangChain OpenAI 1.5.0; vLLM 0.27.1; llama.cpp (version not specified) |
| Libraries | Pydantic 2.13.4; OpenAI Python SDK 2.52.0; Hugging Face Datasets 4.3.0; huggingface_hub 1.26.0; Pillow 12.3.0; scikit-learn 1.9.0; Matplotlib 3.11.1; BitsAndBytes (4-bit; version not stated); python-docx 1.2.0; openpyxl 3.1.5 |
| AI/ML models | Qwen3.5-122B-A10B ('Qwen/Qwen3.5-122B-A10B-GPTQ-Int4') teacher; Google Gemma 4 E2B (Gemma-4-E2B-it) student; deployed 'gemma-4-E2B-it-Q4_K_S.gguf' with multimodal 'mmproj' |
| OCR tools | Not stated in documentation (no conventional OCR engine is named; OCR/KIE is performed end-to-end by the teacher and student VLMs) |
| Databases | Not stated in documentation (JSONL files on disk and Parquet shards on Hugging Face Hub act as data stores) |
| Cloud platforms | Hugging Face Hub ('Peterleo5/uae-ocr-test'); remote vLLM host at 78.100.128.116 (provider not stated) |
| APIs | OpenAI-compatible /v1/chat/completions (vLLM, llama.cpp); llama.cpp GET /v1/models; Hugging Face Hub datasets API; Streamlit HTTP/WebSocket UI on localhost:8501 |
| DevOps tools | Not stated in documentation |
| Version control | Not stated in documentation (dataset versioning via Hugging Face Hub commits) |
| Deployment tools | llama.cpp server on port 8000; Streamlit CLI ('streamlit run demo.py'); Jupyter Notebook |
| Document processing tools | python-docx 1.2.0; openpyxl 3.1.5; Pillow 12.3.0 |
| Automation tools | Jupyter pipeline ('OCR training.ipynb'); 'build_all_artifacts.py' / 'generate_doc.py' builders |

## 5. Technical Skills Demonstrated

- **Software Engineering:** Session-state auth gate ('st.session_state.user', '?user=<name>', 'st.stop()'); unique widget keys (f"{image_path}: {field_name}"); save-before-navigate in 'move(delta)'; JSONL serialization; typed Pydantic schemas; documented HTTP error modes (401/422/500/504).
- **AI Engineering:** Teacher-student distillation; guided JSON decoding via 'response_format'; OpenAI SDK and LangChain integration; disabling Qwen thinking mode; prompt caching; constant 6-schema catalog prompt in v2.
- **Machine Learning:** QLoRA with 4-bit BitsAndBytes; LoRA hyperparameters (r=16, alpha=16, dropout 0.0, all linear projections); 8-bit AdamW with linear decay; gradient accumulation; epoch reduction (5 -> 2); diagnosing memorization via held-out inference; adding a validation split.
- **NLP:** Structured KIE with strict key sets and null handling; ISO date normalization; bilingual Arabic-English extraction; completion-only loss masking ('train_on_completions: true').
- **Computer Vision:** Multimodal VLM fine-tuning; freezing vision layers to stay compatible with the stock GGUF 'mmproj'; image preprocessing and base64 transport.
- **Data Engineering:** JSONL record design; stratified splitting; Arrow/Parquet sharding; Hub versioning; forensic audit for duplicates, swapped dates, and unit contamination; lineage via 'dataset_600C.json'.
- **Cloud:** Hugging Face Hub as versioned dataset store with token auth. No hyperscaler usage documented.
- **MLOps:** YAML-driven, versioned training recipes; GGUF Q4_K_S export; deployment verification via /v1/models metadata; deterministic greedy decoding.
- **API Development:** No custom middleware; skill shown is API consumption and contract documentation against OpenAI-compatible endpoints.
- **Prompt Engineering:** System/user turns with image tokens; schema-constrained assistant outputs; constant catalog prompt to prevent the student conditioning on prompt variation.
- **System Design:** Five-stage governed pipeline; HITL gate placed before training; on-prem inference for PII compliance; teacher capacity (122B remote) vs student cost (E2B local); single REST contract across teacher and student.

## 6. Detailed Technical Contributions

**Features implemented**
- 'demo.py': login prompt 'Enter your name to continue' setting 'st.session_state.user' and '?user=<name>' before 'st.rerun()'. Sidebar 'Document' selectbox ('all' + sorted types) and 'Image (<N> total)' selectbox keyed 'index-<doc_type>'. Main panel 'st.columns([3, 2])': 'st.image(..., width=400)' left; 'st.container(height=550)' right with one 'st.text_input' per key in 'target["fields"]'. 'Save' -> 'save_current(record)' -> 'Changed : <field1>, <field2>' or 'No changes to save'. 'Prev'/'Next' -> 'move(-1)'/'move(1)', saving before the index changes.
- Teacher client with Pydantic guided decoding replacing Phase 1 regex fence-stripping ('find("{")...rfind("}")').
- Unsloth 'messages' dataset builder; Parquet push to 'Peterleo5/uae-ocr-test'.
- QLoRA recipes (Run 1 YAML; 'unsloth_config_v2.yaml'); GGUF export; llama.cpp deployment; documentation builders.

**Models used**
- Teacher: Qwen3.5-122B-A10B GPTQ-Int4 on vLLM 0.27.1, port 8001.
- Student: Gemma-4-E2B-it, 4-bit BitsAndBytes QLoRA via Unsloth.
- Deployed: 'gemma-4-E2B-it-Q4_K_S.gguf' (owned_by 'llamacpp', n_vocab 262144, n_ctx 65536, n_params 4647450147, size 3028113548 bytes, capabilities ['completion', 'multimodal']).

**Pipelines built**
- Distillation notebook: cells 1-12 (29-sample PoC), cells 117-124 (restructuring and 564/100 split), cell 127 (553-image batch log), cell 237 (server verification).
- Review pipeline: JSONL in -> edited JSONL with audit fields out.
- Training pipeline: YAML recipe -> Unsloth/TRL -> LoRA adapters -> merge -> GGUF.

**APIs integrated**
- vLLM teacher POST /v1/chat/completions (Bearer <LLM_VL_KEY>; 401 invalid token, 422 malformed image URL, 500/504 timeout).
- llama.cpp GET /v1/models (Bearer none; 500 or connection refused when down) and POST /v1/chat/completions ('temperature' 0.0, 'max_tokens' 1500; 400 if image dimensions exceed token limits, 500 if mmproj fails).
- Hugging Face POST /api/datasets/Peterleo5/uae-ocr-test (Bearer HF_TOKEN; returns commit hashes and shard URLs; 401/403).
- Streamlit GET localhost:8501 (optional 'user' query parameter).

**Data extraction methods**
- Zero-shot VLM extraction against rigid Pydantic schemas; absent attributes nulled rather than hallucinated.
- Field catalog examples: driving_front (license_authority_id, license_number, name, nationality, date_of_birth, issue_date, expiry_date, place_of_issue); resident_front (id_number e.g. '784-1991-6138598-6', name, date_of_birth, nationality, issuing_date, expiry_date, sex 'M'/'F'); vehicle_back (chassis_number 17-character VIN, engine_number, vehicle_type e.g. 'CHEVROLET CAMARO', model 4-digit year, gross_vehicle_weight kg, empty_weight kg, origin country). Dates normalized to YYYY-MM-DD.
- Remaining catalogued types: driving_back (traffic_code_number, transmission_type e.g. 'Automatic Gear' / 'MANUAL' / null if unrestricted, permitted_vehicle e.g. 'Light Vehicle', 'Motorcycle'); resident_back (card_number at top-left below the label, employer or free-zone entity, occupation, issuing_place, plus legacy date_of_birth / sex / exp present only on older cards); vehicle_front (traffic_plate_number, owner, traffic_code_number 'T.C. No.', registration_date, policy_number).
- Extraction rules are positional as well as semantic: e.g. 'license_authority_id' is the alphanumeric code inside the rectangular box beneath the photograph; 'place_of_issue' is the issuing Emirate/authority (Dubai, Abu Dhabi).

**Validation logic**
- Pydantic enforcement at decode time: zero hallucinated keys, valid JSON, no markdown fences.
- Human verification of 664 UAE and 296 Doha records; real-time diffs; 'edited_by' / 'edited_fields' provenance.
- Audit findings: 12 of 30 flagged labels wrong; day/month swap; unit letter 'K' in weight; garbled Arabic place of issue; duplicate cards. Legacy 'resident_back' fields (date_of_birth / sex / exp) are 96% null.
- Login gate halts with 'st.stop()' when no username is present in session state or query parameters.
- Documented input requirements: a UTF-8 JSON Lines file (e.g. 'doha_teacher_dataset.json', 'annotated_dataset.json') with 'image_path', 'document_type' and a nested 'target.fields' object; every 'image_path' must resolve to a real file relative to the project root (.jpg/.jpeg/.png); a non-empty reviewer username is mandatory for audit traceability.
- Null handling in the UI: null field values are rendered as empty strings in the text inputs.

**Automation workflows**
- Batch teacher inference with 60 s timeout / 2 retries (120 s under LangChain); auto-save on navigation; automated Parquet sharding and Hub push; automated document artifact generation.

**Optimization techniques**
- 4-bit QLoRA, adamw_8bit, LoRA dropout 0.0, effective batch 8, max sequence length 2048 (YAML) / 3072.
- v2 recipe: 'finetune_vision_layers: false', 'train_on_completions: true', 2 epochs, eval split.
- Teacher: greedy decoding, thinking disabled, prompt caching. Serving: Q4_K_S quantization, n_ctx 65,536, temperature 0.0, max_tokens 1500.

**Performance improvements**
- Teacher throughput: 553 images in 18 min 41 s (2.03 s/image).
- JSON syntax failures eliminated by guided decoding (no rate given); zero per-call API cost in production (no monetary figure). Whole-corpus accuracy of the student is explicitly not specified.

## 7. Challenges and Solutions

1. **Corrupted teacher labels (technical and business).** The Phase 1 audit (August 13-14, 2026) showed 12 of 30 flagged labels wrong (40% error rate on flagged fields), day/month swaps, unit contamination, and two duplicate images; training on them would teach the student to memorize errors. *Solution:* Streamlit HITL tool as mandatory quality gate plus dual-pass audit ('dataset_600C.json'). *Alternatives (analyst suggestion):* cross-teacher agreement voting, rule-based validators for dates/VINs/plates, perceptual-hash de-duplication before annotation.
2. **Brittle JSON parsing (technical).** Regex fence-stripping caused ingestion failures from unclosed brackets and markdown fences. *Solution:* vLLM guided decoding with Pydantic 'response_format'. *Alternatives (analyst suggestion):* JSON-repair post-processors or output parsers with retry; constraint at decode time is stricter.
3. **Catastrophic memorization in Run 1 (technical).** With 'finetune_vision_layers: true', 5 epochs, and no validation split, the model emitted training row 115's target ('Waheed Ahmad Khan Ghalib') on 'dba_10.jpeg', a license back with no names. The multi-agent forensic review ('FINETUNE_REPORT.md') blamed Unsloth Studio's auto-mapper misreading the 3-column format (targets learned without visual grounding), amplified by a mismatch between trained vision LoRA layers and the stock GGUF 'mmproj'. *Solution:* 'unsloth_config_v2.yaml' with frozen vision layers, 'train_on_completions: true', constant 6-schema catalog prompt, 2 epochs, eval split. *Alternatives (analyst suggestion):* export a custom mmproj matching the trained vision tower; early stopping on validation loss; augmentation to break image-target shortcuts.
4. **GPU memory constraints (technical).** Multimodal fine-tuning of a 4.6B-parameter model with vision soft tokens plus JSON. *Solution:* 4-bit QLoRA, adamw_8bit, batch 2 x accumulation 4, r=16. *Alternatives (analyst suggestion):* multi-GPU full fine-tuning, gradient checkpointing, smaller rank.
5. **PII residency and vendor cost (business).** Identity documents cannot leave internal infrastructure; commercial OCR bills per call. *Solution:* on-prem llama.cpp serving of the quantized student. *Alternative:* cloud VLM/OCR APIs, rejected on PDPL and cost grounds (documented rationale).
6. **Annotator data loss (technical / UX).** *Solution:* 'move(delta)' saves before changing index; zero data loss reported across 960 records. *Alternative (analyst suggestion):* unsaved-changes prompt.
7. **Documented limitations.** Whole-corpus accuracy, SLA throughput, ROI, and labor savings are not specified; the v2 recipe's evaluation result is not reported. The local server uses 'Bearer none' and hard-coded IPs; llama.cpp version is unspecified. The pipeline is notebook-based with no middleware or CI/CD, and the dataset sits under a personal Hub namespace ('Peterleo5') (analyst observation).

## 8. Impact Analysis

- **Business impact:** A proprietary, regionally tuned Document-AI asset (664 UAE and 296 Doha verified documents) with full lineage. Financial ROI: "Not specified in the project."
- **Productivity improvements:** Teacher pipeline processed 553 images in 18 minutes 41 seconds (2.03 s/image), described as "reducing baseline data ingestion time by orders of magnitude compared to manual transcription" (qualitative). Reviewers work exception-only. Labor reduction percentages not specified.
- **Accuracy improvements:** Guided decoding eliminated syntax and decoding failures; auditing removed the 40% flagged-field error rate from the training set. Whole-corpus exact-match accuracy not specified.
- **Cost savings:** "Zero Per-Call API Costs in Production" via local GGUF serving; monetary savings not specified.
- **Time savings:** Only the 2.03 s/image teacher figure is documented; production SLA throughput not specified.
- **User benefits:** KYC teams receive schema-compliant JSON; reviewers get side-by-side inspection with auto-save and provenance; engineers get a versioned dataset and reproducible YAML recipes; downstream systems consume a stable OpenAI-compatible contract; customer PII stays on internal infrastructure.

## 9. Interview Discussion Points

- Why distill from a 122B teacher instead of hand-labeling: correction is cheaper than transcription, but only safe behind a HITL gate, as the 40% flagged-field error rate showed.
- Explain the Run 1 failure precisely: row 115 memorized on 'dba_10.jpeg', auto-mapper misreading the 3-column format, mmproj mismatch, and why frozen vision layers plus completion-only loss restored grounding.
- Justify r=16, alpha=16, dropout 0.0, all seven projection modules, and effective batch 8 with adamw_8bit under VRAM limits.
- Why guided decoding beats regex parsing and what 'response_format' with a Pydantic class does inside vLLM.
- Why temperature 0.0, thinking disabled, and prompt caching on the teacher, and the trade-off with recall on ambiguous fields.
- How the Streamlit tool guarantees provenance and prevents data loss; how you would add role-based access.
- Data decisions: stratified 85/15 split, Parquet with 'messages' and 'images', Hub versioning, duplicate detection after the audit found two duplicates.
- Cost of Q4_K_S quantization in accuracy and why llama.cpp (n_ctx 65,536) was chosen over vLLM for the student.
- Missing metrics (exact-match accuracy, SLA, ROI) and how you would build an evaluation harness on the 100-row validation set.
- Security and ops gaps: 'Bearer none', hard-coded IPs, notebook pipeline, personal Hub namespace; how you would productionize.
- Why no custom FastAPI/Flask middleware was built: three entry points ('demo.py', 'OCR training.ipynb', 'build_all_artifacts.py') consuming standard REST from vLLM, llama.cpp and Hugging Face — the trade-off between shipping speed and having no place for auth, rate limiting or routing.
- Why two client stacks (OpenAI SDK 2.52.0 and LangChain Core/OpenAI) against the same endpoint, and the observable difference (top_p 1.0 vs 0.1, 60 s / 2 retries vs 120 s).
- Regional and bilingual specifics: Arabic-English card layouts, ISO date normalization, 17-character chassis numbers, ID formats like '784-1991-6138598-6', and legacy 'resident_back' fields that are 96% null on newer cards.
- Schema design as a product decision: six Pydantic schemas over nine card categories (six UAE, three Doha), with positional extraction rules and explicit nulling rather than a free-form JSON prompt.
- How deployment was verified rather than assumed: GET /v1/models in cell 237 returning n_params 4,647,450,147, size 3,028,113,548 bytes, n_ctx 65,536, capabilities ['completion', 'multimodal'].

## 10. Architecture Explanation Points

Draw five boxes left to right. **Capture** loads .jpg/.png card images, scales them, and encodes base64. **Teacher** is Qwen3.5-122B-A10B on vLLM 0.27.1 at port 8001; each call carries a Pydantic schema as 'response_format', so output is valid, key-exact JSON at temperature 0.0 with thinking disabled, at 2.03 s/image. **Review** is 'demo.py': reviewer logs in, sees the image (400 px) beside a 550 px scrollable field list, edits, and the tool diffs, stamps 'edited_by'/'edited_fields', and auto-saves on Prev/Next. **Train** converts audited JSONL into Unsloth 'messages', splits 564/100, pushes Parquet to Hugging Face Hub, then runs 4-bit QLoRA on Gemma 4 E2B (r=16, alpha=16, adamw_8bit, batch 2 x 4, 2 epochs, vision frozen, completion-only loss). **Serve** merges adapters, quantizes to 'gemma-4-E2B-it-Q4_K_S.gguf', and runs llama.cpp at port 8000 with 'mmproj', exposing the same /v1/chat/completions contract as the teacher.

Key decisions: constrain at decode time rather than parse; put humans before training; freeze vision layers so the stock mmproj stays valid; keep one OpenAI-compatible interface so clients are interchangeable; serve on-prem for PDPL compliance and zero per-call cost; deliberately skip custom FastAPI/Flask middleware and consume the REST APIs that vLLM, llama.cpp and Hugging Face already expose; verify the deployment from metadata (GET /v1/models, cell 237) rather than assume it.

Trade-offs: the E2B student is far cheaper than the 122B teacher but its whole-corpus accuracy is unmeasured; Q4_K_S shrinks the model to about 3 GB (3,028,113,548 bytes for 4,647,450,147 params) at some precision cost; frozen vision layers protect mmproj compatibility but limit adaptation to unusual layouts; greedy decoding (temperature 0.0, top_p 1.0, thinking disabled) buys reproducibility at the cost of recall on ambiguous or handwritten fields; effective batch 8 via accumulation fits VRAM but slows the run; a notebook pipeline iterates fast but is not CI-ready; the local host answering with 'Bearer none' behind hard-coded IPs is acceptable only inside the corporate network.

Next improvements: automated exact-match evaluation on the 100-row validation split, authentication on the llama.cpp host, a custom mmproj export to permit safe vision-layer training, de-duplication and date validators before annotation, an organizational Hub namespace, and packaging the notebook into a scheduled training job.


# Project 15: VL_version_OCR — Vision-Language OCR (FastAPI / YOLOv8 / PyPDFium2 / LangChain / VLMs)

## 1. Project Overview

- **Project name:** VL_version_OCR (document no. AT/VLV/V1, dated 01-Sep-2026). Client: Qatar Insurance Group.
- **Business domain:** Insurance — motor insurance underwriting, claims intake, and policy administration. The system ingests GCC governmental identity documents (driving licenses, residency permits / civil IDs, vehicle registrations), international passports, and traffic police accident reports from four police authorities (Saaed Abu Dhabi, Dubai Police, Rafid Sharjah, Doha Traffic Police).
- **Problem statement:** Policyholder credentials arrive as composite scans and PDF bundles containing several ID cards per page, in mixed Arabic/English, across multiple jurisdictions with different layouts. Police accident reports arrive as vector PDFs or raster scans with jurisdiction-specific table structures and bidirectional Arabic text. Downstream underwriting and claims systems need structured, schema-valid JSON — not free text — and cannot tolerate generative hallucinations. Earlier experiments with Azure Document Intelligence (documented in debug.ipynb) "struggled with multi-document scans" and exhibited "variable latency and high cloud API coupling."
- **Project objective:** Build an end-to-end multimodal document intelligence pipeline that (a) detects and crops individual sub-documents from composite uploads with YOLOv8, (b) classifies each crop and extracts entities via a two-stage Vision-Language Model pipeline with strict JSON schema enforcement, (c) parses four governmental police accident report formats with dedicated PDF/OCR engines, and (d) exposes all of this through a single FastAPI service with pluggable model backends (local Qwen3-VL, Google Gemini, Azure Llama).
- **Expected business outcome:** Straight-through processing (STP) for policy onboarding and motor claims registration; elimination of manual pre-processing of composite scans; "zero-hallucination" structured ingestion for downstream systems; bilingual Arabic-English handling; high-throughput concurrency for multi-document pages and multi-page police reports; cross-border regional scalability into a single unified API architecture; and model-provider agility with an on-premise inference option for data sovereignty.
- **Target users (as stated in the source):** (1) downstream Qatar Insurance Group underwriting, claims-intake and policy-administration systems (client inferred by the documentation from the repository path and the `.env` service-account name `ocrqicuser`); (2) claim-operations and underwriting personnel processing motor claims, driver identity verification and liability assessment; (3) API integration clients — microservices and backend integrators calling the endpoints for batch OCR and automated ingestion. The source states that specific end-user personas, employee IDs and external client roles are "Not specified in the project."

## 2. Role and Responsibilities

**Core responsibilities (inferred from the system built):**
- Designed the decoupled pipeline architecture: FastAPI router → YOLOv8 splitter → unified VLM engine → JSON, plus a parallel police-report controller path.
- Applied a custom-trained YOLOv8 multi-document detection model (weights `yolo-multipage-OCR-cls.pt`, 54.5 MB, loaded at `services/yolo_splitter.py:8`; alternate checkpoint `best.pt`, 45.2 MB, referenced only in debug.ipynb) and tuned detection thresholds (conf=0.5, IoU=0.4) in debug.ipynb cells 116-138 before promoting them to `services/yolo_splitter.py:39-40`. (The source describes the weights as "trained" but never states who trained them — candidate authorship of the training run is *inferred* and unconfirmed.)
- Built the pluggable asynchronous VLM execution engine (`services/llm_engine.py`) supporting both native AsyncOpenAI and LangChain Core paradigms across Qwen3-VL-4B-Instruct (local vLLM), Gemini-3-Flash-Preview (Google GenAI), and Dev-Llama-4-Scout-17B-16E-Instruct (Azure AI Services).
- Designed the two-stage classification/extraction prompt pipeline: country-specific keyword anchor rules (`DOCUMENT_ANCHORS`) for document typing, and dynamic strict JSON schema synthesis from `OUTPUT_DICT` (strict=True, temperature=0, additionalProperties=False).
- Authored jurisdictional schema configurations for 18+ document types across UAE, Qatar/Doha, Oman, Kuwait, and six passport nationalities (`config/document_anchors.py`, `config/output_keys.py`).
- Built four police accident report parsers in `services/police_docs/` combining PyMuPDF vector table extraction, Google Cloud Vision OCR, Wand/ImageMagick rendering, and a custom 2D coordinate sorting algorithm.
- Implemented the REST API surface (`app.py`, `routers/extract.py`): `app = FastAPI()` with two mounted routers — `router = APIRouter(prefix='/extract', tags=['OCR'])` and `pd_router = APIRouter(tags=['Police_docs'])` — giving a health-check `GET /`, five `/extract/{country}` endpoints and four police endpoints, with model selection via a required enum query parameter, `asyncio.gather` concurrency, HTTP 400 semantics, and gzip-compressed JSON error responses.
- Authored the per-jurisdiction output field contracts, e.g. UAE (`license_number`, `date_of_birth`, `owner_name_english`, `traffic_plate_number`), Oman `driving_license_front` (`license_number`, `issued_place_english`, `name_english`, `first_issued`, `expiry_date`), Kuwait `residency_permit_front` (`civil_id`, `name`, `passport_number`, `nationality`, `date_of_birth`, `expiry_date`), Doha `residency_permit_front` (`idNo`, `dob`, `expiry`, `nationalityEng`, `nationalityAra`, `nameEng`, `name_ar`), passports (`type`, `code`, `passport_number`, `given_name`, `fullname`, `date_of_birth`, `date_of_expiry`) — including the deliberate Kuwait exception where vehicle registration uses a single `vehicle_registration` key rather than separate front/back keys (inferred).

**Supporting tasks (inferred):**
- Ran and recorded an experimentation program (debug.ipynb, 214 cells) covering Azure Document Intelligence custom and prebuilt-layout models, Qwen 2.5 / Qwen 0.6B local prompt engineering via Ollama/vLLM, YOLO parameter sweeps, and Gemini-3-Flash/Pro structured-output trials, and made the documented keep/abandon decisions.
- Integrated Arabic NLP (arabic_reshaper + python-bidi) and English translation (deep_translator GoogleTranslator) into the police parsers.
- Managed configuration and secrets (`.env` port 5057, police sub-service port 5055, Google service account `OCR_key.json`, service account `ocrqicuser`).
- Refactored duplicated Qwen prototypes (`qwen_langchain.py`, `qwen_openai.py`) into one `config_dict`-driven engine; archived superseded engines under `services/backup/`.
- Image pre-processing for VLM submission (1536x1536 thumbnail at `llm_engine.py:55`, JPEG quality 80 at `llm_engine.py:57`); system documentation with verified code line references.
- Disabled the archived Azure path rather than deleting it — `services/backup/azure_engine.py` (async `azure_kv_extract` against prebuilt-layout) remains an inactive archive and the AZURE engine is explicitly commented out at `routers/extract.py:20` (inferred).
- Assembled and retained the test-asset and artefact inventory: raster test PDFs and scanned police-report JPEGs under `skipped_images/` (`police_report_0..5.jpg`, 1183x1663 to 1191x126 px, produced as output of `multiple_extract`), the in-repo YOLO weights, `OCR_key.json` (referenced at `.env:14` and `ocr_engine.py:24`), an unreferenced `labels.json` (94-byte driving-license-front label dictionary), and `architecture_flow_bw.png` used by an earlier docx report (inferred).

**Estimated ownership level: Senior AI Engineer (inferred).**
Rationale: The candidate owned the full stack — a custom computer-vision detector (training authorship itself unconfirmed), a multi-provider VLM orchestration layer with two execution paradigms, four bespoke document parsers, Arabic NLP integration, and the API layer — and made architecture-level trade-offs (on-premise vLLM vs. cloud foundation models; single-pass VLM vs. OCR+regex; abandonment of Azure DI). The scope (nine functional endpoints, three model backends, 18+ schemas, two OCR engines) and the recorded experimental lineage indicate end-to-end design authority. It stops short of "Solution Architect / Technical Lead" because the document gives no evidence of team leadership, and production-readiness gaps remain: no automated tests, unpinned dependencies with version drift, and a documented critical key-mismatch defect on the UAE path.

## 3. End-to-End Workflow

**Path A — Identity document extraction (`/extract/{country}`):**
1. Client POSTs `multipart/form-data` field `images` (PDF/JPEG/PNG, one or more) with required query parameter `model_name` (enum: `LLAMA-LANGCHAIN`, `LLAMA-OPENAI`, `GEMINI-LANGCHAIN`, `GEMINI-OPENAI`, `QWEN3-LANGCHAIN`, `QWEN3-OPENAI`).
2. FastAPI router (`routers/extract.py`, prefix `/extract`, tag `OCR`) validates, parses model selection, and dispatches via `asyncio.gather`.
3. YOLOv8 Splitter (`services/yolo_splitter.py`): pypdfium2 renders pages at `scale=2` (`page.render(scale=2)`, line 45); YOLOv8 (`yolo-multipage-OCR-cls.pt`) runs with `CONFIDENCE_THRESHOLD = 0.5` (line 39), `IOU_THRESHOLD = 0.4` (line 40); detections are cropped. Non-`doc` classes — in practice police-report pages — are filtered out and written to `skipped_images/<class>_<idx>.jpg`; if nothing is detected, the full uncropped image is safely retained as class `doc`.
4. Image preparation (`llm_engine.py`): `img.thumbnail((1536, 1536))`, JPEG `quality=80`.
5. Stage 1: crop plus country-specific `DOCUMENT_ANCHORS` keyword rules go to the selected VLM to determine `doc_type` (e.g., Doha residency permits keyed on 'ID. Card', 'residency permit', 'employer').
6. Guard: `if not doc_type or doc_type not in OUTPUT_DICT.get(country, {})` → None (`llm_engine.py:162`).
7. Stage 2: a strict JSON schema is synthesized from `OUTPUT_DICT[country][doc_type]`; the VLM is queried with `strict=True`, `temperature=0`, `additionalProperties=False`.
8. Results aggregate into `[{"messageType": "success", "message": "S", "document_type": "<doc_type>", "response": {...}}]`; failures return HTTP 400 `'No cropped images found'` or `'No valid documents found'`.

**Path B — Police accident reports (`/sharjah`, `/abu_dhabi`, `/uae`, `/doha` on `pd_router`, tag `Police_docs`):**
1. Client POSTs `multipart/form-data` field `file` (PDF); `police_report_controller.py` checks file count and enforces `mime_type == 'application/pdf'` (else `Unsupported file type: <mime>`).
2. Jurisdiction parser: Saaed Abu Dhabi — PyMuPDF `fitz.open(file_path)` vector tables, no OCR; Dubai Police — PyMuPDF text blocks, `affected_party`/`liable_party` parsed on `ThreadPoolExecutor(max_workers=6)`; Rafid Sharjah — Wand/ImageMagick 300 DPI render, Google Cloud Vision OCR, custom 2D coordinate sorting (10 px line threshold), 5 workers; Doha — PyMuPDF block spans, table extraction, GCV OCR.
3. Arabic reshaping (`arabic_reshaper`, `bidi.algorithm.get_display`) and GoogleTranslator English translation of accident causes, driver remarks, property damages.
4. Validity gate: >3 populated values (Sharjah, Abu Dhabi, Dubai) or >0 (Doha); otherwise `{"message": "Irrelevant document to proceed/Document quality is low, Please try another...!", "messageType": "E"}` (plain or gzip-compressed).
5. Success: `{"message": "success", "messageType": "S", "response": {...}}`, with an authority-specific response shape:
   - Abu Dhabi (`/abu_dhabi`): `report_date`, `incident_number`, `vehicle_party[{driver_detail, insurance_detail, vehicle_detail}]`, `faulty_damaged_parts`, `claim_description`.
   - Sharjah (`/sharjah`): `accident_info`, `vehicle_party[]`, `injured_party[]`, `property_damage_party[]`, `claim_description`.
   - Dubai (`/uae` on `pd_router`, **not** `/extract/uae`): `reason_of_accident`, `incident_number`, `vehicle_party[]`, `important_notes[]`.
   - Doha (`/doha` on `pd_router`): `report_status`, `incident_date`, `incident_number`, `vehicle_party[]`, `injured_party[]`, `property_damage_party[]`, `guilty_charge`.

**Data flow:** PDF/image bytes → pypdfium2 page renders → YOLO boxes → cropped, downscaled JPEGs → VLM (image + anchor prompt) → `doc_type` → VLM (image + strict schema) → JSON array. Police: PDF → PyMuPDF blocks/tables or 300 DPI raster → GCV OCR with coordinates → sorted lines → parsed dict → reshaped/translated → JSON object.

**Integrations and dependencies:** YOLOv8, pypdfium2, PyMuPDF, OpenCV, Pillow, Wand; Google Cloud Vision (service account `OCR_key.json`); AsyncOpenAI against a local vLLM host (172.20.132.226) serving Qwen3-VL-4B-Instruct; Google GenAI (Gemini-3-Flash-Preview); Azure AI Services (Dev-Llama-4-Scout-17B-16E-Instruct); LangChain Core/OpenAI/Google GenAI; arabic-reshaper, python-bidi, deep_translator.

```
Client -> FastAPI (/extract/{country}?model_name=...) -> pypdfium2 render 2.0x -> YOLOv8 (conf 0.5 / IoU 0.4) crop -> thumbnail 1536 / JPEG q80 -> VLM Stage 1 (DOCUMENT_ANCHORS) -> VLM Stage 2 (OUTPUT_DICT strict schema, T=0) -> JSON array
Client -> FastAPI (/sharjah | /abu_dhabi | /uae | /doha) -> police_report_controller -> PyMuPDF tables/blocks | Wand 300 DPI + GCV OCR + coordinate sort -> arabic_reshaper/bidi + GoogleTranslator -> validity gate (>3 / >0) -> JSON
```

**Architecture Summary:** A decoupled FastAPI service (port 5057) with a police-docs sub-service (port 5055). Identity documents pass through YOLOv8 pre-segmentation before a provider-agnostic VLM engine performs classification and schema-constrained extraction; police reports bypass the VLM and go to hand-built PyMuPDF/GCV parsers. Configuration-driven schemas (`DOCUMENT_ANCHORS`, `OUTPUT_DICT`) allow new document types without engine changes, and `model_name` switches between on-premise and cloud backends per request.

## 4. Technologies and Tools Used

| Category | Technologies (with versions where the document states them) |
|---|---|
| Programming languages | Python 3.11.15 (Conda environment `plp`) |
| Frameworks | FastAPI (unpinned; 0.141.1 installed); Uvicorn (unpinned; 0.52.0 installed); LangChain Core (unpinned; 1.5.4 installed); LangChain OpenAI (unpinned; 1.5.0 installed); LangChain Google GenAI (unpinned; installed) |
| Libraries | Ultralytics YOLOv8 (`==8.4.14` declared; 8.4.113 installed); OpenCV opencv-python 4.11.0.86; Pillow 12.3.0; PyMuPDF (fitz) 1.28.0; pypdfium2; OpenAI Python SDK 2.52.0 (AsyncOpenAI); arabic-reshaper; python-bidi 0.6.11; deep_translator (GoogleTranslator); Wand (ImageMagick binding); asyncio; concurrent.futures ThreadPoolExecutor; Pydantic (referenced for strict schema) |
| AI/ML models | YOLOv8 custom weights `yolo-multipage-OCR-cls.pt` (54.5 MB) and `best.pt` (45.2 MB); Qwen3-VL-4B-Instruct (local vLLM); Gemini-3-Flash-Preview (Google GenAI); Dev-Llama-4-Scout-17B-16E-Instruct (Azure AI Services). Experimental only: Gemini-3-Pro-Preview, Qwen 2.5, Qwen 0.6B, Azure Document Intelligence custom models (`anoud_ocr_custom_uae_vb`, `anoud_ocr_custom_uae_vf`, `prebuild-anoud-ocr-ai`) and prebuilt-layout |
| OCR tools | Google Cloud Vision API (google-cloud-vision); VLM-based OCR via Qwen3-VL / Gemini / Llama; PyMuPDF vector text extraction (no OCR) for Saaed Abu Dhabi and Dubai Police |
| Databases | Not stated in documentation |
| Cloud platforms | Google Cloud (Vision API, GenAI/Gemini); Azure AI Services (Llama-4-Scout); on-premise GPU vLLM host at 172.20.132.226 |
| APIs | FastAPI REST endpoints (GET /, POST /extract/uae, /extract/oman, /extract/kuwait, /extract/doha, /extract/passport, POST /sharjah, /abu_dhabi, /uae, /doha); auto-generated Swagger UI (/docs), ReDoc (/redoc), OpenAPI JSON (/openapi.json); Google Cloud Vision API; Google GenAI API; Azure AI Services API; OpenAI-compatible vLLM API; Google Translate via deep_translator |
| DevOps tools | Not stated in documentation (no CI/CD, containerization, or test framework present; automated testing "Not implemented in the project") |
| Version control | Not stated in documentation (repository path referenced, but no VCS tooling named) |
| Deployment tools | Uvicorn ASGI server; `.env` configuration (port 5057; police sub-service port 5055); Conda environment `plp`. No container or orchestration tooling stated |
| Document processing tools | pypdfium2 (page rendering at 2.0x); PyMuPDF (block spans, table extraction); Wand/ImageMagick (300 DPI rendering, `sharjah_police_report_img.py:731`; no version stated); Pillow/OpenCV (cropping, thumbnailing, JPEG compression); custom `coordinates_sort.py` (2D coordinate sequence sorting, 10 px line threshold at lines 33 and 210) |
| Automation tools | Python asyncio (`asyncio.gather`); ThreadPoolExecutor (5 workers Sharjah `sharjah_police_report_img.py:745`, 6 workers Dubai `dubai_police_report.py:297,312`); Jupyter notebook `debug.ipynb` (214 cells) for experimentation |

Documentation notes on tooling gaps, stated verbatim in the source: no automated test code, test scripts, or test frameworks (pytest/unittest) exist — "Automated testing status: Not implemented in the project"; no spreadsheet assets (.xlsx/.xls/.csv) exist in the project folder; and no specific Swagger UI library version string is declared anywhere in configuration or application code (FastAPI's auto-generated UI is used as-is).

## 5. Technical Skills Demonstrated

- **Software Engineering:** Modular package layout (`app.py`, `routers/`, `services/`, `services/police_docs/`, `config/`); refactoring duplicated Qwen prototypes into a single `config_dict`-driven engine; archiving superseded code under `services/backup/`; environment-based configuration; structured error envelopes and gzip-compressed error responses.
- **AI Engineering:** Designed a multi-backend VLM dispatcher supporting AsyncOpenAI and LangChain Core paradigms; per-request model selection across local and cloud foundation models; two-stage classification→extraction pipeline with dynamic strict JSON schema synthesis; constrained decoding via `strict=True`, `temperature=0`, `additionalProperties=False`.
- **Machine Learning:** Trained/used a custom YOLOv8 multi-document detection model; empirical threshold tuning (iou=0.9 vs. conf=0.5/iou=0.4) with recorded outcomes; model checkpoint management (`yolo-multipage-OCR-cls.pt`, `best.pt`).
- **NLP:** Bidirectional Arabic text correction with arabic_reshaper and `bidi.algorithm.get_display`; automated Arabic→English translation of accident narratives via deep_translator; keyword-anchor document classification (and its heuristic predecessor `detect_document_sides` in `services/detector.py`).
- **Computer Vision:** Object detection and cropping on rendered PDF pages; class-based routing of detections (`doc` vs. non-doc); image downscaling/compression for VLM input; 300 DPI rasterization for OCR; spatial line reconstruction from OCR coordinates (10 px threshold).
- **Data Engineering:** Configuration-driven schema catalogs (`DOCUMENT_ANCHORS`, `OUTPUT_DICT`) covering 18+ document types; structured JSON output contracts per jurisdiction; multi-format ingestion (PDF, JPEG, PNG) with rendering normalization.
- **Cloud:** Google Cloud Vision API with service-account authentication (`OCR_key.json`); Google GenAI (Gemini); Azure AI Services (Llama-4-Scout); hybrid on-premise/cloud inference topology for data sovereignty.
- **MLOps:** Limited — experiment tracking is notebook-based (debug.ipynb); model weights stored in-repo; declared vs. installed version drift documented. No CI/CD, tests, or monitoring demonstrated.
- **API Development:** FastAPI with APIRouter prefixes/tags, multipart uploads, required enum query parameters, auto-generated OpenAPI/Swagger/ReDoc, HTTP 400 error semantics, health-check route.
- **Prompt Engineering:** Country-specific anchor keyword rules for Stage 1 typing; schema-as-prompt Stage 2 extraction; migration from OCR-text+regex prompting to direct image-grounded single-pass VLM prompting; deterministic decoding at temperature 0.
- **Data Extraction / Document AI:** Vector-PDF table and block-span extraction (PyMuPDF `fitz`) chosen over OCR wherever the PDF carries a text layer; raster fallback via Wand 300 DPI + Google Cloud Vision only where the source is a scan (Rafid Sharjah); per-jurisdiction field contracts including camelCase/snake_case and single- vs. front/back-key variants.
- **System Design:** Decoupled pipeline stages; pluggable model engine decoupling business logic from vendors; separate controller path for police reports; asynchronous fan-out (`asyncio.gather`) and thread pools for CPU/IO-bound parsing; fallback behaviors (uncropped `doc` retention, skipped_images diversion).

## 6. Detailed Technical Contributions

**Features implemented**
- Automated multi-document detection and splitting with fallback retention of the full image as class `doc`.
- Two-stage anchor matching and dynamic strict schema decoding for 18+ document types across UAE, Qatar/Doha, Oman, Kuwait, and passports for India, Pakistan, Oman, Philippines, Egypt, Bangladesh.
- Six selectable execution modes via `model_name`: `LLAMA-LANGCHAIN`, `LLAMA-OPENAI`, `GEMINI-LANGCHAIN`, `GEMINI-OPENAI`, `QWEN3-LANGCHAIN`, `QWEN3-OPENAI`.
- Four police accident report parsers (Saaed Abu Dhabi, Dubai Police, Rafid Sharjah, Doha Traffic Police) with bilingual reshaping/translation.
- Health check `GET /` returning `{"status": "Running"}` (declared `async def root()` in `app.py:18-19`; no error responses declared).
- Configuration-level schema exceptions handled per jurisdiction: Kuwait vehicle registration keyed as a single `vehicle_registration` rather than front/back pairs; Doha residency permits anchored on the keywords `'ID. Card'`, `'residency permit'`, `'employer'`; Philippine passports keyed `'fillipino'` consistently across `document_anchors.py` and `output_keys.py`.
- Bilingual output: arabic_reshaper + `bidi.algorithm.get_display` correction plus GoogleTranslator English translation applied specifically to accident causes, driver remarks and property damages.

**Models used**
- YOLOv8 `yolo-multipage-OCR-cls.pt` (production); `best.pt` (referenced in debug.ipynb).
- Qwen3-VL-4B-Instruct on a private vLLM host (172.20.132.226); Gemini-3-Flash-Preview via Google GenAI; Dev-Llama-4-Scout-17B-16E-Instruct via Azure AI Services.

**Pipelines built**
- Identity pipeline: `page.render(scale=2)` → YOLO (conf 0.5, IoU 0.4) → crop → thumbnail 1536x1536 / JPEG q80 → Stage 1 anchor classification → `OUTPUT_DICT` guard → Stage 2 strict extraction → array envelope.
- Police pipelines: four per-authority parsers (Saaed, Dubai, Sharjah, Doha) as detailed in Section 3 Path B, sharing the controller, Arabic reshaping/translation, and validity-gate stages.

**APIs integrated**
- `router = APIRouter(prefix='/extract', tags=['OCR'])` and `pd_router = APIRouter(tags=['Police_docs'])` mounted in `app.py`; unmounted `routers/uae.py` (`POST /uae/`) with an unresolvable `services.qwen_langchain` import.
- Google Cloud Vision (`ocr_engine.py:24`, key from `.env:14`); Google GenAI; Azure AI Services; OpenAI-compatible vLLM; Google Translate via deep_translator.

**Data extraction methods**
- VLM image-grounded extraction into per-type schema fields, e.g., Oman `driving_license_front`: `license_number`, `issued_place_english`, `name_english`, `first_issued`, `expiry_date`; Doha `residency_permit_front`: `idNo`, `dob`, `expiry`, `nationalityEng`, `nationalityAra`, `nameEng`, `name_ar`; passports: `type`, `code`, `passport_number`, `given_name`, `fullname`, `date_of_birth`, `date_of_expiry`.
- PyMuPDF table/block extraction for vector PDFs; GCV OCR plus 2D coordinate sorting (10 px threshold) to reconstruct reading order on scans.

**Validation logic**
- Identity path: `doc_type` must exist in `OUTPUT_DICT[country]`, else the crop yields None; HTTP 400 when no crops or no valid documents.
- Police path: exactly one PDF file (`mime_type == 'application/pdf'`); document validity requires >3 non-empty response values (Sharjah line 770, Abu Dhabi line 520, Dubai line 343) or >0 (Doha line 734).
- Schema-level validation delegated to provider strict mode (`strict=True`, `additionalProperties=False`).

**Automation workflows**
- `asyncio.gather` fan-out over uploaded files/crops; ThreadPoolExecutor for concurrent party parsing; automatic diversion of non-document detections to `skipped_images/`.

**Optimization techniques**
- Single-pass VLM extraction replacing sequential OCR+LLM+regex (documented as high-latency); image downscaling to 1536x1536 and JPEG q80 to bound VLM payload; deterministic decoding at temperature 0; unified engine to remove code duplication.

**Performance improvements**
- Qualitative only: the source records that the OCR+regex prototype "showed high latency from sequential OCR + LLM passes" and Azure DI "exhibited variable latency," motivating the current design. No measured latency or accuracy figures are given.

**Experimentation lineage (debug.ipynb, 214 cells, with recorded keep/abandon verdicts)**
- Cells 1-11: Azure Document Intelligence custom-trained models `anoud_ocr_custom_uae_vb`, `anoud_ocr_custom_uae_vf`, `prebuild-anoud-ocr-ai` — extracted raw key-value pairs successfully but showed variable latency and high cloud API coupling. Abandoned, replaced by open VLM and local inference.
- Cells 27-46: Azure DI `prebuilt-layout` with `keyValuePairs` — produced text blocks and coordinate bounding boxes but struggled with multi-document scans. Abandoned; moved to `services/backup/azure_engine.py`.
- Cells 40-45: Qwen 2.5 and Qwen 0.6B local Ollama/vLLM prompt engineering — demonstrated structured JSON generation on local CPU/GPU. Evolved into the production Qwen3-VL-4B-Instruct engine.
- Cells 116-138: YOLOv8 splitting parameter tuning (iou=0.9 vs conf=0.5/iou=0.4) — stored outputs show successful bounding-box cropping of multiple ID cards from single high-resolution scans. Carried into production.
- Cells 191-214: Gemini-3-Flash-Preview and Gemini-3-Pro-Preview structured JSON schema inference via the Google GenAI SDK — valid JSON with strict field-level schema compliance. Carried into production as `GEMINI-OPENAI` / `GEMINI-LANGCHAIN`.
- Archived/legacy code: `services/backup/azure_engine.py` (async `azure_kv_extract`; AZURE engine commented out at `routers/extract.py:20`); `services/backup/gemini_langchain.py` (OCR text → regex post-processors, superseded by single-pass VLM); `qwen_langchain.py` + `qwen_openai.py` (functional but duplicated, unified into `llm_engine.py` via `config_dict`); `services/detector.py` `detect_document_sides` (keyword-frequency heuristic classification, now imported only by the backup file at `gemini_langchain.py:135`, superseded by VLM prompt classification at `llm_engine.py:144-158`).

## 7. Challenges and Solutions

1. **Composite scans with multiple ID cards per page (technical).** Azure DI prebuilt-layout "struggled with multi-document scans." Solution: custom YOLOv8 detector over 2.0x-rendered pages at conf 0.5 / IoU 0.4 (chosen over iou=0.9), with uncropped fallback. Alternatives: Azure DI layout (documented, abandoned); OpenCV contour-based segmentation (analyst suggestion).
2. **Hallucination and schema drift in LLM output (technical/business).** Solution: dynamic strict JSON schema from `OUTPUT_DICT` with `strict=True`, `temperature=0`, `additionalProperties=False`. Alternatives: OCR text + regex post-processors (documented in `services/backup/gemini_langchain.py`, superseded); Pydantic post-validation with retries (analyst suggestion).
3. **Vendor lock-in and data sovereignty (business).** Solution: pluggable engine with on-premise Qwen3-VL-4B-Instruct on a private vLLM host plus cloud Gemini and Azure Llama. Alternative: Azure DI custom models (`anoud_ocr_custom_uae_vb/vf`) — documented, abandoned for "high cloud API coupling."
4. **Heterogeneous police report formats (technical).** Solution: one parser per authority — PyMuPDF tables (Saaed), blocks + threads (Dubai), Wand 300 DPI + GCV + coordinate sort (Rafid Sharjah), blocks/tables + GCV (Doha). Alternative: route police reports through the VLM strict-schema engine (analyst suggestion; needs evaluation on dense tables).
5. **Reversed/disconnected Arabic in PDFs (technical).** Solution: arabic_reshaper + `bidi.algorithm.get_display`, then GoogleTranslator for narratives. Alternative: let a VLM read Arabic from the rendered page (analyst suggestion).
6. **Reading order from OCR coordinates (technical).** Solution: `coordinates_sort.py` grouping tokens into lines with a 10 px threshold. Alternative: GCV's block/paragraph hierarchy (analyst suggestion).
7. **Latency of sequential OCR→LLM passes (technical).** Solution: single-pass multimodal extraction, `asyncio.gather`, thread pools (5/6 workers). Alternatives: batching several crops into one VLM request, or caching Stage 1 results per page (analyst suggestions).
8. **Documented defects and limitations (technical):** (a) UAE anchor keys (`license_front`, `license_back`, `residence_front`, `residence_back`, `vehicle_front`, `vehicle_back` in `config/document_anchors.py`) do not match `OUTPUT_DICT` keys (`driving_license_front`, `driving_license_back`, `residency_permit_front/back`, `vehicle_registration_front/back` in `config/output_keys.py`), so the guard at `services/llm_engine.py:162` always returns None and `/extract/uae` always returns HTTP 400 'No valid documents found' — the source notes `config/document_anchors - Copy.py` held matching keys before being shortened, i.e. a regression introduced by an edit; (b) `police_report_controller.py:49-52` uses `if len(file_img) > 1` with a misleading "Please attach at least one file/document to proceed...!" message, restricting input to exactly one file, and line 65 accepts only `mime_type == 'application/pdf'` (all other types rejected as `Unsupported file type: <mime>`); (c) inconsistent validity gates — >3 non-empty response values for Sharjah (line 770), Abu Dhabi (line 520) and Dubai (line 343) versus >0 for Doha (`doha_police_report.py:734`); (d) Philippine passports keyed `fillipino`; (e) unmounted `routers/uae.py` (`APIRouter(prefix='/uae', tags=['UAE'])`, `POST /uae/`) never imported into `app.py` and importing a non-existent `services.qwen_langchain` (the file actually lives at `services/backup/qwen_langchain.py`); (f) unpinned dependencies with drift (ultralytics `==8.4.14` declared vs. 8.4.113 installed; FastAPI, Uvicorn, PyMuPDF, LangChain packages, google-cloud-vision and deep_translator all unpinned in `Requirements.txt`); (g) no automated tests, scripts or frameworks of any kind; (h) Google service-account key `OCR_key.json` stored in the project directory and referenced from `.env:14` and `ocr_engine.py:24` (analyst observation: secrets-management risk); (i) unreferenced assets left in the tree (`labels.json`, the four `skipped_images/original_*.pdf` test PDFs).
9. **Security/authentication posture (technical).** The only authentication documented anywhere in the system is the Google Cloud service-account credential (`OCR_key.json`, service account `ocrqicuser`) used for the Vision API. No API-level authentication, authorization, rate limiting, or transport security is described for the FastAPI endpoints — treat this as *not stated in documentation* rather than as an implemented control; adding gateway-level auth would be the first hardening step (analyst suggestion).

## 8. Impact Analysis

The document provides **no quantified accuracy, throughput, latency, or cost-savings figures**. Impact below is qualitative, as stated in the source.

- **Business impact:** Enables straight-through processing for policy onboarding and motor claims registration; centralizes document intelligence for the GCC jurisdictions (UAE, Qatar, Oman, Kuwait — the source says "five GCC nations" but lists four) and six passport nationalities in one API.
- **Productivity improvements:** Eliminates manual pre-processing of composite scans; automates extraction of liable/affected parties, license details, damaged parts, and accident causes from four police authorities.
- **Accuracy improvements:** Strict schema enforcement described as "zero-hallucination" and "guaranteeing schema compliance"; no measured accuracy reported.
- **Cost savings:** Not stated. Provider agility and on-premise inference are positioned as strategic, not costed.
- **Time savings:** Source states extraction "in seconds" (qualitative); prototypes were dropped for "high latency," implying the production design is faster, but no numbers are given.
- **User benefits:** Claims and underwriting staff receive bilingual, structured accident and identity data; integrators get a uniform JSON envelope, Swagger docs, and per-request model choice.
- **The only numbers the source actually states are configuration constants and artefact sizes, not outcomes:** conf 0.5 / IoU 0.4, render scale 2.0x, 1536x1536 thumbnail, JPEG q80, temperature 0, 300 DPI Wand rendering, 10 px coordinate-sort line threshold, ThreadPoolExecutor 5/6 workers, validity gates >3 (>0 Doha), ports 5057 and 5055, model weights 54.5 MB and 45.2 MB, 214 notebook cells, 18+ document types, 6 passport nationalities, 6 model modes, 9 functional endpoints. None of these is an accuracy, throughput, latency or cost measurement, and none should be presented as one.

## 9. Interview Discussion Points

- Why two VLM stages rather than one prompt? Stage 1 resolves `doc_type` so Stage 2 can bind an exact `OUTPUT_DICT` schema; explain the `strict=True`/`additionalProperties=False` guarantee and the membership guard.
- How were YOLO thresholds chosen? Cite the debug.ipynb comparison (iou=0.9 vs. conf=0.5/iou=0.4) and the uncropped `doc` fallback.
- Why abandon Azure Document Intelligence? Variable latency, cloud coupling, failure on multi-document scans — and how local Qwen3-VL on vLLM addresses sovereignty.
- Why support both AsyncOpenAI and LangChain? Unifying `qwen_langchain.py`/`qwen_openai.py` via `config_dict`, and the cost of maintaining two paths.
- How would you fix and prevent the UAE key mismatch? Config-consistency tests asserting anchor keys ⊆ `OUTPUT_DICT` keys per country; acknowledge the missing test suite.
- Why bespoke police parsers instead of the VLM? Vector PDFs allow lossless PyMuPDF table extraction; Rafid Sharjah scans need Wand 300 DPI + GCV + coordinate sorting.
- Arabic handling: what breaks without arabic_reshaper/bidi, and why translate only narratives.
- Concurrency: `asyncio.gather` vs. `ThreadPoolExecutor(max_workers=5/6)`; which stages are IO- vs. CPU-bound.
- Preprocessing: 2.0x render for detection vs. 1536x1536/JPEG q80 for VLM payload — resolution against token/latency cost.
- Validity gates: why >3 non-empty values, the Doha >0 inconsistency, and what a principled quality check looks like.
- Production hardening: unpinned requirements, ultralytics drift, secrets in-repo; what metrics you would instrument next (field-level accuracy per doc type, per-model latency, fallback rate).
- Why keep abandoned engines in-tree rather than delete them? `services/backup/azure_engine.py` plus the commented-out AZURE branch at `routers/extract.py:20` preserve a reversible decision — and the cost is dead code and confusing imports (`routers/uae.py`).
- Why schema shapes differ per jurisdiction (Kuwait's single `vehicle_registration` key, Doha's camelCase `idNo`/`nameEng` alongside snake_case elsewhere): fidelity to the physical document and to consuming systems, versus the maintenance cost of an inconsistent contract — and how you would normalise it behind a stable public schema.
- The police controller's one-file, PDF-only restriction: why the `len(file_img) > 1` guard is inverted relative to its message, what a correct guard looks like, and whether multi-file batch police intake is worth supporting.
- Where the two-stage design can fail closed: an anchor keyword miss produces no `doc_type`, the crop is silently dropped, and the endpoint reports a generic 400 — how you would surface per-crop diagnostics instead of an all-or-nothing error.
- Security: only the GCP service account is authenticated; the API itself has no documented auth — what you would put in front of an endpoint receiving passports and civil IDs.

## 10. Architecture Explanation Points

Draw two lanes. Lane 1: **Client → FastAPI (`/extract/{country}?model_name=`) → YOLOv8 Splitter → Unified VLM Engine → JSON array**. pypdfium2 renders at 2.0x; YOLOv8 (`yolo-multipage-OCR-cls.pt`, conf 0.5/IoU 0.4) crops each sub-document, shunts non-doc classes to `skipped_images/`, and falls back to the whole image on empty detection. The engine downsizes crops (1536x1536, JPEG q80), runs Stage 1 anchor classification and Stage 2 strict-schema extraction at temperature 0, and fans out with `asyncio.gather`. Lane 2: **Client → `/sharjah | /abu_dhabi | /uae | /doha` → police controller → PyMuPDF or Wand+GCV parsers → Arabic reshaping/translation → validity gate → JSON**.

Key decisions: segment before extracting so each VLM call sees one document; configuration-driven schemas to add jurisdictions without code; pluggable backends (on-premise Qwen3-VL for sovereignty, Gemini/Llama for cloud) selectable per request; bespoke parsers where vector PDF text is lossless.

Component boundaries as the source names them: **FastAPI Router** (`app.py`, `routers/extract.py`) owns request validation, `model_name` query parsing and async dispatch via `asyncio.gather`; **YOLOv8 Splitter** (`services/yolo_splitter.py`) owns pypdfium2 rendering, detection, cropping and non-doc diversion; **Multimodal VLM** (`services/llm_engine.py`) owns two-pass classification against `DOCUMENT_ANCHORS` and strict-schema extraction against `OUTPUT_DICT` across the three backends. Configuration (`config/document_anchors.py`, `config/output_keys.py`) is the only place a new jurisdiction is added. The police path shares none of this: `services/police_docs/police_report_controller.py` fronts four independent parser modules and its own `.env` (port 5055) beside the main service (port 5057) — the runtime relationship between the two ports is not explained in the source.

Trade-offs: two VLM calls per crop buy schema certainty at latency cost; two execution paradigms (AsyncOpenAI and LangChain) double the surface to maintain in exchange for vendor portability; bespoke parsers are precise but brittle to layout changes; strict mode depends on provider support; the configuration-driven schema catalog makes new document types cheap but has no mechanism enforcing that anchor keys and output keys stay in sync — exactly the failure that disabled `/extract/uae`; and the 2.0x render for detection versus the 1536x1536 / q80 downscale for inference is a deliberate two-resolution pipeline (detect on detail, infer on payload budget).

Next: fix the UAE key mismatch and add config-consistency and endpoint tests; unify validity gates; pin dependencies and containerize; move `OCR_key.json` to a secrets manager; add per-model accuracy/latency telemetry; evaluate routing police scans through the VLM engine.
