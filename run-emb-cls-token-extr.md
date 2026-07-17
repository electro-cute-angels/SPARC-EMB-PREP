python3 extract_features.py \
  --corpus_dir corpus \
  --checkpoints_dir checkpoints \
  --output_dir outputs \
  --globals_only \
  --batch_size 32 \
  --device cuda:0 \
  --yes



  python3 extract_features.py \
  --corpus_dir corpus \
  --checkpoints_dir checkpoints \
  --output_dir outputs \
  --patch_tokens \
  --batch_size 8 \
  --device cuda:0 \
  --yes



  python3 verify_alignment.py --output_dir outputs --sample_k 10 --seed 42
