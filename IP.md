# IP — Resonance defensibility (what’s proprietary and why it matters)

This document describes the key intellectual property components of Resonance: a **setlist-constrained, on-device (CoreML) concert song recognizer**.

The goal is not just “a good embedding model.” The goal is an end-to-end system that:
- achieves high accuracy in real concert conditions, and
- is difficult to replicate without the same architecture + data + tooling.

---

## 1) Pre-concert acoustic simulation pipeline

**What it is**

A pre-show process that takes clean reference audio and generates **concert-like reference views** by applying:
- venue RIR convolution (if known or estimated)
- PA / device channel coloration (EQ profiles)
- compression/limiting
- mild clipping/saturation and other nonlinearities

These simulated views are embedded and included in the micro-catalog as additional reference points.

**Why it’s novel / defensible**

Most recognition systems either:
- rely on global fingerprints (not venue-adaptive), or
- do domain augmentation during training only.

Resonance uses simulation *as part of the deployment pipeline* to better match the expected concert channel at inference time.

**What a competitor would need to replicate**

- curated RIR library + venue modeling heuristics
- calibration of simulation parameters to real concert distributions
- tooling that reliably produces improved recognition (not just “more augmented data”)

---

## 2) Setlist-constrained micro-catalog architecture

**What it is**

A reframing of the problem: recognition is performed only over the **known setlist** (20–50 tracks), packaged into an on-device catalog.

**Why it’s novel / defensible**

This changes the optimization target:
- higher robustness is achievable because the candidate set is tiny
- exact similarity search is feasible and stable
- temporal consistency and per-track reference strategies become practical

It is a product + systems insight, not just an ML tweak.

**What a competitor would need to replicate**

- setlist ingestion + clean audio acquisition pipeline
- catalog packaging and update UX
- end-to-end tuning and evaluation in the constrained-catalog setting

---

## 3) CoreML packaged embedding model + index format

**What it is**

A deployable “concert pack” that includes:
- CoreML embedding model (or pinned version)
- compact embedding index (SQLite or flat binary)
- metadata needed for UI and debugging
- signing/encryption and versioning

**Why it’s novel / defensible**

The packaging format and runtime choices determine whether the system can be:
- truly offline
- low-latency
- small enough to download quickly
- stable across devices and OS versions

Competitors can build models, but the practical edge comes from shipping a robust, portable, and updateable offline package.

**What a competitor would need to replicate**

- CoreML conversion expertise (PyTorch→ONNX→CoreML) and validation harnesses
- an index representation optimized for iOS constraints
- secure distribution + integrity checks

---

## 4) VEEP-derived training dataset and labeling methodology

**What it is**

A proprietary dataset of real concert recordings (MP4s) with known setlists, converted into:
- standardized audio
- aligned timestamps per song
- labeled segments for training and evaluation

**Why it’s novel / defensible**

Real concert degradation is hard to capture and label at scale.
VEEP gives access to:
- true venue acoustics and PA behavior
- realistic crowd noise and dynamics
- real recording device artifacts

The labeling methodology (setlist-constrained alignment + human correction) creates high-quality supervised data.

**What a competitor would need to replicate**

- similar access to large volumes of concert recordings with ground truth
- the alignment tooling and annotation workflow
- legal/operational ability to curate and maintain such a dataset

---

## 5) Temporal consistency matching algorithm

**What it is**

A streaming decision layer that stabilizes predictions:
- compute embeddings on 3s windows with 50% overlap
- retrieve top candidate(s) via cosine similarity
- require **2 of the last 3** windows to agree (plus thresholds/margins)

**Why it’s novel / defensible**

In concerts, single-window decisions are noisy and flickery.
Temporal consistency is a simple but powerful systems component that:
- reduces false positives
- improves user trust (“it doesn’t jump around”)
- enables reliable timestamped recognition

**What a competitor would need to replicate**

- extensive tuning on real concert streams
- threshold calibration across devices/venues
- integration into the offline pipeline and UX

---

## 6) Data flywheel (user concerts as training signal)

**What it is**

A feedback loop where each real concert usage can produce additional labeled training data:
- setlist provides a strong prior and weak supervision
- user confirmations + telemetry (privacy-preserving) refine labels
- model and simulation parameters improve over time

**Why it’s novel / defensible**

Global fingerprinting systems optimize for scale, but they don’t get rich supervision from setlist-constrained, per-event usage.
Resonance can improve rapidly because:
- each event is a “mini dataset” with clear candidate set
- per-venue patterns can be learned and reused

**What a competitor would need to replicate**

- product distribution and user base attending concerts
- privacy-safe data collection and labeling mechanisms
- infrastructure to continuously retrain, validate, and ship updated packs

---

## Bottom line

The defensible advantage is the combination of:

- **problem reframing** (setlist micro-catalog)
- **real domain data** (VEEP)
- **pre-concert adaptation** (acoustic simulation)
- **offline deployability** (CoreML packaging + index format)
- **streaming stability** (temporal consistency)
- **continuous improvement loop** (data flywheel)

A competitor can copy individual ideas, but replicating the full system requires rebuilding the data + tooling + deployment pipeline end-to-end.
