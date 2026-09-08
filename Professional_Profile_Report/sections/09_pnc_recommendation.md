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
