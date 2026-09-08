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
