# ROADMAP — Phase plan

This roadmap is structured to de-risk the project by locking down **evaluation** and **data** before investing heavily in modeling.

## Phase 0 — Baseline benchmark (2–7 days)

**Goal:** quantify the gap.

- Build simulation pipeline MVP (enough to produce noisy/reverbed/clipped queries).
- Run `experiments/baseline_shazam_eval.md`.

Deliverables:

- baseline report: accuracy vs SNR/RT60 for ShazamKit, ACRCloud, Chromaprint
- dataset manifest + reproducible scripts (even if rough)

## Phase 1 — Dataset construction pipeline (1–2 weeks)

**Goal:** scalable, reproducible training/eval dataset.

- Curate crowd noise sources (FreeSound IDs + licenses).
- Curate RIRs or implement synthetic RIR generation.
- Implement augmentation pipeline with config files.
- Split into train/val/test with track-level separation.

Deliverables:

- `dataset/` manifest + generation scripts
- fixed evaluation suite for regression testing

## Phase 2 — First embedding model (CNN baseline) (1–2 weeks)

**Goal:** working end-to-end retrieval.

- Implement CNN encoder.
- Train with triplet or contrastive loss.
- Build FAISS index for segment embeddings.

Deliverables:

- training scripts + checkpoints
- evaluation results vs baselines

## Phase 3 — Iterate (transformer, better augmentation) (2–4 weeks)

**Goal:** close the gap under harsh conditions.

- Upgrade model to AST or hybrid conv-transformer.
- Add hard-negative mining.
- Add enhancement module ablations (RNNoise/Demucs).

Deliverables:

- improved accuracy at SNR 0 / -5 dB
- latency/throughput profiling

## Phase 4 — Real-time inference pipeline (1–3 weeks)

**Goal:** streaming recognition <5s.

- Overlapping window inference.
- Temporal aggregation + confidence scoring.
- FAISS index packaging/loading.

Deliverables:

- demo CLI or service
- stable recognition behavior in long streams

## Phase 5 — Field test at real concerts (ongoing)

**Goal:** validate in the real world.

- Define recording protocol: devices, distances, venues.
- Collect labeled snippets (with timestamps/setlists if possible).
- Measure accuracy/latency and iterate.

Deliverables:

- field test report
- failure analysis with actionable fixes

## Phase 6 — AUGE iOS integration spec (1–2 weeks)

**Goal:** integration path.

- Decide deployment: on-device vs hybrid.
- Define model format (CoreML), index format, update mechanism.
- Privacy constraints: ephemeral audio, on-device processing preferred.

Deliverables:

- integration spec doc (API, UX, performance)
- risk register (battery, privacy, latency)
