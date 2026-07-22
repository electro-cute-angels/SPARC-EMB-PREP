Task: Per-image latent decomposition viewer (new standalone app)

Build a second, separate local webapp — new subfolder image_decomposition/ inside the run directory, independent from latent_trajectories/. Uses the existing trained SPARC model (n_latents=4096, k=64, DINOv3 + V-JEPA 2.1) and its already-computed activations. No retraining, no recomputation of embeddings. Don't modify existing outputs.

Visual style: match the existing latent_trajectories/ app — same typography, palette, spacing, component styling. This should feel like a sibling tool, not a different project. Reuse its CSS/design tokens where possible.

What it does

Reads the sparse code in reverse: pick any corpus image, see which latents compose it, each latent represented visually by its own top-activating exemplar images. We have no text labels — concepts are shown ostensively, by exemplar, not named.

Layout — grid-based, matching the house style

Row 1, first cell: the query image. Displayed prominently in the first cell of the first row. Pickable from all images available in the run (searchable/browsable by filename or row index, plus a random-image button). Show filename, row index, and a summary line: total active latents, and the shared/divergent breakdown (e.g. "9 shared, 3 divergent").

Also in row 1 (alongside or beside the query image): the full-dictionary activation map — a visualisation of all 4096 latents for this image, e.g. a dense grid/strip where each cell is one latent coloured by activation magnitude (zero = blank/dark). With k=64, ~64 cells light up out of 4096, making the sparsity immediately legible. Clicking a lit cell scrolls to / highlights that latent's column below.

Row 2 onward: one column per latent. Each column is a single latent from the image's decomposition, ordered left-to-right by rank, containing:

latent_id and its activation magnitude on this image (numeric + a small bar so relative strength reads at a glance)
a column of 6 exemplar images (M = 6, fixed) — that latent's top-6 activating images from the corpus, stacked vertically
a shared/divergent badge — whether the latent is well-aligned across DINOv3 and V-JEPA (use the cross-stream alignment scores already computed) or largely stream-specific. Make this visually obvious via colour/badge.
Exclude the query image itself from its own exemplar column.

N (number of latent columns) must be configurable and dynamically adjustable in the UI — a slider/input that re-renders live (range ~5–30, default ~10), so I can expand or contract the decomposition without reloading. The grid should reflow cleanly as N changes.

Stream handling — one at a time, keep it clean

Decompose in one stream at a time. A toggle switches between DINOv3 and V-JEPA 2.1; switching re-renders the entire decomposition (different active latents, magnitudes, and exemplars — exemplars always come from the currently selected stream). Do not show both streams' exemplars in the same column.

Ranking modes

Toggle for how the N latents are ordered:

By activation magnitude (default)
By activation × rarity — down-weighting latents that fire on a large fraction of the corpus, surfacing what's distinctive about this image rather than merely strong

I expect high-frequency, low-specificity latents (overall tonality, print material, general composition) to dominate the raw ranking, so the rarity-weighted view is likely where per-image character shows. Precompute each latent's corpus-wide firing frequency to support this.

Navigation
Clicking any exemplar image jumps to its decomposition — so I can traverse the corpus concept-by-concept.
Simple browse/paging through the corpus alongside direct selection.
Practical
Reuse existing activation records and the row_idx↔filename manifest; verify alignment before rendering so nothing is mislabelled.
Precompute per-image JSON (or an efficient lookup) so the UI is responsive; provide a script to (re)generate it.
Images served at good resolution — exemplar columns must be legible, not thumbnails.
Lightweight local stack (Flask/FastAPI + static frontend, or fully static reading precomputed JSON — your call, keep it simple), consistent with how latent_trajectories/ was built.
Short README in image_decomposition/ covering how to precompute and launch.
First step

Build the core grid view for a handful of sample images and show me before polishing — I want to check that the grid reads clearly at realistic scale, that both ranking modes behave as expected, that the N slider reflows well, and that the 4096-latent activation map is legible.
