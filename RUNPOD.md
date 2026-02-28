# RunPod — Resonance R&D Compute

Resonance training requires a GPU. We use **RunPod** for on-demand GPU instances.

## 1) Create RunPod account + API key
1. Sign up: https://www.runpod.io/
2. Add billing
3. Create an **API Key**: https://www.runpod.io/console/user/settings

## 2) Pick a Pod
Recommended starting points:
- **RTX 4090**: best value for iteration
- **A100**: faster + more VRAM if we move to transformers

Suggested pod settings:
- Disk: **50–100 GB**
- Image: PyTorch + CUDA 12.x (or Ubuntu + install)

## 3) Run the repo on the pod
```bash
sudo apt-get update && sudo apt-get install -y git ffmpeg

git clone https://github.com/vampthedge/resonance
cd resonance

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# (coming next) dataset + training
python -m training.make_dataset --help
python -m training.train --help
python -m training.eval_veep --help
```

## 4) Artifacts (until HF is set up)
Until you create a Hugging Face org/account, we’ll store:
- checkpoints in a private object store (S3 / Cloudflare R2) **or** as GitHub Releases (small only)
- metrics reports in `reports/`

## 5) Rules
- Never commit API keys.
- Treat audio data as licensed: keep it in controlled storage.
