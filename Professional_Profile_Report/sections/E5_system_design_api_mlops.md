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
