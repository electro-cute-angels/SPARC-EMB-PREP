1. The problem, stated so the group feels it

Open with the question that motivates everything, in the paper's own terms: do different architectures, trained with different objectives, converge on similar ways of representing the world? For our project this becomes: do machines trained on natural photographs and video organise an art-historical archive in the same way — and does either of them organise it in a way an art historian would recognise?

The obstacle is concrete and worth stating plainly: each encoder produces its own isolated representation. Dimension 47 of DINOv3 has no relationship to dimension 47 of V-JEPA. There is no shared coordinate system. Every comparison method has to solve this somehow.

2. Why SPARC — and how it relates to what we were already doing

This is where you position the methodological move honestly.

Where we started: RDX. RDX (Kondapaneni et al.) sidesteps the coordinate problem by working with within-model neighbour rankings — scale-invariant, training-free, and it surfaces differences between two encoders. It answers "what does A see as similar that B doesn't?" It remains the foundation of the project.

Note that the SPARC paper itself cites Kondapaneni as comparing models by aligning interpretable concept masks, and groups it with methods that "operate post-hoc or depend on modality-specific heuristics." That's the gap SPARC fills.

What RDX doesn't give us. RDX gives difference explanations, pairwise, asymmetric. It doesn't give a shared vocabulary — a set of concepts you can point at and say "both encoders have this one." For the corpus-side question ("what concepts organise the Fototeca?") we needed something that produces a dictionary, not a difference.

Why not USAE. USAE was the obvious candidate — one dictionary across models — but the paper documents its failure mode precisely: soft alignment through reconstruction, no constraint on which latent indices activate, so different streams can use different latents for the same input. Measured on the same protocol, USAE achieves 0.22 concept alignment against SPARC's 0.80.

Why SPARC. Two mechanisms, both worth explaining because they're what make the apps possible:

Global TopK — logits are aggregated across streams and one shared top-k index set is selected, so every latent either activates across all models or stays inactive across all of them. This is what makes "latent #123" mean the same thing in both encoders.
Cross-Reconstruction Loss — each stream's code must reconstruct the other stream's features, creating pressure toward shared semantics rather than mere statistical correlation.

Frame this as disciplinary transfer: SPARC is an existing CV interpretability method; the contribution is adopting it for visual culture studies.

3. What the training actually looks like

Keep this brief and concrete — the group needs to know it's tractable, not to follow the maths.

Input: frozen, pre-extracted embeddings. Nothing is fine-tuned. DINOv3, V-JEPA 2.1, CLIP stay exactly as shipped.
Architecture: per-stream encoders → shared 4096-latent dictionary → Global TopK (k=64) → per-stream decoders.
Corpus: ~18k Fototeca images, 224px, split train/val.
Cost: minutes on our hardware. Runs are cheap and re-runnable — the bottleneck is human interpretation, not compute.
What comes out: val_latents_<stream>.npy (the sparse codes), latent_metrics.csv (alignment, frequency, variance per latent), exemplar grids.

Worth mentioning the diagnostic work, briefly, as evidence of rigour: an alarming early R² result turned out to be a reporting artifact (one near-dead DINOv3 coordinate distorting a uniform per-dimension average), and confirming preprocessing was method-faithful required reading the repo's actual code. This is the kind of thing that silently corrupts results if unchecked.

4. What SPARC enables that nothing else does

This is the conceptual heart. Three things follow from the shared dictionary:

Sparsity → discrete, nameable units. An image becomes a short list of ~64 active latents instead of a dense blur.
Shared index space → cross-encoder comparison. Because latent #123 describes the same concept for both encoders, align_cos per latent is definable, and "do DINOv3 and CLIP agree about this image?" becomes an answerable question.
Fixed basis → spatial attribution. A latent's response can be projected back onto patch tokens to make a heatmap showing where in the image it fires.

State the critical caveat clearly: the dictionary belongs to one checkpoint. Latent #123 in DINOv3×CLIP has nothing to do with latent #123 in DINOv3×V-JEPA. Every metric, exemplar and heatmap is per-run. This is why all three apps are keyed by run.

5. What I built — the three apps

Present them as a coherent triad: three directions through the same trained artifacts.

	Direction	Question
concept_app (8752)	latent → images	What did each dictionary entry learn, and do both encoders agree?
image_decomposition (8771)	image → latents	Which learned concepts compose this specific photograph?
latent_trajectories (8770)	latent × latent → images	Is there a conceptual bridge between two concepts, and do the encoders see it the same way?

For each, one sentence on what it shows and one on why it's only possible with SPARC:

concept_app — browse alive latents ranked by alignment or variance; per-latent exemplars per stream with spatial heatmaps. The forward view: this is what the dictionary contains.
image_decomposition — the reverse view. Pick an image, see its k active latents ranked by magnitude or by activation × rarity (the rarity weighting matters: high-frequency latents like paper tone and print material otherwise dominate every image). Shows the full 4096-cell activation map and shared/divergent counts.
latent_trajectories — partition the corpus into Region 1 (only X fires), Region 2 (both — the "between"), Region 3 (only Y). Within Region 2, order by log(act_X/act_Y). Bin counts are a primary result, not a byproduct: their shape tells you whether two concepts are disjoint-with-a-thin-bridge or genuinely blended.

All three read the same artifacts, share a run picker, compute lazily, and cache.

6. A worked example — and an honest result

Use the 2234 (angel wings) × 3942 (golden circular frames) finding, because it demonstrates the method and shows you're reporting what you found rather than what you hoped:

Over 3531 val images: X fires on 629, Y on 45, and only 10 images sit in the bridge. Wings are common, gilt roundels rare, and they barely co-occur. But: the 10 bridge images are identical across all three modes, while their ordering differs by encoder — 9 of 10 change bin between DINOv3 and V-JEPA. DINOv3 spreads them X-dominant (log-ratio to ~1.34); V-JEPA compresses toward balanced (~0.71).

That's the finding in miniature: the two encoders agree on what's there and disagree on how much. A thin bridge is a real result about iconographic co-occurrence in the Fototeca, not a failure of the method.

7. Why this is urgent for the group

Frame in terms the workshop collaborators will care about:

It operationalises the bidirectional move — reading encoders through the corpus and the corpus through the encoders — with concrete, inspectable artifacts rather than assertion.
The dictionary is not a neutral window. It's a manufactured third space that didn't exist before training and creates the very possibility of "shared concept" as a legible category. This is the Stieglerian tertiary-retention point, and it's instantiated rather than argued.
The domain gap is a live question, not a footnote: none of these encoders was trained on painted or archival material. What the dictionary finds on the Fototeca is a genuine open question.
Concepts are shown, not named. With no labels, latents are defined ostensively by their exemplars — which for a corpus whose categories may resist clean naming is arguably truer than labelling.
8. Where next
Concept-reading at scale — the human interpretive pass is the actual bottleneck.
Additional pairs (DINOv3×CLIP, V-JEPA×CLIP) — the runs exist; the comparison across runs is the next analysis.
Compositionality: do latents bundle? Does the same bundle appear in both encoders? (Cross-stream co-occurrence is a test nobody in the SAE literature can run, because nobody else has an aligned cross-model dictionary.)
The open methodological caution worth naming to the group: aggregation-based alignment can make latents look aligned without being concept-consistent. Triangulating alignment scores against human reading of the grids is what separates "co-fires" from "shares a concept."

Two suggestions on delivery. The 2234×3942 example is your strongest slide — it's concrete, visual, and the thin-bridge result is more persuasive than a clean one would be, because it shows the method reporting reality. And I'd put the "dictionary belongs to one checkpoint" caveat early rather than late; it's the single thing most likely to cause confusion when collaborators start clicking between runs and wondering why latent #123 changed meaning.

sum-up.md

155 lines

MD

README.md

123 lines

MD

I am presenting the work done with SPARC, i've been training some models. Attached you can find some informations, please make an outline and re read the whole paper and clearly try to explain why I've decided to use SPARC, how we can use it, what we can do, why is it relevant or urgent for the group research, what SPARC enables, how does the training looks like, what did i work on (the three different apps)

Here's the outline, drawing on the paper's own framing alongside your build.

Presenting the SPARC work — outline
1. Opening: the question, and why it's hard

Lead with the paper's own motivating question, reframed for the group: as AI models rapidly grow in numbers, a fundamental question emerges — do different architectures, trained with different objectives and modalities, independently converge on similar ways of representing the world?

For us: do encoders trained on natural photographs and video organise an art-historical archive the same way — and does either organise it in a way an art historian would recognise?

The obstacle, stated plainly: each encoder produces its own isolated representation. Dimension 47 of DINOv3 has no relationship to dimension 47 of V-JEPA. There is no shared coordinate system. Current interpretability methods, including sparse autoencoders, are designed to analyze models in isolation, creating isolated concept spaces unique to each model and making direct comparison difficult.

2. Why SPARC — positioned against what we already had

Where we started: RDX. Training-free, scale-invariant, works on within-model neighbour rankings, surfaces differences between two encoders. Still the project's foundation. Note the SPARC paper cites it directly — Kondapaneni et al. compares models by aligning interpretable concept masks — and groups it with approaches that operate post-hoc or depend on modality-specific heuristics. That's the gap.

What RDX doesn't give. Difference explanations, pairwise and asymmetric — but no shared vocabulary. For the corpus-side question ("what concepts organise the Fototeca?") we need a dictionary, not a difference.

Why not USAE. The obvious candidate, and the paper documents its failure precisely: USAE relies on soft alignment through reconstruction without an explicit constraint on which latent indices activate, allowing different streams to use different subsets of latents for the same input. Measured on the same protocol: SPARC achieves a concept alignment Jaccard similarity of 0.80 on Open Images; USAE achieves only 0.22.

Why SPARC. Two mechanisms — worth explaining because they're exactly what make the apps possible:

Global TopK: enforces identical active indices across all streams, ensuring each latent dimension either activates consistently across models or remains inactive across all of them. This is what makes "latent #123" mean the same thing in both encoders.
Cross-Reconstruction Loss: trains each model's latent representation to reconstruct inputs from other models, creating optimization pressure toward shared semantic understanding rather than mere statistical correlation.

Frame the contribution as disciplinary transfer: SPARC is an existing CV interpretability method; the move is adopting it for visual culture studies and digital art history.

3. What training actually looks like

Brief and concrete — the group needs to know it's tractable.

Input: frozen, pre-extracted embeddings. Nothing is fine-tuned; DINOv3, V-JEPA 2.1, CLIP stay as shipped.
Architecture: per-stream encoders → shared 4096-latent dictionary → Global TopK (k=64) → per-stream decoders.
Corpus: ~18k Fototeca images, 224px, train/val split.
Cost: minutes on our hardware. Runs are cheap and re-runnable — the bottleneck is human interpretation, not compute.
Outputs: val_latents_<stream>.npy (sparse codes), latent_metrics.csv (per-latent alignment, frequency, variance, alive flag), exemplar grids.

Worth a line on rigour: an alarming early R² result turned out to be a reporting artifact — one near-dead DINOv3 coordinate distorting a uniform per-dimension average — and confirming the preprocessing was method-faithful meant reading the repo's actual code rather than assuming. That class of error corrupts results silently.

4. What SPARC enables — the conceptual core

Three consequences of the shared dictionary, none of which are definable without it:

Sparsity → discrete units. An image becomes ~64 active latents instead of a dense blur.
Shared index space → cross-encoder comparison. align_cos per latent becomes measurable; "do DINOv3 and CLIP agree about this image?" becomes answerable.
Fixed basis → spatial attribution. A latent's response projects back onto patch tokens as a heatmap.

State the caveat early, not late: the dictionary belongs to one checkpoint. Latent #123 in DINOv3×CLIP has nothing to do with latent #123 in DINOv3×V-JEPA. Every metric, exemplar and heatmap is per-run. This is why all three apps are keyed by run — and it's the thing most likely to confuse collaborators clicking between runs.

5. What I built — three apps, three directions
App	Direction	Question
concept_app (8752)	latent → images	What did each dictionary entry learn, and do the encoders agree?
image_decomposition (8771)	image → latents	Which concepts compose this photograph?
latent_trajectories (8770)	latent × latent → images	Is there a bridge between two concepts, and do encoders see it alike?
concept_app — browse alive latents by alignment or variance; per-stream exemplars with heatmap overlays. The forward view: what the dictionary contains.
image_decomposition — the reverse. Ranked by magnitude or activation × rarity (the rarity weighting matters — otherwise ubiquitous latents like paper tone and print material dominate every image). Full 4096-cell activation map, shared/divergent counts.
latent_trajectories — Region 1 (only X) / Region 2 (both, the "between") / Region 3 (only Y); within Region 2, ordered by log(act_X/act_Y). Bin counts are a primary result — their shape says whether two concepts are disjoint-with-a-thin-bridge or genuinely blended.

All three are thin readers over the same artifacts, share a run picker, compute lazily, cache per run.

6. A worked example — and an honest result

The 2234 (angel wings) × 3942 (golden circular frames) pair, because it demonstrates the method and shows you report what you find:

Over 3531 val images — X fires on 629, Y on 45, only 10 in the bridge. Wings common, gilt roundels rare, barely co-occurring. But the 10 bridge images are identical across all three modes, while their ordering differs by encoder: 9 of 10 change bin between DINOv3 and V-JEPA. DINOv3 spreads them X-dominant (log-ratio to ~1.34); V-JEPA compresses toward balanced (~0.71).

The finding in miniature: the encoders agree on what's there and disagree on how much. A thin bridge is a real result about iconographic co-occurrence in the archive, not a failure.

7. Why this matters for the group now
It operationalises the bidirectional move — reading encoders through the corpus, the corpus through the encoders — with inspectable artifacts rather than assertion.
The dictionary is not a neutral window. It's a manufactured third space that didn't exist before training, and it creates the very possibility of "shared concept" as a legible category — the tertiary-retention point, instantiated rather than argued.
The domain gap is a live question: none of these encoders saw painted or archival material in training.
Concepts are shown, not named. Without labels, latents are defined ostensively by exemplars — arguably truer for a corpus whose categories resist clean naming.
8. Where next
Concept-reading at scale — the human interpretive pass is the real bottleneck.
Additional pairs (DINOv3×CLIP, V-JEPA×CLIP) — runs exist; cross-run comparison is the next analysis.
Compositionality: do latents bundle, and does the same bundle appear in both encoders? Cross-stream co-occurrence is a test the SAE literature can't run — nobody else has an aligned cross-model dictionary.
Methodological caution to name openly: aggregation-based alignment can make latents look aligned without being concept-consistent. Triangulating scores against human reading of the grids is what separates "co-fires" from "shares a concept."
