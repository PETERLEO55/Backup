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
