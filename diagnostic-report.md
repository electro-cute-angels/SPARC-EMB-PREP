# DIAGNOSTIC_REPORT

## Scope
Diagnosis only. No retraining or method changes were performed.

Artifacts generated in this folder:
- `feature_diagnostics.json`
- `variance_r2_detail.json`
- `compute_feature_diagnostics.py`

---

## Step 1 + 3: Side-by-Side Feature Statistics (Raw Pre-Extracted Vectors)

### Streams compared
- DINOv3 CLS: `outputs/dinov3/global.pt`
- V-JEPA mean-pool: `outputs/vjepa21/global_meanpool.pt`
- V-JEPA max-pool: `outputs/vjepa21/global_maxpool.pt` (pooling sensitivity check)

All shapes are `(17656, 1024)`, source dtype `float16` (analyzed as `float32`).

### A) Per-feature mean/std across 1024 dims

| Metric | DINOv3 CLS | V-JEPA mean-pool | V-JEPA max-pool |
|---|---:|---:|---:|
| mean per-dim (min / med / max) | -1.70 / 0.0207 / 0.957 | -8.50 / -0.0362 / 22.48 | 0.182 / 3.67 / 33.04 |
| std per-dim (min / med / max) | 0.000143 / 0.361 / 0.658 | 0.230 / 0.387 / 3.159 | 0.213 / 0.738 / 4.578 |

### B) Per-vector L2 norm distribution across images

| Metric | DINOv3 CLS | V-JEPA mean-pool | V-JEPA max-pool |
|---|---:|---:|---:|
| mean +/- std | 14.74 +/- 0.78 | 38.83 +/- 1.78 | 142.80 +/- 5.72 |
| min / p1 / p50 / p99 / max | 11.69 / 12.76 / 14.76 / 16.34 / 17.45 | 33.93 / 35.51 / 38.56 / 43.83 / 48.86 | 113.14 / 127.57 / 143.19 / 154.46 / 160.11 |

### C) Global scale

| Metric | DINOv3 CLS | V-JEPA mean-pool | V-JEPA max-pool |
|---|---:|---:|---:|
| mean abs | 0.360 | 0.631 | 3.965 |
| variance | 0.213 | 1.475 | 4.235 |
| std | 0.461 | 1.214 | 2.058 |

### D) Effective rank / intrinsic dimensionality (covariance spectrum)

| Metric | DINOv3 CLS | V-JEPA mean-pool | V-JEPA max-pool |
|---|---:|---:|---:|
| stable rank ((sum λ)^2 / sum λ^2) | 43.33 | 11.42 | 39.29 |
| participation ratio (exp entropy) | 157.19 | 37.54 | 212.74 |
| top-10 explained variance | 40.5% | 66.5% | 37.9% |
| top-50 explained variance | 63.8% | 86.6% | 56.1% |

### Interpretation from statistics
- V-JEPA mean-pool is much more low-rank than DINOv3 CLS (stable rank 11.4 vs 43.3, top-10 EV 66.5% vs 40.5%).
- This supports the pooling-smoothing hypothesis: mean-pool geometry is easier to compress/reconstruct in a shared bottleneck.
- V-JEPA max-pool is very different from mean-pool (much larger scale and much higher effective rank), confirming pooling choice strongly changes geometry.

---

## Step 2: What preprocessing was actually applied vs intended

### What actually happened in this run
1. `.pt -> .h5` adapter (`pt_to_h5.py`) only cast to `float32` and split train/val by shared indices.
   - No per-dim standardization.
   - No centering.
2. SPARC dataloader (`sparc/datasets.py`) applies **per-sample L2 normalization** in `__getitem__`:
   - `x = x / (||x|| + 1e-8)`
   - This is done independently for each stream vector.
3. Model has learnable per-stream `pre_biases` and subtracts/adds them in encoder/decoder, but there is no explicit fixed z-score standardization transform in the pipeline.

### Intended (paper-style) preprocessing vs run
- Intended in your note: per-stream centering/standardization prior to training (or equivalent treatment).
- Actual run: only per-vector L2 normalization + learnable `pre_bias`; **no fixed per-dim zero-mean/unit-variance standardization**.

### Mismatch assessment
Yes, there is a mismatch from the intended standardization-first interpretation. This is a prime suspect for metric/pathology behavior.

---

## Step 4: R² computation sanity check

### How R² is computed in repo
In `R2.py`:
- Uses `sklearn.metrics.r2_score(raw[target], pred)` with 2D arrays.
- sklearn default multioutput mode is uniform average across dimensions.
- So final R² is mean of per-dimension R² values (equal weight per dim), not flattened global variance-weighted R².

### Is it on standardized or raw targets?
- Computed on `analysis_cache_val.h5` "raw" and reconstructions.
- Those cached features come from `HDF5FeatureDataset`, so they are already per-sample L2 normalized vectors.
- Therefore R² is scale-consistent between streams for that normalization convention.

### Critical artifact found
For DINOv3 (on normalized val features):
- Per-dim variance min: `6.08e-10`
- Exactly 1 dimension has extremely tiny variance and yields per-dim self R² around `-3892`.
- Remaining DINO dimensions are mostly good (median per-dim self R² ~`0.666`; 1st percentile ~`0.503`).
- Because repo averages per-dim R² uniformly, that one pathological dimension drags overall DINO self R² to `-3.14`.

Evidence:
- Reported repo R²: DINO self `-3.138`, V-JEPA self `0.595`.
- Flattened R² (variance-weighted by total target variance) from same cache:
  - DINO self: `0.800`
  - V-JEPA self: `0.952`

So the extreme negative DINO self R² is largely a reporting artifact of uniform per-dim averaging with a near-constant dimension.

### Alignment check
Re-ran `verify_alignment.py` with sample checks: **PASS**.
No evidence of row misalignment contributing.

---

## Most likely explanations (ranked)

1. **R² reporting artifact due to near-zero-variance DINO dimension**
   - Label: **(a) preprocessing/normalization artifact [fixable, not a finding]**
   - Why likely: one DINO dimension has tiny variance after L2 normalization and catastrophic per-dim R²; uniform averaging amplifies it.

2. **Preprocessing mismatch: L2 normalization only, no per-dim standardization**
   - Label: **(a) preprocessing/normalization artifact [fixable, not a finding]**
   - Why likely: this mismatch can create unstable per-dim variance structure and make per-dim R² sensitive/outlier-prone.

3. **Pooling-choice geometry effect (V-JEPA mean-pool is low-rank/easier)**
   - Label: **(b) pooling-choice artifact [partly a finding, partly a choice]**
   - Why likely: strong rank disparity (stable rank 11.4 vs 43.3) plausibly favors V-JEPA reconstruction in shared sparse bottleneck.

4. **Genuine representational asymmetry**
   - Label: **(c) genuine representational difference [real finding]**
   - Still plausible, but current evidence suggests metric/preprocessing artifacts explain most of the alarming negative DINO self value.

---

## Single best next test (without broad method changes)

Run post-analysis R² with a robust metric protocol on the same checkpoint:
- compute per-dim R² but exclude dims with target variance below a threshold (e.g. `< 1e-8`), and
- also report flattened R² (global variance-weighted).

If DINO self R² becomes reasonable under variance-aware reporting, the `-3.14` was mostly an artifact, not a model failure.

---

## Bottom line
The strongest current explanation is **artifact, not fundamental failure**:
- negative DINO self R² is dominated by one near-constant normalized DINO dimension under uniform per-dim averaging;
- preprocessing deviates from intended per-stream standardization;
- V-JEPA mean-pool is intrinsically lower-rank and easier to reconstruct.

No retraining was done in this diagnostic task.
