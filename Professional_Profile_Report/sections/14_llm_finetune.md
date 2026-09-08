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
