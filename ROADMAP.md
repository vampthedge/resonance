# ROADMAP — Setlist micro-catalog + CoreML offline plan

This roadmap reflects the current product architecture:

- **On-device, offline iOS (CoreML)**
- **Setlist-based micro-catalog** (20–50 tracks)
- **VEEP concert MP4s** as real degraded ground truth

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

## Phase 1 — VEEP pipeline (extract + label concert audio)

**Goal:** turn concert MP4 downloads into labeled training/eval data.

- ffmpeg extraction to standardized WAV
- setlist alignment to get song timestamps
- human-in-the-loop tool to correct boundaries

**Deliverables**
- `veep/` pipeline scripts
- labeled dataset format + examples
- first VEEP-based evaluation suite

---

## Phase 2 — Train embedding model on clean vs VEEP-degraded pairs

**Goal:** learn invariances from real degradation.

- start from pretrained backbone (PANNs/OpenL3/MusiCNN)
- train with contrastive objective
- mine hard negatives within setlists

**Deliverables**
- training code + checkpoints
- retrieval metrics on VEEP holdout concerts

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

## Phase 4 — Pre-concert pipeline (setlist → micro-catalog → CoreML package)

**Goal:** generate a small offline “concert pack”.

- setlist ingestion
- clean audio fetch + normalization
- optional acoustic simulation (RIR/EQ)
- reference embedding generation
- micro-index packaging (SQLite/flat) + signing

**Deliverables**
- micro-catalog builder tool
- concert pack format spec
- end-to-end demo: build pack → run offline recognition

---

## Phase 5 — Field test at a real concert

**Goal:** validate end-to-end in the wild.

- run with known setlist
- measure recognition stability and time-to-recognition
- capture failure modes for next training iteration

**Deliverables**
- field test report + error analysis
- prioritized fixes

---

## Phase 6 — AUGE iOS integration (replace ShazamKit with Resonance)

**Goal:** ship inside the AUGE app.

- replace ShazamKit flow
- implement setlist download + concert pack distribution
- privacy model (on-device first; optional opt-in data collection)

**Deliverables**
- iOS integration spec + APIs
- production performance targets
- rollout plan
