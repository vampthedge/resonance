# RESEARCH — Literature review & study plan

This document tracks research directions for building a **concert-robust** song recognition system optimized for:

- **Setlist-based micro-catalogs** (20–50 tracks)
- **On-device, offline iOS inference** (CoreML)
- Real-world concert degradation (noise, reverb, PA coloration, clipping)

---

## FOH Operator Replacement: prior art and novelty

**Current state:** in many concert workflows, a **Front of House (FOH) operator** manually cues show elements (lyrics, lighting, visuals) by watching the stage and triggering events.

**Gap:** while there are many music recognition products and many show-control systems, there is no widely deployed system that **autonomously recognizes live audio against a known setlist** and **triggers real-time lyric synchronization** end-to-end with **no manual cueing**.

**Resonance’s novelty:**

- **Constrained recognition (closed-world):** the setlist defines the candidate set, enabling robust retrieval in a tiny catalog.
- **Offline CoreML inference:** reliable operation in venues with poor connectivity, low latency, privacy-friendly.
- **Automatic event triggering:** recognition is directly coupled to lyric-sync events, eliminating the FOH cueing role for lyrics.

---

## 1) Classical audio fingerprinting (context)

### 1.1 Shazam / landmark hashing

- Wang, Avery Li-Chun (2003). **“An Industrial Strength Audio Search Algorithm.”** ISMIR 2003.

Summary:
- STFT magnitude → salient peaks → hash peak pairs (f1, f2, Δt)
- Query hashes collide with DB entries; vote on track + offset

Strengths:
- Extremely fast at global scale
- Compact representation

Weaknesses in concerts:
- Severe reverb + crowd noise + channel distortion destabilize peak constellations

### 1.2 Echoprint / Chromaprint

Useful as baselines but similarly affected by:
- temporal smearing from reverb
- non-stationary broadband noise
- nonlinear distortion / clipping

---

## 2) Micro-catalog retrieval vs large-scale fingerprinting

### Why micro-catalog is a different (easier) problem

With a **known candidate set**, the system changes from:

- **Global identification**: “Which song out of tens of millions?”

to:

- **Constrained retrieval**: “Which of these 20–50 songs is playing right now?”

Implications:

- We can use **learned embeddings** and exact cosine similarity search with negligible latency.
- Collision rates and “hash entropy” constraints are far less important.
- We can incorporate **temporal consistency** (2/3 sliding windows) to suppress false positives.
- We can precompute **multiple reference views** per song (clean + simulated venue/channel), which is not feasible at global catalog scale.

Expected outcome:
- Better accuracy under concert degradation, and more stable streaming behavior.

---

## 3) Learned embeddings / neural fingerprints for robust retrieval

### 3.1 Metric learning objectives

- Triplet loss
- NT-Xent / contrastive learning
- Supervised contrastive learning

Key idea:
- Train invariances by pairing **clean** and **concert-degraded** views of the same segment.

### 3.2 Transfer learning candidates (starting points)

We should start from a pretrained audio backbone and fine-tune for retrieval:

- **PANNs** (Pretrained Audio Neural Networks)
- **OpenL3**
- **MusiCNN**

Fine-tuning strategy:
- Replace classifier head with projection head → embedding
- Train on (clean, degraded) positives and hard negatives from similar music

### 3.3 Validation under real concert conditions (VEEP)

Most of the value comes from matching the deployment domain:
- crowd noise
- long RT60
- PA EQ + compression
- phone mic distortion

VEEP concert recordings (with known setlists) are a strong **held-out evaluation harness** to measure whether the full system (including temporal gating) holds up under real concert degradation.

---

## 4) CoreML deployment for audio embeddings (iOS offline)

### 4.1 Why CoreML

- Runs locally (privacy + latency + reliability)
- Uses Apple Neural Engine (ANE) when supported
- Enables shipping an offline “concert pack” (model + micro-index)

### 4.2 Typical conversion path

1. Train in **PyTorch**
2. Export to **ONNX**
3. Convert to **CoreML** using `coremltools`

Notes:
- Prefer ops that CoreML supports well (conv, pooling, matmul, activations)
- Quantization (e.g., 16-bit / 8-bit) may be required for speed and size
- Validate numerics: cosine similarity ranking should be stable across conversions

### 4.3 Runtime considerations

- log-mel frontend can be CPU (Accelerate/BNNS) or CoreML if beneficial
- 3s windows with 50% overlap are a good starting point for <5s recognition

---

## 5) Study questions / experiments

- Micro-catalog size sensitivity: 10 vs 50 vs 200 tracks
- Best reference strategy:
  - single embedding per track (pooled)
  - multi-segment embeddings per track
  - clean-only vs clean+simulated reference bank
- Impact of enhancement (RNNoise / Demucs-lite) on end-to-end accuracy
- CoreML conversion drift: PyTorch vs CoreML top-1 agreement and similarity calibration

---

## 6) Key evaluation framing

We should measure:
- top-1 accuracy vs SNR / RT60
- time-to-recognition in streaming
- false positive rate during inter-song noise
- stability (track flips/min)

Baselines:
- Chromaprint / Echoprint (open)
- ShazamKit (black box) in the same constrained setlist setting
