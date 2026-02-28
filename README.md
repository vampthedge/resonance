# resonance

**Robust song recognition for live concerts (crowd noise, reverb, distortion).**

This repository lays the scientific + engineering groundwork for an **AI-powered, embedding-based music recognition system** that is explicitly trained to handle the failure modes of classical audio fingerprinting in real-world concert recordings.

> Working thesis: **learned audio embeddings trained on concert-degraded audio** (noise/reverb/PA coloration/mic saturation) will outperform **spectral-peak hashing** (e.g., Shazam-style fingerprints) under severe acoustic corruption.

## Motivation: why Shazam-style fingerprinting fails in concerts
Most deployed music recognition systems are based on variants of the classic approach introduced by:

- Avery Li-Chun Wang, **“An Industrial Strength Audio Search Algorithm”** (ISMIR 2003).

These systems typically:

1. Convert audio to a time–frequency representation (STFT).
2. Detect salient spectral peaks.
3. Hash **pairs of peaks** (time/frequency anchors) into compact fingerprints.
4. Match fingerprints against a database with voting / offset consistency.

This approach is excellent for **clean-ish** recordings (radio, streaming, TV), but live concerts induce conditions that break its assumptions:

- **Crowd noise (non-stationary, broadband):** reduces SNR and introduces competing peaks.
- **Reverberation & spectral smearing:** peaks blur in time/frequency; peak locations shift.
- **PA system + room coloration:** EQ changes alter relative spectral energy and peak salience.
- **Dynamic compression / limiting:** flattens dynamics; reduces contrast that peak-pickers rely on.
- **Mic saturation / clipping:** introduces harmonics and broadband artifacts that create false peaks.
- **Overlapping sources:** vocals + instruments + crowd + reflections mix in ways not seen in studio.

Result: fewer stable peak pairs, more collisions/false votes, and reduced temporal-consistency signals.

## Core hypothesis
Instead of hashing a sparse set of spectral peak pairs, we learn an embedding function:

> \( f(\text{audio window}) \rightarrow \mathbb{R}^d \)

trained with **metric learning** so that:

- windows from the **same song** (clean, live, different phones, different venues) land **close** in embedding space
- windows from **different songs** land **far apart**

Key idea: we explicitly generate (or collect) **concert-degraded views** of the same underlying track and train with contrastive objectives (triplet loss / NT-Xent / supervised contrastive).

## High-level system architecture
At runtime we want <5s recognition even at very low SNR.

1. **Ingest**: microphone stream
2. **Denoise / dereverb (optional)**: RNNoise / Demucs / spectral gating
3. **Feature**: log-mel spectrogram
4. **Encoder**: CNN or transformer (e.g., EfficientNet-style convnet, AST)
5. **Index**: FAISS ANN search over reference embeddings
6. **Decision**: similarity + temporal consistency across overlapping windows

See [ARCHITECTURE.md](ARCHITECTURE.md) for a concrete proposal.

## Research roadmap (phases)
This is intentionally staged to avoid premature modeling.

- **Phase 0 — Baseline**: quantify failure of ShazamKit / ACRCloud / Chromaprint on simulated concert audio.
- **Phase 1 — Data pipeline**: build reproducible concert-degradation simulator (noise, RIR, saturation, compression).
- **Phase 2 — First model**: train a small CNN embedding model; establish robust evaluation.
- **Phase 3 — Iterate**: stronger augmentations, transformer encoder, hard-negative mining.
- **Phase 4 — Retrieval + latency**: FAISS index + streaming inference + temporal aggregation.
- **Phase 5 — Field tests**: real concerts; measure recognition latency and top-1 under uncontrolled conditions.
- **Phase 6 — Productization**: integrate into AUGE iOS app (on-device or hybrid).

See [ROADMAP.md](ROADMAP.md).

## Repository structure

- `RESEARCH.md` — literature review + key papers
- `ARCHITECTURE.md` — end-to-end system design
- `TRAINING.md` — training pipeline and evaluation
- `ROADMAP.md` — phase plan and milestones
- `experiments/` — structured experiment plans
  - `baseline_shazam_eval.md`
  - `dataset_simulation.md`

## Non-goals (for now)

- Training a production-grade model in this repo immediately.
- Solving licensing/distribution of commercial catalogs (treated as an integration concern).

## References (starter)
- Wang, A. L.-C. (2003). *An Industrial Strength Audio Search Algorithm*. ISMIR.
- Choi, K., Fazekas, G., Sandler, M., & Cho, K. (2017). *Convolutional Recurrent Neural Networks for Music Classification*. ICASSP.
- Gong, Y., Chung, Y.-A., & Glass, J. (2021). *AST: Audio Spectrogram Transformer*. Interspeech.
- Chang, J. H. et al. (2021). *Neural Audio Fingerprinting*. (See `RESEARCH.md` for details and related work.)
