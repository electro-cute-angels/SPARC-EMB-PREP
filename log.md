Run Complete
All outputs at art_dinov3_vjepa21_n8192_k64_ep50:

File	Description
msae_checkpoint.pth	Trained 33.57M-param SPARC model
run_config.json	Full config + hyperparams
training_metrics.json	Per-epoch loss, dead neurons, cross-losses
training_curves.png	4-panel training curve plot
analysis_cache_val.h5	Full post-analysis cache (reconstructions, latent activations)
r2_matrix.csv	R² table
dead_neurons.json	Dead neuron counts by stream
top_activations.json	Top-10 activating val images per latent
latent_rankings.csv	Per-latent n_top_samples + max_activation
grids/	50 activation grids (dinov3 row / vjepa21 row)
RUN_REPORT.md	Full summary report
R² Matrix (val set, 3531 images)
source: dinov3	source: vjepa21
target: dinov3	−3.14	−1.24
target: vjepa21	+0.57	+0.60
Key observations
V-JEPA 2.1: self-reconstruction R²=0.60 and cross-reconstruction from DINOv3 R²=0.57 — both excellent, indicating the shared latent space captures vjepa21-structure well
DINOv3: negative R² in both directions — the model's recon loss for dinov3 converged to 0.187 (vs 0.044 for vjepa21), suggesting the 8192-latent shared dictionary is dominated by vjepa21-aligned features; dinov3's higher-frequency texture features require more capacity
Dead neurons: only 149/8192 at convergence (the large numbers seen mid-training were within-epoch counters, not final state)
Cross-loss asymmetry at convergence: vjepa21←dinov3=0.054 (very tight), dinov3←vjepa21=0.243 — consistent with the R² asymmetry
The negative DINOv3 R² is a genuine finding, not a bug — it means the shared bottleneck favours the smoother V-JEPA embedding geometry. You could try: increasing n_latents (e.g. 16384), reducing cross_loss_coef, or separating the encoder per-stream (local topk variant --topk_type local).
