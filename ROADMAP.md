# ROADMAP — Setlist micro-catalog + CoreML offline plan

This roadmap reflects the current product architecture:

- **On-device, offline iOS (CoreML)**
- **Setlist-based micro-catalog** (20–50 tracks)
- **Studio recordings** as the learning source (clean, isolated)
- **VEEP concert MP4s** as a validation benchmark (test harness only)

---

## Phase 0 — Baseline benchmark (unchanged)

**Goal:** quantify the gap and lock evaluation.

- Build simulation pipeline MVP (noisy/reverbed/clipped queries)
- Evaluate baselines in a **micro-catalog** setting (same setlist constraint):
  - Chromaprint/Echoprint (open)
  - ShazamKit (black box)

**Deliverables**
- accuracy vs SNR/RT60
- streaming time-to-recognition
- reproducible scripts + dataset manifests

---

## Phase 1 — Studio recording ingestion pipeline (pre-concert build foundation)

**Goal:** make “setlist → studio audio → embeddings → micro-catalog” reproducible.

- setlist ingestion
- studio audio acquisition (Apple Music / clean sources) + normalization
- windowing (e.g., 3s with overlap)
- reference embedding generation
- micro-catalog packaging format spec (target: **<10MB**)

**Deliverables**
- studio ingestion + reference embedding builder scripts
- micro-catalog format + example “concert pack”

---

## Phase 1b — VEEP-based validation benchmark

**Goal:** evaluate under real concert conditions (crowd noise, reverb, distortion) without using VEEP as training data.

- ffmpeg extraction to standardized WAV
- known-setlist evaluation harness
- metrics: accuracy, time-to-recognition, stability

**Deliverables**
- `veep_eval/` scripts + reporting
- first VEEP benchmark results

---

## Phase 2 — Train embedding model on studio + simulated concert degradations

**Goal:** learn invariances from augmentation applied to studio tracks.

- start from pretrained backbone (PANNs/OpenL3/MusiCNN)
- train with contrastive objective
- mine hard negatives within setlists

**Deliverables**
- training code + checkpoints
- retrieval metrics on simulated degradations + VEEP benchmark (held-out)

---

## Phase 3 — CoreML conversion + on-device inference test

**Goal:** validate deployment feasibility and performance.

- convert PyTorch → ONNX → CoreML
- benchmark on target device class (e.g., iPhone 15 Pro)
- verify numerical stability (ranking agreement)

**Deliverables**
- CoreML model package
- latency/energy report
- on-device streaming prototype

---

## Phase 4 — At-concert automation + lyric-sync triggering

**Goal:** complete the FOH replacement loop.

- integrate temporal consistency gate (2/3)
- expose “matched song + confidence” events
- automatically trigger lyric sync events in Lyrical (no manual cueing)

**Deliverables**
- end-to-end demo: mic → recognition → lyric sync trigger
- tuning guide for thresholds and anti-flicker

---

## Phase 5 — Field test at a real concert

**Goal:** validate end-to-end in the wild.

- run with known setlist
- measure recognition stability and time-to-recognition
- capture failure modes for next iteration

**Deliverables**
- field test report + error analysis
- prioritized fixes

---

## Phase 6 — AUGE iOS integration

**Goal:** ship inside the AUGE app.

- setlist download + concert pack distribution
- privacy model (on-device first; optional opt-in data collection)

**Deliverables**
- iOS integration spec + APIs
- production performance targets
- rollout plan
