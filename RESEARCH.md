# RESEARCH — Literature review & study plan

This document is a rigorous (but living) literature review for building a **concert-robust** song recognition system.

## 1) Classical audio fingerprinting

### 1.1 Shazam / landmark hashing

**Core reference**

- Wang, Avery Li-Chun (2003). **“An Industrial Strength Audio Search Algorithm.”** *Proceedings of ISMIR 2003.*

**Method summary**

- Compute STFT magnitude spectrogram.
- Detect **salient spectral peaks**.
- Create **hashes** from peak pairs (anchor peak + target peak) using (f1, f2, Δt).
- Query: generate hashes, look up collisions in DB, vote for track IDs and time offsets.

**Why it works well**

- Hashes are compact.
- Matching is fast at scale.
- Robust to moderate noise and compression when peaks remain stable.

### 1.2 Echoprint

- Echoprint is an open fingerprinting system inspired by landmark hashing; it uses time–frequency landmarks and compact codes.
- Practical note: Echoprint’s open implementation makes it a useful baseline for controlled evaluation; however, its fingerprint stability is still limited under severe reverberation and nonlinear distortion.

### 1.3 Chromaprint / AcoustID

- Chromaprint uses chroma-like features (pitch class energy) and uses sequence matching to compute a fingerprint.
- Strengths: robustness to some equalization and mild noise; weaknesses: polyphonic mixtures, heavy distortion, and short query segments can reduce discriminability.

## 2) Why fingerprinting fails in live concert conditions

Concert recordings violate the assumptions behind stable peak pairing / chroma stability.

### 2.1 Severe, non-stationary noise

- Crowd noise is highly non-stationary and broadband.
- It introduces spurious peaks and masks musical harmonics.

**Failure mode:** fewer reliable landmarks → fewer correct hash votes; more spurious collisions.

### 2.2 Reverberation and temporal smearing

- Reverb spreads energy in time; reflections create comb filtering and interfere with direct sound.

**Failure mode:** peak locations shift; peak-pair Δt becomes unstable; temporal offset voting weakens.

### 2.3 Device/PA channel effects

- PA EQ + venue acoustics + microphone frequency response produce a compound transfer function.

**Failure mode:** relative spectral salience changes; peaks used for hashing may disappear.

### 2.4 Nonlinear distortion

- Phone mic saturation, clipping, and aggressive compression produce harmonics and broadband artifacts.

**Failure mode:** fingerprint becomes dominated by artifacts rather than song identity.

### 2.5 Scale & collision considerations

- At large catalog scale, any approach relying on a limited hash space must address collision rates.
- Under degradation, the effective entropy of the query hashes decreases (more ambiguity), increasing collision probability.

## 3) Alternative approaches: learned embeddings & neural fingerprints

### 3.1 Deep metric learning for audio retrieval

**Relevant families**

- Triplet loss (anchor, positive, negative)
- Contrastive loss / NT-Xent (SimCLR-style)
- Supervised contrastive learning

**Key idea:** learn invariances directly by training on **multiple views** of the same content.

Practical implication for concerts:

- generate positives as (clean window, degraded window) and (degraded1, degraded2)
- generate hard negatives from similar tempo/key/timbre songs

### 3.2 Neural audio fingerprinting

Neural approaches replace sparse hashing with a network that outputs either:

- a dense embedding for ANN search, or
- a compact binary code optimized for retrieval.

**Paper to prioritize**

- Chang, J. H. et al. (2021). **Neural Audio Fingerprinting.**
  - (Add the exact venue/arXiv/DOI once confirmed; search and pin the canonical citation before publication-quality writing.)

Study questions:

- What invariances are learned vs engineered?
- How do they build positives/negatives?
- How do they evaluate robustness to noise/reverb/time-scale changes?
- Do they learn binary codes or continuous embeddings?

### 3.3 Representation learning backbones

- Gong, Yuan, Chung, Yu-An, & Glass, James (2021). **“AST: Audio Spectrogram Transformer.”** *Interspeech 2021.*
- CNN baselines used widely in music tagging and audio classification:
  - Choi, Keunwoo et al. (2017). **“Convolutional Recurrent Neural Networks for Music Classification.”** *ICASSP 2017.*

Relevance:

- AST provides strong general-purpose audio representations; can be fine-tuned for metric learning.
- CNNs can be faster/cheaper and easier to deploy on-device.

## 4) Front-end enhancement: denoising, dereverberation, beamforming

These can be used either:

- as a preprocessing stage to increase SNR for embedding extraction, or
- as augmentations during training so the embedding model learns to ignore residual artifacts.

### 4.1 Spectral subtraction / Wiener filtering

- Classical denoising; works when noise is quasi-stationary.
- Concert noise is not stationary → limited but still a baseline.

### 4.2 RNNoise

- Real-time neural noise suppression.
- Strength: low-latency and practical on CPU.

### 4.3 Demucs / music source separation

- Can suppress crowd noise by separating “music” vs “noise” components.
- Risk: artifacts from separation can harm fingerprint stability; must be evaluated end-to-end.

### 4.4 Beamforming (multi-mic)

If hardware allows (e.g., phone mic arrays), beamforming could:

- attenuate crowd noise
- reduce reverberant field

But: phone geometry constraints + OS access may limit feasibility.

## 5) Datasets for training and evaluation

### 5.1 Public music datasets (clean)

- **FMA** (Free Music Archive): diverse genres, multiple subsets (small/medium/large).
- **MTG-Jamendo**: music tagging dataset with licensing suited for research.
- **MusicCaps**: captions dataset for music/audio understanding (less direct for ID, but useful for representation learning).

### 5.2 How to get “concert-degraded” training data

Two strategies:

1. **Simulation**: take clean tracks and apply degradations (noise + RIR + device distortion) → produces paired clean/degraded examples.
2. **Real recordings**: crowd-sourced concert snippets paired with setlists / known IDs (harder labeling; legal and privacy considerations).

Simulation is the recommended Phase 1 path because it is reproducible and label-clean.

## 6) Evaluation methodology

We want metrics that reflect the deployment scenario:

- **Top-1 accuracy vs SNR** at {-5, 0, +5, +10} dB
- **Time-to-recognition** under streaming (latency budget <5s)
- **Robustness vs reverb (RT60)** and device/mic responses
- **Confusion matrix**: which tracks are confused (genre/tempo similarity)

Baselines to include:

- Chromaprint (AcoustID-style)
- (Optional) Echoprint if feasible
- Commercial: ShazamKit, ACRCloud (as black-box baselines)

## 7) Key papers to add (prioritized reading list)

1. Wang (2003) — Shazam landmark hashing.
2. AST (Gong et al., 2021).
3. Neural audio fingerprinting (Chang et al., 2021) — confirm canonical citation.
4. Metric learning / contrastive learning foundations:
   - Schroff, F., Kalenichenko, D., & Philbin, J. (2015). **FaceNet: A Unified Embedding for Face Recognition and Clustering.** CVPR.
   - Chen, T. et al. (2020). **A Simple Framework for Contrastive Learning of Visual Representations (SimCLR).** ICML. (Conceptual transfer.)
   - Khosla, P. et al. (2020). **Supervised Contrastive Learning.** NeurIPS.

## 8) Open questions (to resolve early)

- **Catalog constraints:** do we embed full-length tracks offline, or per-segment reference windows?
- **Window size:** 1.0–3.0s windows vs longer context; effect on latency and accuracy.
- **On-device feasibility:** model size, CPU/GPU availability, energy budget.
- **Licensing & privacy:** storing user mic audio? how much to keep? how to anonymize?

---

*Note:* This file intentionally includes a “citation TODO” for the neural fingerprinting paper. Before using this repo as an external-facing research artifact, we should confirm exact bibliographic details (authors, venue, DOI/arXiv ID).