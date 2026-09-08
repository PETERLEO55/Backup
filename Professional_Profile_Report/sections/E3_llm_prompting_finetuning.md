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
