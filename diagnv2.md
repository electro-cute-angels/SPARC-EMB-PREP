# PREPROCESSING_AND_R2

Scope: diagnosis + corrected reporting on the EXISTING checkpoint. No retraining, no
method changes, no preprocessing changes. Existing run outputs were not overwritten;
all new files live in this diagnostics folder.

Run analyzed: `sparc_experiment/runs/art_dinov3_vjepa21_n8192_k64_ep50`
Source data: `analysis_cache_val.h5` (3531 val samples), raw `.pt` features.

New artifacts in this folder:
- `compute_corrected_r2.py`
- `r2_matrix_corrected.csv`
- `r2_corrected_details.json`

---

## Part 1 — Corrected R² (existing checkpoint)

### Three variants, full source→target matrix (rows = target, cols = source)

**1. Uniform per-dim average (ORIGINAL, sklearn default `uniform_average`)**

| target \ source | dinov3 | vjepa21 |
|---|---:|---:|
| **dinov3** | -3.1382 | -1.2438 |
| **vjepa21** | +0.5653 | +0.5955 |

**2. Variance-weighted / flattened (`multioutput='variance_weighted'`) — the true quality metric**

| target \ source | dinov3 | vjepa21 |
|---|---:|---:|
| **dinov3** | **+0.6829** | +0.6153 |
| **vjepa21** | +0.6639 | **+0.6988** |

**3. Per-dim average with variance floor (exclude target dims with var < t, then uniform-average)**

At **t = 1e-6** and **t = 1e-8** (identical here — only one dim is affected):

| target \ source | dinov3 | vjepa21 |
|---|---:|---:|
| **dinov3** | **+0.6634** | +0.5975 |
| **vjepa21** | +0.5653 | +0.5955 |

Dimensions excluded by the floor (depends only on target variance):

| target | dims excluded @1e-6 | dims excluded @1e-8 |
|---|---:|---:|
| dinov3 | 1 | 1 |
| vjepa21 | 0 | 0 |

### One-line verdict
**Yes — under robust metrics both streams reconstruct healthily.** Every cell of the
variance-weighted matrix is in **0.62–0.70**, and the floored per-dim matrix restores
DINOv3 self-reconstruction from −3.14 to **+0.66** by removing a single dimension.

### What changed and why
- The original **uniform** average gives every dimension equal weight regardless of how
  much variance it carries. One near-constant DINOv3 dimension has an almost-zero
  denominator, producing a per-dim R² of ≈ **−3892**, which single-handedly drags the
  mean to −3.14 (and the cross value to −1.24).
- **Variance-weighted** R² weights each dim by its target variance, so the near-constant
  dim contributes almost nothing. Result: DINOv3 self **+0.68**, all cells healthy.
- **Floored** per-dim average (dropping the 1 sub-1e-6 dim) gives DINOv3 self **+0.66**,
  confirming the negative score was entirely due to that single dimension.

Note on a prior number: the earlier diagnostic quoted a "flattened" DINOv3 self of ~0.80.
That used a single global mean over the fully flattened array (which inflates total
variance). The correct dimension-aware figure is the **variance-weighted** value, **0.68**.
Either way the conclusion is identical: healthy reconstruction.

---

## Where the pathological DINOv3 dimension comes from (raw vs post-normalization)

Measured on all 17,656 images:

| | DINOv3 dim #757 | V-JEPA worst dim (#736) |
|---|---:|---:|
| raw variance | 2.04e-08 | 5.28e-02 |
| raw std | **0.000143** | 0.2297 |
| raw mean | -0.00592 | 0.2759 |
| post-L2-norm variance | 6.15e-10 | 3.40e-05 |
| dims below var 1e-6 (raw) | 1 | 0 |
| dims below var 1e-6 (normalized) | 1 | 0 |

**Origin: the dimension is near-constant already in RAW DINOv3 space.** Its raw std
(0.000143) is ~2500× smaller than DINOv3's median per-dim std (0.36). Per-sample L2
normalization does not create it — it merely preserves it as near-constant (var 6e-10).
This is an **intrinsic quirk of the DINOv3 CLS token** (one effectively-dead output
coordinate), not an artifact introduced by our `.pt→.h5` adapter or by the dataloader.
V-JEPA mean-pool has **no** such dimension.

---

## Part 2 — Is L2-only the repo's intended preprocessing?

**Definitive answer: YES. Per-sample L2 normalization (plus the model's learnable
per-stream `pre_bias`) is exactly what the official repo does. Our adapter did NOT skip
a standardization step — the repo performs none.** Grounded in the code:

### (a) Extraction stores RAW features (no normalization, no standardization)

`sparc/feature_extract/extract_coco.py` — DINO:
```python
feats = model(imgs).cpu().float()
...
feat_ds[idxs, :] = feats.numpy()   # stored raw
```
`sparc/feature_extract/extract_coco.py` — CLIP image/text:
```python
f_img = model.encode_image(img_tensor).cpu().float()
f_txt = model.encode_text(text_tokens).cpu().float()
...
feat_img[idxs, :] = f_img.numpy()   # stored raw
feat_txt[idxs, :] = f_txt.numpy()   # stored raw
```
`sparc/feature_extract/extract_open_images.py` — same pattern:
```python
f_img = model.encode_image(img_tensor).cpu().float()
...
feat_img[idxs, :] = f_img.numpy()   # stored raw
```
The only `T.Normalize(mean=IMAGENET_MEAN, std=IMAGENET_STD)` in these files is the
**image-pixel** transform before the vision backbone — it does not touch the output
feature vectors. (One exception, irrelevant to image streams: the Open-Images **text**
stream via sentence-transformers uses `normalize_embeddings=True`, i.e. L2 — still not
per-dimension standardization.)

### (b) The dataloader applies per-sample L2, and nothing else

`sparc/datasets.py`, `HDF5FeatureDataset.__getitem__`:
```python
x = self.features[stream_name][idx].astype(np.float32)
# Normalize each feature vector to unit norm individually
norm = np.linalg.norm(x)
if norm > 0:
    x = x / (norm + 1e-8)
```
There is no mean-subtraction, no variance scaling, no `StandardScaler`, no stored
statistics. A repo-wide search for z-score / standardization utilities found none.

### (c) The model centers with a LEARNABLE bias initialized to zero (not to data means)

`sparc/model/model_global.py`:
```python
self.pre_biases[stream_name] = nn.Parameter(torch.zeros(d_in))
```
Encoder subtracts `pre_bias`, decoder adds it back. It is learned during training; it is
**not** initialized to the feature mean / geometric median, so there is no data-driven
standardization anywhere in the pipeline.

### Conclusion for Part 2
A user running the repo's own COCO / Open-Images pipeline gets **raw features on disk,
L2-normalized per sample at load time, and a learnable zero-init pre-bias** — precisely
what our `.pt→.h5` + SPARC dataloader path produced. **Our pipeline is method-faithful.**
The −3.14 was purely a metric artifact, not a preprocessing bug.

---

## Interaction with a future 768-dim CLIP stream (and ViT)

- L2 normalization equalizes overall **vector magnitude** across streams regardless of
  dimensionality, and the loss (`normalized_mean_squared_error`) further divides each
  sample's error by its own energy. So streams of different D and native scale are made
  broadly comparable **at the whole-vector level** — this is why the repo can mix DINO,
  CLIP-image and CLIP-text without per-dim standardization.
- However, L2-only does **not** equalize **per-dimension variance**. A 768-dim unit
  vector spreads unit energy over fewer coordinates, so its average per-dim variance is
  ~1024/768× higher than a 1024-dim stream; and, as DINOv3 shows, individual streams can
  carry near-constant coordinates. These do not hurt training (normalized-MSE is
  whole-vector) but they **will** distort the **uniform-average R² report** again.
- Recommendation for reporting once CLIP/ViT join: use **variance-weighted R²** (and/or a
  variance floor) as the primary metric for every stream. This is a reporting choice, not
  a preprocessing change.

---

## Recommendation

**(a) Our preprocessing matches the repo — no change needed.** The −3.14 was purely a
metric artifact caused by uniform per-dim averaging over one intrinsically near-constant
DINOv3 dimension. Under variance-weighted or variance-floored R², both streams reconstruct
healthily (0.62–0.70).

Proposed follow-ups (reporting only; not implemented in this task):
1. Adopt **variance-weighted R²** as the default in `run_analysis.py` / `R2.py`, keeping
   uniform + floored as secondary columns for transparency.
2. Keep preprocessing as-is (raw store + per-sample L2 + learnable pre-bias) to stay
   faithful to the SPARC method when CLIP (768) and ViT (1024) streams are added.

No rerun is warranted on preprocessing grounds. Awaiting your go-ahead before changing any
reporting defaults.
