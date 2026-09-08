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
