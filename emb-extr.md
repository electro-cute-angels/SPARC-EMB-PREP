Task: Download and verify four vision-model checkpoints locally
Download the pretrained weights for four vision encoders into a local checkpoints/ subfolder and verify each one is complete and loadable. This task is downloading only — do NOT run the models on any images, do NOT build an extraction pipeline, do NOT train anything. The goal is simply to have all four checkpoints saved locally and confirmed working, so a later pipeline can load them offline.
Target folder
Save everything under checkpoints/, one subfolder per model:
checkpoints/
  dinov3_vitl16/
  vjepa21_vitl_384/
  vit_large_supervised/
  clip_vitl14/
  DOWNLOAD_REPORT.md
The four checkpoints (use the ViT-Large / ~300M tier for all)

DINOv3 ViT-L/16

Source: HuggingFace repo facebook/dinov3-vitl16-pretrain-lvd1689m
This repo is GATED: it requires being logged into HuggingFace and having accepted Meta's license on the model page. If the download fails with a 401/403/gated error, STOP and tell me clearly that I need to (a) log in via huggingface-cli login with a token, and (b) visit the model page in a browser and click to accept the license terms — then report exactly what error you hit. Do not try to work around gating.
Download the full repo (weights + config + preprocessor files) into checkpoints/dinov3_vitl16/.


V-JEPA 2.1 ViT-L/16 @384

Primary source: Meta's official checkpoint vjepa2_1_vitl_dist_vitG_384.pt from the facebookresearch/vjepa2 GitHub repo (the 2.1 weights, referenced under app/vjepa_2_1/). This is normally loaded via torch.hub, but I want the raw checkpoint file saved locally.
Check first whether I already have this file locally — ask me for a path before re-downloading, since I believe I already have a V-JEPA 2.1 checkpoint. If I do, just copy/symlink it into checkpoints/vjepa21_vitl_384/ and verify it, rather than downloading again.
If I don't have it: there is also a community HuggingFace port at Dev-Jahn/vjepa2.1-vitl-fpc64-384 that may be easier to pull. If you use the community port instead of Meta's official file, flag this explicitly in the report (it matters for provenance / the paper).
This is the riskiest download — the checkpoint is new and not in HF transformers yet. If the torch.hub / GitHub route is unclear, report what you found rather than guessing.


ViT-Large (supervised, ImageNet-21k→1k)

Source: timm / HuggingFace, identifier vit_large_patch16_224.augreg_in21k_ft_in1k
Repo: timm/vit_large_patch16_224.augreg_in21k_ft_in1k. Not gated.
Save weights + config into checkpoints/vit_large_supervised/.


CLIP ViT-L/14

Use OpenCLIP. Model ViT-L-14. For pretrained weights, prefer a DataComp tag if readily available (closest to the reference setup); otherwise fall back to the openai tag. Whichever you use, record the exact (model_name, pretrained_tag) in the report.
Alternatively the OpenAI HF repo openai/clip-vit-large-patch14 is acceptable — but document which one you actually downloaded.
Save into checkpoints/clip_vitl14/. Not gated.



Offline-friendliness

Download the actual weight files to disk (don't rely on a runtime cache that lives elsewhere). If a library defaults to caching in ~/.cache/huggingface or ~/.cache/torch, either set the cache/download dir to point into checkpoints/<model>/ or copy the resolved files there afterward, so the entire checkpoints/ folder is self-contained and portable.
The end state I want: I could disconnect from the internet and still load all four models from checkpoints/.

Verification (per model — this is the important part)
For each of the four, after downloading, verify it is complete and loadable WITHOUT running it on real data:

Load the checkpoint / instantiate the model from the local files (loading weights is fine; just don't run a forward pass on a corpus — a single tiny dummy tensor forward to confirm it instantiates is OK if easy, but not required).
Confirm the weight files exist, are non-zero size, and aren't truncated (checksum or at least file-size sanity).
Print, for each: the local path, total size on disk, the resolved identifier/tag used, expected embedding dimension (should be 1024 for DINOv3 / V-JEPA / ViT; note CLIP's is 1024 pre-projection or 768 post-projection), and load status (OK / failed + error).

Deliverable: checkpoints/DOWNLOAD_REPORT.md
A short report listing, for each model: exact source (repo + tag/filename), local path, file size, verification status, and any caveats (e.g. "used community port for V-JEPA", "DINOv3 required license acceptance", "used openai tag for CLIP because DataComp wasn't accessible"). I need this for provenance.
Environment

Set up whatever's needed to download: huggingface_hub / huggingface-cli, timm, open_clip_torch, and note V-JEPA's dependency on Meta's vjepa2 repo. Pin versions in a requirements.txt.
If any download requires credentials (HF token for gated DINOv3), stop and tell me what to provide rather than failing silently.
