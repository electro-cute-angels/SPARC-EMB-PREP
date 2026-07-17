Task: Multi-model embedding + dense-token extraction for SPARC
Build a robust, resumable feature-extraction system that runs four frozen, pre-downloaded vision encoders over an image corpus and caches, per model, (a) one global vector per image (for SPARC training) and (b) the full dense/patch token grid per image (for concept heatmaps later). Correctness and strict cross-model alignment matter far more than speed. This is extraction only — do not train anything, do not touch the SAE/SPARC step.
Checkpoints — load LOCALLY, do not re-download
All four checkpoints are already downloaded under checkpoints/ (see checkpoints/DOWNLOAD_REPORT.md for exact identifiers, tags, and filenames):

checkpoints/dinov3_vitl16/ — DINOv3 ViT-L/16 (facebook/dinov3-vitl16-pretrain-lvd1689m)
checkpoints/vjepa21_vitl_384/ — V-JEPA 2.1 ViT-L/16 @384 (Meta official .pt or community HF port — check the report)
checkpoints/vit_large_supervised/ — ViT-L supervised (vit_large_patch16_224.augreg_in21k_ft_in1k, timm)
checkpoints/clip_vitl14/ — CLIP ViT-L/14 (OpenCLIP or OpenAI — check the report for the exact tag)

Read DOWNLOAD_REPORT.md FIRST to get exact tags/paths. Load each model from its local files only; the entire pipeline must run offline.
Input corpus

--corpus_dir points to a folder containing subfolder corpus/ with ~18,000 images, mostly ~224–225px but assume heterogeneity: varying sizes, non-square, greyscale, CMYK, alpha channels, and occasional corrupt/unreadable files. Convert everything to RGB.
No labels, no captions, no class subfolders.

CRITICAL: Cross-model row alignment (the single most important requirement)
SPARC requires that row i in every model's output corresponds to the exact same source image. If row i is a different image across models, SPARC will train without error but learn meaningless correspondences — a silent, catastrophic failure (plausible loss curves, garbage results). Alignment must be guaranteed by construction AND explicitly verified.
Guarantee by construction:

Pre-validate the entire corpus before any extraction. Loop over all images once, attempt to open + convert each to RGB, and build a canonical ordered manifest (sorted by filename, deterministic) containing ONLY images that successfully load. Persist this manifest to disk. It is the single source of truth for ordering.
Every model, in every phase, iterates this exact persisted manifest — never re-globs the directory independently, never shuffles (shuffle=False).
Because skips are resolved up front during pre-validation, no model skips a file mid-run. Row counts and row→filename mappings are therefore identical across all four models by construction. (If, rarely, a file fails for one specific model during extraction, drop it from ALL models' outputs to preserve alignment, and log it prominently — but pre-validation should make this nearly impossible.)
Store the canonical row_idx → filename manifest (and a hash of it) inside each model's output folder for per-model provenance.

Verify explicitly (must PASS before any output is trusted):

Assert all four models' global-vector tensors have identical N. Fail loudly with the discrepancy if not.
Assert each model's stored manifest is byte-for-byte identical to the canonical one (compare hashes). Fail loudly on mismatch.
Semantic spot-check: pick ~10 random row indices; confirm all four models' outputs at each row trace to the same filename; log those filenames side-by-side for me to eyeball.
Provide a standalone verify_alignment.py I can run anytime that re-checks all of the above (identical N, identical manifests, random-row filename agreement) and prints a clear PASS/FAIL.
Phase 2 (dense tokens) must run verify_alignment.py against Phase 1's manifest before writing anything, so dense tokens cannot drift out of alignment with the global vectors.
If alignment cannot be verified, STOP and report — never emit possibly-misaligned outputs.

Global vector definition — PER MODEL (differs by model; get each right)

DINOv3 — global = CLS token (1024-dim). Exclude the 4 register tokens and patch tokens.
V-JEPA 2.1 — NO CLS token exists. Global = mean-pool over all output tokens (1024-dim); also save max-pool as an alternative file. Use the model's image tokenizer / image mode — feed a genuine still image, do NOT fake a video by duplicating frames.
ViT (supervised) — global = CLS token (1024-dim).
CLIP-image — primary global = post-projection encode_image output (768-dim; the contrastive, language-aligned embedding — matches the reference SPARC paper's use of CLIP). ALSO save the pre-projection CLS token (1024-dim) as a secondary file. Primary = post-projection.

SPARC handles differing input dimensions per stream, so CLIP's 768 vs the others' 1024 is fine — do NOT pad or project to force uniformity.
Dense/patch token definition — PER MODEL (for heatmaps)
Save the full spatial token grid per image, per model:

DINOv3 — 196 patch tokens @224 (patch-16 → 14×14), 1024-dim. Exclude CLS/registers from the grid. Record grid shape.
V-JEPA 2.1 — full spatial token grid from the image tokenizer. This is the riskiest part: tokens derive from a spatiotemporal (tubelet) scheme even in image mode. Empirically determine the token count and how it maps to a 2D spatial grid for a single image, verify on a test image, and document the reshape logic in a code comment + grid_shape.json. If the mapping is ambiguous, STOP and flag it for me.
ViT — 196 patch tokens @224 (14×14), 1024-dim.
CLIP — patch-14 → 256 patch tokens @224 (16×16), 1024-dim (transformer patch outputs, pre-projection — projection applies only to the pooled vector).

Grids have different spatial resolutions across models (14×14 vs 16×16, V-JEPA different again). This is expected and fine — each heatmap reshapes to its own grid; they're never compared patch-to-patch. Just record each grid shape accurately.
Preprocessing — per model, no silent uniformity
Follow each model's own standard preprocessing and log it to <model>/preprocessing.json (resize, crop, interpolation, normalization mean/std, final input resolution):

DINOv3 / ViT / CLIP: 224px per their standard transforms.
V-JEPA 2.1: its native 384px (upscale the ~224 input for V-JEPA only).
Make resolution per-model configurable. Do NOT force one global resolution across models.

Two-phase design (matches my workflow)

Phase 1 — global vectors only (--globals_only): extract just the global .pt files for all four models. Tiny (~37MB/model) and all SPARC training needs. I run this FIRST, start SPARC, and defer dense extraction.
Phase 2 — dense patch tokens (--patch_tokens): large (e.g. 18k×196×1024 float16 ≈ 7GB/model; CLIP bigger). Requirements:

Never hold dense tokens in RAM — write incrementally to a memory-mapped array / HDF5 / zarr, shaped (N, n_tokens, dim) float16.
Support --subset <file_list_or_row_indices> to extract dense tokens for ONLY a chosen set of images (the ones I'm inspecting for a concept) instead of all 18k. This is the main way I'll use Phase 2.
Reuse Phase 1's exact manifest/ordering, and run verify_alignment.py before writing.



Progress visibility & sequential execution
Always show live progress. For every extraction pass (Phase 1 globals and Phase 2 dense tokens alike), display a running progress bar (e.g. tqdm) showing images processed / total, current rate (img/s), and elapsed + estimated-remaining time. Print a clear header when each model starts (=== Extracting DINOv3 [1/4] — globals ===) and a summary line when it finishes (count done, count skipped, time taken, output file + size). I want to glance at the terminal at any moment and know exactly what stage it's on and how far along. Also log periodic checkpoints to stdout (e.g. "flushed at image 6000/18000") so progress is visible even if the bar is buffered.
Process one model at a time, sequentially — never all four in parallel. Load one model, run it fully over the corpus, write and flush its outputs, then explicitly free it (delete the model, torch.cuda.empty_cache(), gc.collect()) BEFORE loading the next. This keeps peak GPU/RAM to a single model's footprint and prevents out-of-memory crashes. Within a model, process images in batches (not all at once). Never hold more than one model's weights resident at a time.
Order: default to DINOv3 → ViT → CLIP → V-JEPA 2.1 (V-JEPA last, since it's the largest input resolution and riskiest — by then the other three have validated the pipeline). Make the order configurable, and support running a single model in isolation via a --model <name> flag, so if one fails I can re-run just that one without redoing the others (alignment still holds, since ordering comes from the frozen canonical manifest).
Between models, print an alignment-so-far note — after each model finishes, report its N and confirm it matches the canonical manifest's N, so a drift is caught immediately after the offending model rather than only at the end.
Output layout — STRICT per-model separation
<output_dir>/
  manifest.csv                  # CANONICAL: row_idx, filename, orig_size, mode, status
  manifest.hash
  verify_alignment.py
  dinov3/
    global.pt                   # (N,1024) float16
    patch_tokens.<mmap|h5|zarr> # (N,196,1024) float16 — dense, memory-mapped
    grid_shape.json             # {"h":14,"w":14,"n_tokens":196,"dim":1024,"has_cls":true,"n_reg":4}
    manifest.hash               # must equal canonical
    preprocessing.json
  vjepa21/
    global_meanpool.pt          # (N,1024) — PRIMARY
    global_maxpool.pt           # (N,1024) — alt
    patch_tokens.<...>
    grid_shape.json             # + documented spatiotemporal→2D mapping
    manifest.hash
    preprocessing.json
  vit/
    global.pt                   # (N,1024)
    patch_tokens.<...>          # (N,196,1024)
    grid_shape.json
    manifest.hash
    preprocessing.json
  clip_img/
    global.pt                   # (N,768) post-projection — PRIMARY
    global_preproj.pt           # (N,1024) pre-projection CLS — secondary
    patch_tokens.<...>          # (N,256,1024)
    grid_shape.json
    manifest.hash
    preprocessing.json
  EXTRACTION_REPORT.md
Engineering requirements

Resumable: checkpoint every N batches; on restart, skip already-done rows (by row index against the canonical manifest — never by re-globbing). A crash at image 12k must not restart from zero and must not disturb ordering.
Batched, configurable batch_size / num_workers, torch.no_grad(), .eval(), cast to CPU float16 before accumulating.
Two GPUs (2× RTX 4500 Ada, 24GB, no NVLink): support a --device flag. Since models run sequentially (one resident at a time), a single-GPU-at-a-time run is the default; optionally allow pinning to a chosen GPU. Don't over-engineer sharding.
Deterministic: fixed seed; sorted, persisted file order; stable row indices across all models and both phases.
Per-model sanity checks: log output shape, dtype, mean/std/min/max of global vectors, count of NaNs/Infs (fail loudly if any), processed-vs-skipped counts, and ~8 sample global-vector norms for me to eyeball.
Alignment check is mandatory (see Cross-model row alignment) — a run that passes per-model sanity but fails alignment is a FAILED run.
Dummy-forward check before the full run: load each model, run one dummy image, print the resulting global-vector dim and patch-grid shape, and PAUSE for me to confirm they match expectations BEFORE processing 18k images. This is where a V-JEPA reshape bug or a wrong CLIP output surfaces cheaply.

Deliverable: EXTRACTION_REPORT.md
Per model: checkpoint used (from DOWNLOAD_REPORT), exact preprocessing, global-vector definition (CLS / mean-pool / post-projection), grid shape, output file paths + sizes, images processed/skipped. Plus, globally: final N, skip count, skip policy, and the alignment-check result (PASS/FAIL) across all four models. I need this verbatim for the paper's methods section.
What NOT to do

Don't train or touch SPARC/SAE. Extraction only.
Don't re-download checkpoints — load from checkpoints/.
Don't load multiple models simultaneously — one model resident at a time, freed before the next loads.
Don't resize all models to one resolution to "simplify."
Don't merge models into shared files — strict per-model folders.
Don't let orderings drift between Phase 1 and Phase 2, and never lose the row_idx↔filename mapping.
Don't emit outputs if alignment can't be verified — STOP and report instead.

Before writing the full implementation
Read checkpoints/DOWNLOAD_REPORT.md, then propose back to me: (1) the module structure; (2) the exact local-load code path per model; (3) your plan for the V-JEPA 2.1 image-mode token extraction and 2D grid reshape (the riskiest part — flag uncertainties); (4) the pre-validation + alignment-verification approach. Confirm the four global-vector definitions and the CLIP post-projection-as-primary choice before proceeding.
