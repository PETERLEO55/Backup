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
