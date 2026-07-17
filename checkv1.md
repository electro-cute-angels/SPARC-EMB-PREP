Task: Diagnose the DINOv3/V-JEPA R² asymmetry before changing the method
The first SPARC run (DINOv3 + V-JEPA 2.1, n_latents=8192, k=64) produced a striking R² asymmetry:
                  source: dinov3    source: vjepa21
target: dinov3       −3.14            −1.24
target: vjepa21      +0.57            +0.60
V-JEPA reconstructs well (self 0.60, cross-from-DINOv3 0.57); DINOv3 is strongly negative in BOTH directions, including self-reconstruction (−3.14). A negative self-reconstruction is suspicious — a stream should reconstruct itself reasonably even in a shared space. Before interpreting this as a real finding or trying method-level fixes, we need to rule out preprocessing/normalization/pooling artifacts. This task is diagnostic only — do NOT retrain with new method settings yet. Investigate, report, and we decide next steps together.
Step 1 — Per-stream feature statistics (the most likely culprit)
Load the raw pre-extracted global vectors for both streams (dinov3/global.pt, vjepa21/global_meanpool.pt) and report, for each stream:

Overall shape, dtype
Per-feature mean and std (report the distribution: min/median/max across the 1024 dims, not all 1024 values)
Per-vector L2 norm: distribution across the N images (mean, std, min, max, a few percentiles)
Global scale: overall mean absolute value, overall variance
Effective rank / intrinsic dimensionality: compute the covariance matrix eigenvalue spectrum and report the participation ratio (or stable rank = (Σλ)²/Σλ²) for each stream. This tells us if one stream is effectively much lower-rank (hence easier to reconstruct) than the other.
Present DINOv3 vs V-JEPA side by side and explicitly flag any large disparity in scale, norm, or effective rank.

Hypothesis being tested: if DINOv3 and V-JEPA differ substantially in scale/variance/rank, the shared MSE-based bottleneck may be implicitly favoring the lower-variance / lower-rank stream (V-JEPA), producing the asymmetry as an artifact rather than a fact about the models.
Step 2 — Confirm what preprocessing SPARC applied to the features
Inspect the SPARC training pipeline and our data-adapter, and report EXACTLY what normalization/standardization was applied to each stream before training:

Did each stream get per-stream standardization (zero-mean, unit-variance per dimension) or normalization (unit L2 norm per vector) before entering the encoders?
The SPARC paper standardizes/centers features per stream (learnable pre-bias b_pre subtracted at the encoder, decoder adds it back; and features are typically standardized before training). Confirm whether our run actually did this, or whether raw native-scale features went in.
Check specifically: does the repo's own dataloader (the HDF5 path) apply a standardization step that our .pt→adapter path might have skipped or done differently? If the repo standardizes in its data pipeline and our adapter bypassed it, that's likely the bug.
Report the exact preprocessing our features received vs. what the repo/paper intends. If there's a mismatch, that's a prime suspect — describe it precisely but don't fix it yet.

Step 3 — Assess the mean-pooling concern for V-JEPA
V-JEPA's global vector is a mean-pool over many tokens (V-JEPA has no CLS token), while DINOv3's is a single CLS token. Mean-pooling is a smoothing operation that tends to produce lower-variance, more-Gaussian, easier-to-reconstruct targets.

Compare the statistics from Step 1 through this lens: is V-JEPA's mean-pooled vector notably lower-variance / lower-rank / smoother than DINOv3's CLS token? Quantify it.
We also saved vjepa21/global_maxpool.pt (max-pool has different statistics). Report max-pool's stats alongside mean-pool's, so we can see how pooling choice affects the geometry. (Do NOT retrain yet — just compare the input statistics.)

Step 4 — Sanity-check the R² computation itself

Confirm how R² is being computed (per-dimension then averaged? over flattened vectors? on standardized or raw targets?). A negative R² can be exaggerated by computing it on a different scale than the reconstruction was optimized on.
Report whether R² is computed on standardized features or raw features, and whether that's consistent between the two streams. Confirm the −3.14 isn't partly a reporting-scale artifact.
Verify row-alignment held during this run: spot-check that batches paired the same image across streams (re-run verify_alignment.py if needed). Rule out that misalignment is contributing.

Deliverable
A concise DIAGNOSTIC_REPORT.md with:

The side-by-side per-stream statistics (Steps 1, 3)
A clear statement of what preprocessing was actually applied vs. intended (Step 2)
How R² was computed and whether it's scale-consistent (Step 4)
A ranked list of the most likely explanations for the asymmetry, each labeled as: (a) preprocessing/normalization artifact [fixable, not a finding], (b) pooling-choice artifact [partly a finding, partly a choice], or (c) genuine representational difference [a real finding]. Say which is most likely given the evidence, and what single change would test it.

Do NOT do yet

Don't retrain with different n_latents, cross_loss_coef, or local TopK. Those are next-step experiments we'll choose AFTER this diagnosis.
Don't "fix" anything silently — if you find the features weren't standardized, report it and propose the fix, but wait for confirmation before rerunning.
Don't overwrite the existing run's outputs — put diagnostics in a separate folder.

Report back with the DIAGNOSTIC_REPORT and your assessment before we decide the next run.
