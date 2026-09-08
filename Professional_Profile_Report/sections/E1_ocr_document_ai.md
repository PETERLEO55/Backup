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
