# ARCHITECTURE — Setlist-based, on-device concert song recognition (CoreML)

Resonance is an audio recognition system optimized for **live concerts** where the catalog is known **ahead of time** (the setlist). Instead of competing with internet-scale fingerprinting, we build a **micro-catalog** (20–50 tracks) and run **fully offline on iOS** using **CoreML**.

## Design principles

- **Micro-catalog retrieval beats global search**: With 20–50 candidates, we can use stronger learned embeddings + consistency logic and get higher accuracy under heavy degradation.
- **Offline-first (CoreML)**: At-concert recognition must work with no network and low latency.
- **IP moat is the packaging + simulation pipeline**: The “secret sauce” is how we (a) simulate venue/channel effects pre-show, and (b) package a tiny, fast, CoreML-compatible catalog + index.

---

## 1) Two pipelines: pre-concert build vs at-concert inference

### 1.1 Pre-concert pipeline (before the show)

Runs on device or on a server *before* the concert. Output is a small bundle (<10MB typical) containing:
- CoreML embedding model (or version reference)
- micro-catalog metadata
- reference embeddings (+ optional per-track/per-segment embeddings)
- an index optimized for fast cosine similarity search

Steps:

1. **Input: setlist**
   - Song titles + artists (typically 20–50 songs)
   - Optionally include alternate versions / live edits if known

2. **Fetch clean studio audio**
   - Apple Music / YouTube Music APIs, or user-provided local files
   - Normalize sample rate and loudness

3. **Optionally simulate venue acoustics**
   - Apply the expected venue’s **room impulse response (RIR)** if known
   - Apply PA/mic coloration profiles (EQ curves), compression, mild clipping
   - Goal: generate “concert-like” reference views that match what the phone hears

4. **Embed reference audio**
   - Window audio into ~3s segments (50% overlap)
   - log-mel frontend → embedding model → L2-normalized vectors

5. **Pack to CoreML-compatible embedding index**
   - Store embeddings + metadata in a compact format:
     - SQLite (fast lookup, simple updates) **or**
     - flat binary file (minimal overhead)
   - Index can be brute-force cosine over <=50 tracks, or precomputed matrices for SIMD/ANE-friendly ops

6. **Bundle onto device**
   - Deliver as an encrypted/signed “concert pack”
   - Includes setlist metadata for UI (track names, artists, artwork pointers)

### 1.2 At-concert inference pipeline (during the show, offline)

Runs entirely on-device with strict latency constraints.

1. **Mic input**
   - Continuous audio capture

2. **Noise suppression (CoreML)**
   - RNNoise-style low-latency suppression **or** Demucs-lite (if feasible)
   - Treated as an ablation: keep only if it improves end-to-end recognition

3. **Log-mel spectrogram**
   - Sliding windows: **3.0s** window, **50% overlap** (hop = 1.5s)
   - Per-window normalization

4. **Embedding model (CoreML)**
   - CNN / MobileNet-style spectrogram encoder
   - Output: L2-normalized embedding (e.g., 128–256 dims)

5. **Similarity search against micro-catalog**
   - Cosine similarity against reference embeddings
   - With a micro-catalog, we can do exact search cheaply and deterministically

6. **Temporal consistency gate (anti-flicker)**
   - Require **2 of the last 3** consecutive windows to agree on the same song
   - Optionally enforce similarity margin vs runner-up and a minimum confidence threshold

7. **Output**
   - Matched song + confidence + timestamp (and optional offset estimate)

---

## 2) ASCII architecture diagram

```
                 ┌──────────────────────────────────────────────────────┐
                 │                 PRE-CONCERT (build)                   │
                 └──────────────────────────────────────────────────────┘

   Setlist (20–50)      Clean audio fetch         Optional acoustics sim
 (title+artist) ───▶ (Apple/YouTube/local) ───▶ (RIR + EQ + comp + clip)
         │                      │                           │
         v                      v                           v
  track metadata        normalized audio                simulated views
         │                      │                           │
         └───────────────┬──────┴───────────────┬───────────┘
                         v                      v
                 log-mel windows         log-mel windows
                         │                      │
                         v                      v
                CoreML embedding model   CoreML embedding model
                         │                      │
                         └───────────────┬──────┘
                                         v
                            reference embeddings (L2)
                                         │
                                         v
                 CoreML-compatible micro-index (SQLite/flat)
                                         │
                                         v
                           "Concert Pack" bundled to iPhone


                 ┌──────────────────────────────────────────────────────┐
                 │              AT-CONCERT (offline inference)           │
                 └──────────────────────────────────────────────────────┘

   iPhone mic ─▶ RNNoise/Demucs-lite ─▶ log-mel (3s, 50% overlap) ─▶ CoreML
                                                                    embed
                                                                      │
                                                                      v
                                                    cosine similarity vs
                                                    micro-catalog index
                                                                      │
                                                                      v
                                             temporal consistency (2/3)
                                                                      │
                                                                      v
                                               song + confidence + time
```

---

## 3) What changes vs Shazam-style global fingerprinting

- We **do not** need internet-scale indexing, collision handling, or global uniqueness.
- The task becomes **robust retrieval within a tiny candidate set** under severe degradation.
- The system can afford stronger invariances (learned embeddings, simulation alignment) because the compute and memory budget is small.

---

## 4) IP moat (why this is hard to copy)

The defensibility is not “a better embedding model” alone.

Our moat is the **pre-concert build system**:
- **Acoustic simulation & adaptation**: generating reference embeddings that match the expected venue and device channel.
- **CoreML packaging**: shipping a compact, fast, offline-ready index + model bundle.
- **Setlist constraint**: problem reframing that enables higher accuracy and better UX.

A competitor without our data (VEEP live recordings) and without our packaging pipeline would have to rebuild:
- venue/channel simulation heuristics and RIR library
- on-device index format and update distribution
- end-to-end tuning for temporal stability in real concerts
