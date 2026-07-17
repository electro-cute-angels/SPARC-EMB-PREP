Task: Fix the R² reporting and settle the preprocessing question — no retraining
The diagnostic established that the alarming DINOv3 self-R² of −3.14 is largely a reporting artifact: one near-zero-variance DINOv3 dimension (var ≈ 6e-10) gets a per-dim R² of ≈ −3892, and because the repo averages per-dimension R² with uniform weight, that single dimension drags the whole score negative. Computed as flattened/variance-weighted R², DINOv3 self-reconstruction is actually ~0.80 and V-JEPA ~0.95 — both healthy. This task does two independent things: (1) fix the R² reporting on the EXISTING checkpoint, and (2) definitively answer whether our preprocessing matches SPARC's intended preprocessing. No retraining, no method changes (no new n_latents, cross_loss_coef, or local TopK) in this task.
Part 1 — Corrected R² reporting (on the existing checkpoint, no retraining)
Using the existing trained model and its analysis_cache_val.h5, recompute and save a corrected R² report that presents multiple robust variants side by side, so we see the true reconstruction quality:

Flattened / variance-weighted R² — computed over all dimensions jointly, weighted by each target dimension's variance (equivalently, r2_score(..., multioutput='variance_weighted') or a global-variance formulation). This is the number that reflects actual reconstruction quality.
Per-dim R² with a variance floor — compute per-dimension R² but EXCLUDE dimensions whose target variance is below a threshold (report at a couple of thresholds, e.g. 1e-6 and 1e-8), then uniform-average the survivors. Report how many dimensions were excluded per stream.
The original uniform-per-dim R² (unchanged) for reference, so we can see the delta.
Produce the full source→target matrix (both streams, self and cross) for each of the three variants. Save as r2_matrix_corrected.csv and a short R2_CORRECTED.md summarizing what changed and confirming whether, under robust metrics, both streams reconstruct healthily.
Also: identify and report the pathological low-variance DINOv3 dimension(s) — how many dims sit below each threshold, in raw space vs. after the pipeline's normalization, so we understand where the near-constant dimension comes from (this feeds Part 2).

Part 2 — Settle the preprocessing question (read the repo, don't assume)
The diagnostic found our pipeline applies per-sample L2 normalization (in sparc/datasets.py __getitem__: x = x / (||x|| + 1e-8)) plus the model's learnable per-stream pre_bias, but no fixed per-dimension standardization (zero-mean/unit-variance). We need to know whether that is SPARC's intended preprocessing or whether our .pt→.h5 adapter skipped a standardization step the repo normally performs. Determine this from the actual code, not assumption:

Read the repo's own feature-extraction scripts (sparc/feature_extract/extract_coco.py, extract_open_images.py) and its dataset/dataloader (sparc/datasets.py) and any preprocessing/normalization utilities. Report exactly what transformation the repo applies to features between extraction and entering the model, in order.
Specifically answer: Is per-sample L2 normalization the repo's genuine, intended preprocessing for features entering SPARC — i.e., would a user running the repo's own pipeline on COCO/Open-Images get L2-normalized features and nothing else? Or does the repo apply per-dimension standardization (or store already-standardized features) somewhere that our adapter bypassed?
Quote the relevant lines/functions so the answer is grounded in the code.
If the repo's HDF5 files it ships / its extraction scripts produce features that are standardized before the L2 step (or instead of it), then our adapter diverged and we should match the repo. If the repo genuinely does L2-only, then our pipeline is method-faithful and the near-constant dimension is just a quirk to handle in the metric (Part 1), not a preprocessing bug.
Also note how the repo's convention would interact with a 768-dim CLIP stream added later at a different scale — i.e., is L2-only normalization robust to streams of differing dimensionality/scale, or does the paper/repo rely on standardization to make streams comparable? This matters because CLIP and ViT join soon.

Deliverable
A single PREPROCESSING_AND_R2.md containing:

The corrected R² matrix (three variants) and a one-line verdict: under robust metrics, do both streams reconstruct healthily? (Expected: yes.)
Where the pathological DINOv3 dimension comes from (raw vs. post-normalization).
The definitive answer to "is L2-only the repo's intended preprocessing, or did our adapter skip standardization?" — grounded in quoted code.
A clear recommendation: either (a) "our preprocessing matches the repo, no change needed, the −3.14 was purely a metric artifact," or (b) "our adapter diverged from the repo; here is the standardization step we should add to match, and here's how it would change things" — but do NOT implement the change yet in this task.

Do NOT do

Don't retrain. Don't change n_latents / k / cross_loss_coef / topk_type.
Don't modify the preprocessing yet — just diagnose and recommend.
Don't overwrite the existing run's outputs; put new files in a clearly separate diagnostic folder.

Report back with PREPROCESSING_AND_R2.md and your recommendation before we decide whether any rerun is warranted.

