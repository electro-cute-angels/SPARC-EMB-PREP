# Latent-pair Trajectory Analysis

Explore the "trajectory" between any two SPARC latents on the trained
DINOv3 + V-JEPA 2.1 run (`n_latents=4096, k=64, 100 epochs`).

Everything here is **read-only** with respect to the trained run. It reuses the
already-computed val-set sparse activations
(`../concept_analysis/val_latents_<stream>.npy`) and the corpus manifest — no
embeddings or model forward passes are recomputed.

## Idea

For a latent pair `(X, Y)` and a mode, the val corpus is partitioned into three
regions from the sparse activations:

- **Region 1** — X fires, Y does not
- **Region 2** — both fire (the *between* region)
- **Region 3** — Y fires, X does not

`fires` defaults to strict nonzero (top-k gated activation `> 0`). An optional
per-latent percentile floor (`--percentile`) keeps only stronger activations.

Within **Region 2** we compute `log(act_X / act_Y)` per image and bin it into
`N` bins (default 9) spanning X-dominant → balanced → Y-dominant. **Bin counts
are a primary result** — their shape (U-shaped vs flat) tells us whether the two
concepts are disjoint-with-a-thin-bridge or genuinely blended. Counts are never
normalised and images are never forced into equal-size bins. Bins with `<5`
images are flagged as under-populated.

### Modes (Case A vs Case B)

- **shared** (Case A, primary): the encoder-independent ordering. The two
  per-encoder activations are reduced to one scalar per (image, latent) via
  `--reduce {mean,min,max}` (default `mean`), used for both firing and the
  log-ratio ordering.
- **dinov3** / **vjepa21** (Case B): the same partition/binning computed from
  each encoder's own activations, so their orderings can be compared against
  each other and against `shared`. The tool reports where they disagree — rows
  that change region, and Region-2 rows that land in a different bin depending on
  the encoder.

## Files

- `trajectory_core.py` — core analysis (activation loading, region masks,
  Region-2 log-ratio binning, cross-encoder disagreement, manifest alignment).
- `analyze_pair.py` — CLI. Prints the region-count report and optionally emits
  the per-pair JSON the app reads (`--emit`).
- `serve.py` — zero-dependency stdlib web server for the app.
- `app/` — static frontend (`index.html`, `style.css`, `app.js`).
- `data/` — precomputed per-pair JSON + `pairs.json` index (generated).

## Run the analysis

Region-count first step (the gate — is there a meaningful "between"?):

```bash
python3 analyze_pair.py --x 2234 --y 3942
```

Stricter firing floor, or a different shared reduction:

```bash
python3 analyze_pair.py --x 2234 --y 3942 --percentile 25 --reduce min
```

Precompute the JSON for the app (any pair):

```bash
python3 analyze_pair.py --x 2234 --y 3942 --emit
```

`--emit` writes `data/pair_<X>_<Y>.json` and updates `data/pairs.json`.
Useful flags: `--n_bins` (default 9), `--exemplar_cap` (max exemplars stored per
bin, default 64; the app shows a configurable subset).

> Paths are derived from this folder's location, so run the scripts by their
> absolute path or `cd` into this directory first.

## Launch the app

```bash
python3 serve.py --port 8770
# open http://localhost:8770/
```

In the app you can:

- Type a latent pair `X × Y` and **load** it (must be emitted first).
- Toggle **shared / dinov3 / vjepa21**, or **compare all** to see the three
  bin distributions side by side.
- See the overall **trajectory strip** — every between-image ordered
  X-dominant → balanced → Y-dominant, each labelled with its log-ratio. Click a
  strip image to select its bin.
- See the **Region 1/2/3** totals and the **bin histogram**; under-populated
  bins (`<5`) are hatched.
- Click a bin (in the histogram or the trajectory strip) to view its images
  (ordered by total activation `act_X + act_Y`), each labelled with `act_X`,
  `act_Y`, and `log-ratio`. Click an exemplar image to open it full-size in a
  new tab.
- Set **shown/bin** to control how many exemplars are displayed; the true bin
  count is always shown alongside.

Images are served from the corpus via the `row_idx ↔ filename` manifest, and the
server validates every requested filename against that manifest before reading
it (no path traversal, no mislabelled images).

## Reference pair finding (2234 = angel wings, 3942 = golden circular frames)

Over `N = 3531` val images, strict nonzero firing:

| mode | fires X | fires Y | R1 (X only) | R2 (between) | R3 (Y only) |
|------|--------:|--------:|------------:|-------------:|------------:|
| shared | 629 | 45 | 619 | 10 | 35 |
| dinov3 | 629 | 45 | 619 | 10 | 35 |
| vjepa21 | 629 | 45 | 619 | 10 | 35 |

The bridge is **thin (10 images)**: X is common, Y is rare. The 10 bridge images
are the same set in all three modes, but their within-bridge *ordering* differs
by encoder (9 of 10 change bin between DINOv3 and V-JEPA) — DINOv3 spreads them
X-dominant (log-ratio up to ~1.34), V-JEPA compresses toward balanced
(up to ~0.71). Expect sparse/under-populated bins for this pair; richer pairs
will populate the histogram more fully.
