# ARCHITECTURE — Proposed system design

Goal: **recognize songs from live concert audio** in <5 seconds, under 80 dB+ crowd noise, reverberant venues, PA coloration, and phone mic distortion.

## 1) End-to-end pipeline

### 1.1 Streaming ingest

- Input: phone microphone PCM stream (16 kHz or 32 kHz recommended; 16 kHz is sufficient for ID but may lose some high-frequency cues).
- Chunking: 20–40 ms frames for feature extraction; aggregate into overlapping windows.

### 1.2 Optional enhancement (denoise / dereverb)

Two options depending on latency/compute budget:

- **RNNoise**: low-latency noise suppression.
- **Demucs (or similar)**: heavier, potentially better separation but introduces artifacts.

Important: treat enhancement as **a module to evaluate**, not an assumption. If enhancement hurts embeddings, remove it and rely on training augmentations.

### 1.3 Feature extraction

- STFT → mel filterbank → log-mel spectrogram
- Typical config (starting point):
  - sample rate: 16 kHz
  - window length: 25 ms
  - hop: 10 ms
  - mel bins: 64 or 96
  - normalization: per-window mean/var

### 1.4 Embedding encoder

Two candidate model families:

- **CNN encoder** (fast baseline): EfficientNet-like blocks or a ResNet-ish spectrogram CNN.
- **Transformer encoder** (stronger, heavier): **AST — Audio Spectrogram Transformer** (Gong et al., 2021).

Output:

- embedding dimension \(d\) (e.g., 128–512)
- L2-normalize embeddings to enable cosine similarity.

Training objective: triplet loss or NT-Xent / supervised contrastive (see TRAINING.md).

### 1.5 Retrieval index

- Use **FAISS** Approximate Nearest Neighbor index.
- Reference: precompute embeddings for a catalog of tracks.

Design choice (critical):

- **Segment-level indexing:** embed many short windows per track (e.g., every 0.5s). Pros: robust to partial queries. Cons: larger index.
- **Track-level indexing:** single embedding per track (requires pooling). Pros: smaller index. Cons: may lose temporal specificity.

Recommendation: start with **segment-level** in Phase 2–4 for best accuracy.

### 1.6 Decision logic (confidence + temporal consistency)

Single-window nearest neighbor is not enough under heavy noise. We require:

- similarity threshold: cosine >= τ
- **temporal consistency** over N consecutive overlapping windows:
  - same predicted track ID for ≥K of N
  - consistent offset drift within tolerance

We also maintain a short rolling buffer (e.g., last 5 seconds) to compute a stable decision.

## 2) ASCII diagram

```
  Microphone stream
        |
        v
  [PCM frames]
        |
        v
  Optional enhancement
  (RNNoise / Demucs)
        |
        v
  log-mel spectrogram
        |
        v
  Embedding encoder f(.)  --->  e_t in R^d (L2-normalized)
        |
        +------------------------------+
                                       |
                                       v
                               FAISS ANN search
                               (segment embeddings)
                                       |
                                       v
                        Top-k candidates + similarities
                                       |
                                       v
                         Temporal aggregator / voting
                    (consistency across overlapping windows)
                                       |
                                       v
                           Predicted track + confidence
```

## 3) Latency budget (<5s)

Target: recognition within **5 seconds** end-to-end.

Suggested operating point:

- window length: 2.0s
- hop: 0.5s
- decision: require 3 consistent windows (≈ 3.0s of audio after warmup)

Compute budget considerations:

- Feature extraction: CPU ok.
- CNN encoder: feasible on mobile CPU/NN accelerator.
- AST: may require on-device NN accelerator or server-side.

## 4) Interfaces & modules

- `audio_io`: streaming capture + resampling
- `frontend`: STFT/mel/log + normalization
- `enhancement`: RNNoise/Demucs wrappers
- `model`: encoder inference (PyTorch / ONNX / CoreML)
- `index`: FAISS build/load/query
- `matcher`: temporal aggregation + confidence

## 5) Failure modes to explicitly test

- SNR below 0 dB
- heavy clipping
- long RT60 (reverb)
- crowd chants with tonal components
- strong bass boosting by PA

These should be part of the evaluation suite to avoid “lab wins” that don’t generalize.
