# training/

This folder will contain the actual R&D code:

- `make_dataset.py` — builds training pairs from studio audio + concert-style augmentations
- `train.py` — trains/finetunes an embedding model (contrastive / triplet)
- `eval_veep.py` — evaluation harness on VEEP concert MP4 extracts (validation only)
- `export_coreml.py` — converts the model to CoreML for on-device inference

Next: implement these scripts in a minimal, reproducible way.
