Task: Adapt and run SPARC on our pre-extracted DINOv3 + V-JEPA 2.1 embeddings
We have pre-extracted, row-aligned global embeddings for a ~18k-image corpus from multiple frozen vision encoders. Goal: get the official SPARC repo (github.com/AtlasAnalyticsLab/SPARC, TMLR 2026, "Sparse Autoencoders for Aligned Representation of Concepts") training on our own features, starting with a two-stream run: DINOv3 + V-JEPA 2.1. Then produce a defined set of analysis outputs. Use the official repo's model/training code as-is wherever possible — only replace the data-loading/feature-extraction layer. Do not reimplement SPARC from scratch.

Working directory — keep everything self-contained
Do all of this inside a single subfolder, sparc_experiment/ (create it in the project root). Everything the agent creates — the cloned SPARC repo, the env/requirements, the adapted dataloader, configs, run outputs, and analysis artifacts — lives under sparc_experiment/. Do NOT scatter files across the project root.
Suggested layout:
sparc_experiment/
  SPARC/                     # cloned official repo (untouched except the dataloader hook)
  configs/                   # our configs, e.g. config_fototeca_dino_vjepa.json
  data_adapter/              # our features dataloader + stream-config code
  runs/                      # all training runs + outputs, one subfolder per run
    <run_name>/
      checkpoint / config / R2_matrix.csv / latent_rankings/ / grids/ / RUN_REPORT.md
  requirements.txt
  README.md                  # how to run, where things are
The pre-extracted features stay where they are (<features_dir>/, outside this subfolder) — reference them by path, do not copy them in. Everything else the agent produces is confined to sparc_experiment/.

Background on the repo (verify against the actual code once cloned)

Entry point is main.py, config-driven via configs/config_*.json, with CLI flags including --topk_type {global,local}, --n_latents, --k, --cross_loss_coef, --only_eval.
The core method: per-stream affine encoders → Global TopK (aggregate logits across streams, select one shared top-k index set) → per-stream affine decoders; trained with self-reconstruction + cross-reconstruction loss (NMSE), coefficient --cross_loss_coef (λ). Auxiliary loss (AuxK) for dead neurons.
Feature extraction lives in sparc/feature_extract/extract_*.py and is hardwired for DINO/CLIP on COCO/Open Images. We are NOT using this — we already have features. The main integration work is a new data loader that feeds our features into the existing training loop.
Clone the repo, read its actual structure (main.py, the model definition, the training loop, the config schema, the dataset/dataloader class), and report back how the dataloader hands batches to the model BEFORE adapting — match its expected batch format exactly.

Our input features (already on disk, row-aligned)
Located under <features_dir>/, one subfolder per model, with a shared canonical manifest.csv (row_idx → filename) already verified so row i is the same image across all models:

dinov3/global.pt — (N, 1024) float16, CLS token
vjepa21/global_meanpool.pt — (N, 1024) float16, mean-pooled (PRIMARY global for V-JEPA)
(also present, not used in this first run: vit/, clip_img/, *_maxpool, *_preproj, and dense patch_tokens.* files)

For THIS run use exactly two streams: DINOv3 global.pt and V-JEPA 2.1 global_meanpool.pt. Both are 1024-dim here, but do NOT hardcode equal dims — SPARC supports differing per-stream input dimensions, and later runs will add CLIP (768-dim), so the loader and config must handle arbitrary per-stream dims from the start.
Data-loading adaptation (the core task)

Write a dataset/dataloader that loads our .pt global-vector tensors and yields batches in the exact format SPARC's training loop expects (inspect the repo to confirm — likely a dict or tuple keyed by stream, e.g. {stream_name: tensor[B, d_s]} for the same B rows across streams).
Alignment is guaranteed by row index — batches must draw the same row indices across streams. Do not shuffle streams independently. If SPARC's loader shuffles, shuffle indices once and apply the same permutation to all streams.
Add a startup assertion: all loaded stream tensors have identical N and match the manifest's N. Fail loudly otherwise.
Respect a train/val split (the repo uses train_ratio ~0.8, seed 42 — reuse its convention). The split must be applied by row index identically across streams.
Make the stream set config-driven: a list of (stream_name, path_to_pt, dim) so adding streams later (CLIP, ViT) is a config edit, not a code change.

Training configuration for the first run
Create a config (e.g. configs/config_fototeca_dino_vjepa.json) with:

Two streams: dinov3 (1024) and vjepa21 (1024)
--topk_type global (this is the SPARC method; we want shared concepts)
--n_latents 8192 and also be ready to sweep 4096 (our corpus is ~18k, ~100× smaller than the paper's Open Images, so a smaller dictionary may be more appropriate — make n_latents an easy CLI/config override)
--k 64
--cross_loss_coef 1.0 (λ=1). Also support a λ=0 ablation run (self-reconstruction only) for comparison, since that ablation is central to the paper's argument.
Optimizer/schedule: reuse the repo defaults (Adam, lr 1e-4, 50 epochs, batch 256, seed 42, AuxK for dead neurons) unless they conflict with our smaller data — flag if so.
Since these are cached features, training should be fast (paper reports <25 min for 3-stream/1.7M on an H100; our 2-stream/18k on a single RTX 4500 Ada should be minutes). Treat runs as cheap and re-runnable.

Required outputs (this is what I actually need out of the run)
After training, produce and save the following into a runs/<run_name>/ folder. These correspond to the SPARC paper's own analyses — reuse the repo's evaluation code where it exists, implement where it doesn't:

Trained model checkpoint + the exact config used.
Reconstruction R² matrix (source→target, both streams, incl. diagonal self-recon and off-diagonal cross-recon). This is the primary sanity check — I especially need to see whether V-JEPA reconstructs sanely and whether DINOv3↔V-JEPA cross-recon is meaningfully positive (near-zero/negative = the two share little linearly-recoverable structure). Print as a clean table and save as CSV.
Activation-pattern / dead-neuron statistics: for the 8192 (or 4096) latents, how many are alive-in-both, dead-in-both, or alive-in-only-one stream. Report per-stream dead rates. (This is the effective size of the shared dictionary — the paper's Table 1 analysis.)
Per-latent, per-stream activation records: for every latent, the top-K activating images (row indices + filenames) for each stream, and each latent's mean activation / activation frequency. Save as a structured file (JSON/parquet) so I can build downstream tooling. This is the raw material for concept inspection.
Latent ranking tables — precompute and save several rankings of the latents so I can pick what to inspect without recomputing:

by cross-stream alignment (how consistently the same latent's top-activating images agree across streams — use the paper's generalized Jaccard on whatever grouping is available, or a co-activation overlap measure if no labels)
by activation frequency / energy
by variance across the corpus (discriminativeness)
Save each as a sorted table (latent_id, score).


Top-activating image grids for the top ~50 latents by cross-stream alignment AND the top ~50 by variance: a rendered contact sheet per latent showing its top-N images from each stream side by side (DINOv3 row, V-JEPA row), labeled with latent_id and scores. This is the main human-inspection artifact — mirror the paper's Figure 3 layout. Save as image files.
Training curves (loss components over epochs) and a short auto-generated RUN_REPORT.md summarizing config, final losses, R² table, dead-neuron stats, and where each output lives.

Explicitly NOT in this run (but design for it)

No heatmaps yet (those need the dense patch tokens — separate later step). But keep the per-latent top-image records keyed by row index so heatmaps can be added later for chosen latents/images.
No CLIP/ViT streams yet — but the stream set is config-driven so they drop in later.
Don't touch the repo's COCO/Open-Images extraction scripts — we bypass them entirely.

Engineering / hygiene

Set up a clean env with pinned requirements.txt; install the repo (pip install -e . if it supports it). Note it depends on the overcomplete package — install per the repo's instructions.
Fixed seed (42), deterministic split. Log everything (config, git commit of the SPARC repo, our feature-dir path + manifest hash) into the run folder for provenance.
Live progress output during training (epoch, loss components, ETA) and during the post-hoc analysis passes (which can be slower than training — computing top-activating images over 18k×8192 needs a batched, progress-barred pass, not an all-in-RAM operation).
Guard memory: the activation matrix (N × n_latents) for 18k×8192 float32 is ~590MB — fine in RAM, but the top-image computation over both streams should still be batched and reported with a progress bar.

Before implementing
Clone the repo and report back: (1) its actual module/训练-loop structure and the exact batch format the model expects; (2) how its existing dataloader and config schema work, so we match them; (3) your plan for the features dataloader and the stream-config format; (4) which of my 7 required outputs the repo already provides vs. which you'll implement. Confirm the two-stream DINOv3+V-JEPA setup and n_latents plan before running the full thing. Do a 1-epoch smoke test and show me the R² table + a couple of top-activating grids before committing to the full 50-epoch run.
Start by inspecting the repo — don't write the full adaptation until you've reported its actual structure back to me.
Note: parts of this repo's utils may be rough research code; if anything in the training loop or loss is unclear, surface it rather than guessing, especially anything touching the Global TopK aggregation or the cross-reconstruction loss (those are the method's core and must not be silently altered).
