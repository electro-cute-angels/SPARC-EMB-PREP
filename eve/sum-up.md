# Interpretability apps — what they are and why SPARC makes them possible

This folder hosts three browser apps that sit **next to each other** under
`sparc_experiment/` and all work **across every trained run**:

| App | Port | Launch | One‑line purpose |
|-----|------|--------|------------------|
| `concept_app/` | 8752 | `python3 -m http.server 8752 --directory concept_app` | Browse a run's learned **dictionary** — what each latent means. |
| `latent_trajectories/` | 8770 | `python3 latent_trajectories/serve.py --port 8770` | Explore the **space between two latents** and whether both encoders agree. |
| `image_decomposition/` | 8771 | `python3 image_decomposition/serve.py --port 8771` | Read a single image **in reverse** — which latents compose it. |

Each app has a **run picker** (DINOv3 × V‑JEPA 2.1, DINOv3 × CLIP, V‑JEPA 2.1 ×
CLIP …). They share one corpus, manifest and validation split, so image
alignment is identical everywhere; only the per‑run sparse activations differ.

---

## 1. What is trained, and what it produces

SPARC is a **multi‑stream sparse autoencoder (SAE)**. For each image we already
have two dense embeddings — one per encoder in the pair (e.g. DINOv3 = 1024‑d,
V‑JEPA 2.1 = 1024‑d, CLIP = 768‑d). SPARC learns, jointly for both streams:

- a shared **overcomplete dictionary** of `n_latents = 4096` concept directions;
- per‑stream **encoders** that map an embedding to latent activations;
- a **global top‑k** gate (`k = 64`): every image lights up **exactly k** of the
  4096 latents, and the same latent index means "the same concept" in both
  streams (this cross‑stream tying is the point of SPARC).

After training, the `concept_analysis/` pipeline of a run emits the artifacts all
three apps read:

- `val_latents_<stream>.npy` — the `(N_val, 4096)` **sparse codes** for every
  validation image, one matrix per stream;
- `latent_metrics.csv` — per‑latent `align_cos` (cross‑encoder agreement),
  `freq_<stream>` (how often it fires), `var_*`, and an `alive` flag;
- exemplar grids / preview strips used as thumbnails.

Everything the apps show is derived from these; **no embeddings or model forward
passes happen at request time** except the cheap, on‑demand steps noted below.

---

## 2. Why these views only exist *because* SPARC was trained

A raw embedding is a dense 768/1024‑d vector: no axis is individually meaningful,
and two encoders live in **different, non‑comparable** coordinate systems. SPARC
changes three things that unlock every app:

1. **Sparsity → discrete, nameable units.** With top‑k, an image is a short list
   of ~64 active latents instead of a dense blur. That list *is* the
   decomposition the third app shows, and each latent can be characterised by the
   images that most activate it.
2. **A shared index space → cross‑encoder comparison.** Because latent #123 is
   forced to describe the same concept for both encoders, we can measure
   `align_cos` per latent, and we can ask "do DINOv3 and CLIP agree about this
   image / this bridge?" — the core question in the trajectories app.
3. **A fixed basis → stable coordinates for attribution.** The dictionary is a
   concrete set of weight vectors in the checkpoint, so a latent's response can be
   projected back onto an image to make a heatmap.

Without the trained SAE none of `align_cos`, the 4096‑latent activation map, the
region partition, or the heatmaps are even definable.

### Why "one heatmap per trained model"

A latent's spatial heatmap is produced by applying **that run's trained encoder
weights** to an image's patch tokens (the same encoder, evaluated per patch
instead of on the pooled global vector; for CLIP the patch tokens are first sent
through `visual.proj` so they live in the 768‑d space the SAE was trained on).

The dictionary is a property of **one checkpoint**. A different run learns a
different dictionary, so latent #123 in "DINOv3 × CLIP" has **nothing to do**
with latent #123 in "DINOv3 × V‑JEPA 2.1". Consequently:

- heatmaps, exemplars and metrics are **computed per run**, never shared between
  runs;
- for a given image, a latent has **one** heatmap **per stream** — the one its
  own trained encoder defines. There is no "generic" heatmap, because there is no
  concept without the model that learned it.

This is exactly why every app is keyed by run: switching the picker swaps the
entire dictionary, and with it every latent id, metric and heatmap.

---

## 3. `concept_app` — the dictionary browser (port 8752)

**Question it answers:** *what concept did each latent learn, and how consistent
is it across the two encoders?*

- Pick a run; browse its **alive** latents ranked by alignment or variance.
- For a selected latent, see its **top‑activating exemplar images per stream**,
  each with a spatial **heatmap** overlay, plus `align_cos`, per‑stream `freq`,
  and composite alignment/variance grids.
- Backend: a static site (`http.server`); the per‑run data under
  `concept_app/data/<run>/` is produced by `build_concept_app.py`, which is where
  the per‑image heatmaps are rendered from the trained encoder.

This is the "forward" view: **latent → the images that exemplify it.**

## 4. `latent_trajectories` — the between‑two‑latents explorer (port 8770)

**Question it answers:** *is there a smooth conceptual bridge between two latents,
and do both encoders see it the same way?*

- Pick a run and two latents **X**, **Y**. The validation set is partitioned into
  **Region 1** (only X fires), **Region 2** (both fire — the "between"), and
  **Region 3** (only Y fires).
- Within Region 2, images are ordered by `log(act_X / act_Y)` — X‑dominant →
  balanced → Y‑dominant — shown as a histogram and a **trajectory strip** of
  exemplars.
- The **encoder‑disagreement** panel counts images that land in different regions
  or different bins depending on which encoder (or the symmetric *shared*
  reduction) you use.
- Backend (`serve.py`): loads a run's two `val_latents_<stream>.npy` matrices,
  and **computes any of the ~8.4M latent pairs on demand** (a few array ops),
  caching each result to `data/<run>/pair_<X>_<Y>.json`.

This view is only meaningful because top‑k gives a crisp "fires / doesn't fire"
per latent, and because the shared index lets X and Y be compared across encoders.

## 5. `image_decomposition` — the reverse view (port 8771)

**Question it answers:** *which learned concepts make up this specific image, per
encoder?*

- Pick a run and one image. For the chosen stream you get its **k active latents**
  ranked by magnitude or by activation × rarity (idf‑style), each latent shown
  ostensively by **its own** top exemplars.
- A **4096‑cell activation map** shows the whole dictionary lit up for that image,
  plus **shared / divergent** counts (how many of the active latents are
  cross‑encoder‑aligned).
- Backend (`serve.py`): the image universe (`data/images.json`) is shared; the
  per‑run `data/<run>/meta.json` (per‑latent exemplars + freq + alignment) and
  each `/(image)/<run>/<valrow>.json` decomposition are **computed on demand** by
  a single row `argsort` of the sparse codes, then cached.

This is the "inverse" of `concept_app`: **image → the latents that compose it.**

---

## 6. How the three fit together

- `concept_app`: **latent → images** (what each dictionary entry means).
- `image_decomposition`: **image → latents** (the sparse code of one picture).
- `latent_trajectories`: **latent × latent → images** (the relationship between
  two dictionary entries, and cross‑encoder agreement about it).

All three are thin readers over the same trained artifacts, they all offer the
same run picker, and they all compute the expensive pieces (pairs, per‑image
decompositions, heatmaps) lazily and cache them under `data/<run>/`. Retraining a
run, or adding a new encoder pair, is all it takes for the apps to expose it —
because the dictionary the apps visualise *is* the thing SPARC learns.
