Task: Latent-pair trajectory analysis + visualisation webapp

Using the existing trained SPARC model (n_latents=4096, k=64, 100 epochs, DINOv3 + V-JEPA 2.1) and its already-computed activations, build an analysis + small webapp for exploring the "trajectory" between any two latents. No retraining. Create a new subfolder latent_trajectories/ inside the run directory; all code and outputs live there. Don't modify or overwrite existing run outputs.

Reference pair for development: latent 2234 (appears to be angel wings) and latent 3942 (golden circular frames). But everything must be parameterised for any latent pair.

Core analysis

For a given latent pair (X, Y) and a given stream, partition the corpus into three regions using the sparse activations:

Region 1: X fires, Y does not
Region 2: both fire — the "between" region
Region 3: Y fires, X does not

Note: with k=64 of 4096, most images have hard zeros on both — the working subset is small. "Fires" = nonzero (in the top-k) by default; make the threshold configurable (e.g. optional percentile floor) but default to strict nonzero.

First output, before anything else: report the three region counts. This determines whether the pair has a meaningful "between" at all. Print it plainly.

Then, within Region 2:

Compute log(act_X / act_Y) per image.
Bin the log-ratio range into N bins (default 9, configurable) spanning X-dominant → balanced → Y-dominant.
Record per-bin: count, image row indices, both activation values, and total activation (act_X + act_Y) for ranking exemplars within a bin.
Bin counts are a primary result, not a byproduct — the shape of the distribution (U-shaped vs. flat) tells us whether the concepts are disjoint-with-a-thin-bridge or genuinely blended. Never normalise this away or force equal images per bin.
Case A vs Case B
Case A (do this first, the primary deliverable): compute the above in the shared SPARC latent space — the encoder-independent ordering.
Case B (secondary): compute the same partition/binning separately using DINOv3's activations and V-JEPA's activations, so the two per-encoder orderings can be compared against each other and against Case A. Report where they disagree (e.g. images that land in different bins, or in different regions, depending on the encoder).
Webapp (the main visual deliverable)

A lightweight local web app in the visual style of the existing one concept_app called trajectory_app (single-page; Flask/FastAPI + static frontend, or fully static reading precomputed JSON — your call, keep it simple) that lets me:

Pick a latent pair (X, Y) and a stream/mode (shared / dinov3 / vjepa21).
See the distribution across bins as a histogram — bin counts clearly displayed, plus the Region 1/2/3 totals.
Click a bin to see the images in that bin, shown at good resolution, ordered by total activation, with each image's act_X, act_Y, and log-ratio labelled.
Show a configurable number of exemplars per bin (default ~5) but always display the true bin count alongside, and visually flag bins with very few images (e.g. <5) so they're not read as equivalent to well-populated ones.
Ideally: toggle between shared / DINOv3 / V-JEPA views on the same pair to compare orderings side by side.

Serve images from the existing corpus via the row_idx↔filename manifest — verify alignment before rendering so no image is mislabelled.

Practical
Reuse existing activation records / caches; don't recompute embeddings.
Precompute per-pair JSON so the app is fast; provide a script to generate it for any pair.
Progress bars on any batch computation.
Short README in latent_trajectories/ explaining how to run the analysis and launch the app.
First step

Run the region-count analysis for the 2234/3942 pair in all three modes (shared, dinov3, vjepa21) and report the numbers to me before building the webapp — if Region 2 is nearly empty, we'll rethink the approach before investing in UI.
