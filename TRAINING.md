# TRAINING — Data, augmentation, and evaluation

This document defines how we train an embedding model that is robust to **live concert audio** while remaining deployable **on-device via CoreML**.

**Key correction:**
- **Training data:** studio recordings (clean, isolated)
- **VEEP concert MP4s:** validation/testing only (held-out benchmark), **not training data**

---

## 1) Problem formulation

We learn an embedding model \( f_\theta \) that maps an audio window \(x\) to an embedding \(e\in\mathbb{R}^d\) such that:

- same song → embeddings close
- different songs → embeddings far

Inference is **retrieval**: compute an embedding from the mic stream and match via cosine similarity against a **setlist micro-catalog**.

---

## 2) Training data (studio recordings)

### 2.1 Sources

- Studio tracks fetched from Apple Music (or other clean sources), or provided locally
- These are the “canonical” representations of what each song sounds like in perfect conditions

### 2.2 Windowing

- Segment into short overlapping windows (e.g., **3.0s** with **50% overlap**)
- Standardize sample rate and loudness

---

## 3) Augmentation: simulate concert degradation

We train robustness by applying concert-like corruption to studio audio:

- **Crowd noise** (non-stationary, broadband)
- **Room impulse responses (RIR)** / reverb (venue acoustics)
- **Compression/limiting** (PA chain)
- **EQ / channel coloration** (PA + phone mic)
- **Mic clipping / saturation**

The model learns: **“even when this song sounds terrible live, it still matches the studio embedding.”**

---

## 4) Objectives

Recommended starting point:

- **Supervised contrastive** (track ID) or **NT-Xent** with two augmented views per segment
- Embedding dimension: 128–256
- L2-normalized embeddings; cosine similarity for retrieval

Hard negative mining:
- within-setlist negatives are naturally hard
- mine confusable songs using online retrieval during training

---

## 5) Validation with VEEP (held-out test set)

VEEP concert MP4s are used as a **benchmark** to answer:

> does the system still recognize songs correctly when there’s crowd noise, reverb, distortion?

Workflow:

1. Download concert MP4s where the **setlist is known**
2. Extract audio (ffmpeg)
3. Run the full streaming pipeline (windowing → embedding → cosine similarity → temporal gate)
4. Measure:
   - top-1 accuracy
   - time-to-recognition
   - false positive rate during inter-song noise

**Important:** VEEP is a test harness and reliability check. It is not used to generate training pairs or supervise the model.

---

## 6) Deployment constraints (training must respect them)

- Model must convert cleanly: PyTorch → ONNX → CoreML
- Prefer CoreML-friendly operations
- Quantization-aware training may be required
- Evaluate with the full streaming pipeline (temporal consistency) because that is what users experience

