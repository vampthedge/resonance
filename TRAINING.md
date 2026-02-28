# TRAINING — Data, objectives, augmentation, evaluation

This document specifies how to build a training pipeline for **concert-robust song embeddings**.

## 1) Problem formulation

We learn an embedding model \( f_\theta \) that maps an audio window \(x\) to an embedding \(e\in\mathbb{R}^d\) such that:

- same track → embeddings close
- different tracks → embeddings far

At inference, we do ANN retrieval against a catalog of reference embeddings.

## 2) Data strategy

### 2.1 Base corpus

Start with legally usable datasets (for research/prototyping):

- FMA (various sizes)
- MTG-Jamendo

Also maintain a private “target catalog” later for AUGE integration.

### 2.2 Windowing

- Sample windows of length 1–3 seconds.
- Random offsets per epoch.
- Consider multi-window sampling per track to cover intros/outros.

## 3) Concert degradation augmentations

We create positives by generating a degraded view \(\tilde{x}\) from clean \(x\).

### 3.1 Crowd noise injection

- Mix crowd noise recordings (e.g., FreeSound) at SNR levels in {-10, -5, 0, +5, +10} dB.
- Use non-stationary segments: chants, claps, whistles.

Implementation: scale noise to target SNR based on RMS or LUFS.

### 3.2 Reverberation via impulse response convolution

- Convolve clean audio with room impulse responses (RIR).
- Use a distribution of RT60 values (small club to arena).

Tools:

- `pyroomacoustics` (simulation)
- public RIR datasets (to be curated)

### 3.3 EQ / channel coloration

- Random parametric EQ filters to mimic PA tuning + device mic response.
- Random low-shelf/high-shelf + peaking filters.

### 3.4 Dynamic range compression / limiting

- Apply compressor with randomized threshold/ratio/attack/release.
- Add brickwall limiting to simulate mastered live feed.

### 3.5 Mic clipping / saturation

- Hard clipping at randomized thresholds.
- Soft saturation (tanh / arctan waveshaping) to mimic analog-ish distortion.

### 3.6 Time/frequency perturbations (use cautiously)

- Small time-stretch (±1–2%) and pitch shifts (±10–20 cents) to model drift.
- Excessive warps can destroy identity; tune with care.

## 4) Objectives

### 4.1 Triplet loss

- Anchor: degraded window
- Positive: another degraded view of same underlying clean window (or clean version)
- Negative: different track window

Key: **hard negative mining** improves retrieval quality.

### 4.2 NT-Xent / contrastive loss (SimCLR-style)

- For each clean window, generate two degraded views; treat as positives.
- Use batch negatives.

Pros: simple; cons: may need large batches.

### 4.3 Supervised contrastive

- Use track ID labels to define positives.
- Can stabilize training if multiple windows per track per batch.

## 5) Triplet mining strategy (hard negatives)

Hard negatives should be “confusable”:

- same genre
- similar tempo
- similar key / harmonic profile
- similar instrumentation

Implementation ideas:

- precompute simple audio descriptors (tempo, chroma) and sample negatives from nearest neighbors
- online mining within a batch: choose negatives with highest cosine similarity that are not same track

## 6) Training infrastructure

Recommended stack:

- PyTorch + PyTorch Lightning
- torchaudio / librosa for I/O and transforms
- FAISS for evaluation retrieval at scale

Experiment tracking:

- Weights & Biases or MLflow (optional)

## 7) Evaluation

### 7.1 Retrieval metrics

- Top-1 / Top-5 accuracy
- Mean Reciprocal Rank (MRR)

Evaluate vs SNR:

- -5 dB, 0 dB, +5 dB, +10 dB (and optionally -10 dB)

### 7.2 Streaming recognition metrics

- time-to-first-correct (seconds)
- false positive rate under noise-only segments
- stability: number of track flips per minute

### 7.3 Ablations

- with/without enhancement module (RNNoise/Demucs)
- CNN vs AST
- effect of each augmentation class

## 8) Practical notes for deployment

- Model export: ONNX → CoreML (iOS)
- Quantization: int8 where feasible
- Keep embedding dimension modest (128–256) to reduce index memory.

---

*Deliverable definition:* A successful Phase-2 model should beat Chromaprint and approach/beat ShazamKit on the simulated concert benchmark at SNR 0 dB and +5 dB, while maintaining usable latency (<5s).