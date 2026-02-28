# TRAINING — Data, objectives, and the VEEP-driven flywheel

This document defines how we train an embedding model that is robust to **real concert audio** and deployable **on-device via CoreML**.

---

## 1) Problem formulation

We learn an embedding model \( f_\theta \) that maps an audio window \(x\) to an embedding \(e\in\mathbb{R}^d\) such that:

- same song (same underlying recording) → embeddings close
- different songs → embeddings far

Inference is **retrieval**: compute an embedding from the mic stream and match via cosine similarity against a **setlist micro-catalog**.

---

## 2) Data sources

### 2.1 Clean reference audio

- Studio tracks fetched from Apple Music / YouTube Music, or provided locally
- Used to build:
  - simulated degradations
  - clean anchors/positives for contrastive learning

### 2.2 VEEP concert MP4s (ground-truth real degradation)

Founder access to downloaded concert MP4s (with known setlists) changes the game:
- These recordings contain the *actual* venue acoustics, PA chain, crowd behavior, and recording device effects.

We should treat VEEP as:
- primary *domain* data for robustness
- the gold standard for evaluation

---

## 3) VEEP MP4 pipeline (extract → align → label)

### 3.1 Audio extraction

Use ffmpeg to extract audio:

```bash
ffmpeg -i concert.mp4 -vn -ac 1 -ar 16000 -c:a pcm_s16le concert.wav
```

Standardize:
- mono
- 16 kHz (or 32 kHz if needed)
- consistent loudness normalization

### 3.2 Setlist alignment to create labels

Goal: convert a full-concert waveform into labeled segments:

- Input: (concert.wav, setlist order)
- Output: timestamps for each song (start/end), optionally with confidence

Alignment strategies (in increasing sophistication):

1. **Coarse alignment**
   - detect long applause/banter gaps
   - match approximate durations

2. **Audio-to-audio alignment**
   - embed sliding windows of concert audio
   - embed clean reference tracks
   - dynamic programming / Viterbi to align sequence under the setlist constraint

3. **Human-in-the-loop correction**
   - fast UI to adjust boundaries
   - store corrected annotations as ground truth

### 3.3 Labeled training pairs

Once aligned, we can create training samples:
- positive pairs: (clean segment, VEEP segment of same song)
- also positives: (VEEP segment A, VEEP segment B) from different moments of same song
- negatives: other songs in the setlist (hard negatives are extremely valuable)

---

## 4) Simulation augmentations (still valuable)

Simulation remains important for:
- generalization to unseen venues
- pre-concert reference building (matching expected acoustics)

Augmentations to apply to clean audio:
- crowd noise mixing (non-stationary)
- RIR convolution (venue reverb)
- EQ / channel coloration (PA + phone mic)
- compression/limiting
- clipping / saturation

The key difference now:
- we can **calibrate simulation** using VEEP statistics (RT60 distributions, EQ shapes, typical SNR ranges).

---

## 5) Objectives

Recommended starting point:

- **Supervised contrastive** (track ID) or **NT-Xent** with two views per segment
- Embedding dimension: 128–256
- L2-normalized embeddings, cosine similarity

Hard negative mining:
- within-setlist negatives are naturally hard
- mine confusable songs (similar tempo/harmony) using embeddings during training

---

## 6) Fine-tuning per venue (optional, high upside)

If we know the venue (or can estimate its RIR characteristics), we can optionally fine-tune:

- **Pre-show adaptation**: a short, lightweight fine-tune pass (or reference embedding regeneration) that incorporates the venue’s RIR/EQ.

This is especially attractive because the catalog is tiny and the timeframe is controlled (pre-concert build).

---

## 7) Data flywheel (AUGE advantage)

Every concert attended by AUGE users can generate more training data:

- Users download setlist → build micro-catalog
- During the show, the app can store **ephemeral, privacy-preserving** features or short opt-in snippets
- With setlist ground truth + user confirmations, we obtain labeled examples

Over time:
- model robustness improves
- per-venue adaptation becomes stronger
- the micro-catalog packaging pipeline becomes a defensible system

---

## 8) Deployment constraints (training must respect them)

- Model must convert cleanly: PyTorch → ONNX → CoreML
- Prefer CoreML-friendly operations
- Quantization-aware training may be required
- Evaluate with the full streaming pipeline (temporal consistency) because that is what users experience
