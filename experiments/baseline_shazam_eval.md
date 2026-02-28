# Experiment: baseline benchmark (ShazamKit / ACRCloud / Chromaprint)

Purpose: establish the **performance baseline we must beat** in a controlled setting that approximates live concert acoustics.

## 1) Systems to benchmark

1. **ShazamKit** (Apple)
2. **ACRCloud** (commercial)
3. **Chromaprint** (open baseline)
4. (Optional) **Echoprint** if practical

## 2) Test set construction

### 2.1 Track selection

- Choose N tracks (start: N=500; scale later).
- Balance genres (pop/rock/hip-hop/electronic) and production styles.

### 2.2 Query generation (simulated concert)

For each track:

- sample M query windows at random offsets
  - window lengths: 2s, 3s, 5s
- apply degradation pipeline (see `dataset_simulation.md`):
  - crowd noise (various SNR)
  - RIR convolution (various RT60)
  - EQ coloration
  - compression/limiting
  - mic saturation/clipping

Generate a factorial design (keep manageable):

- SNR ∈ {-5, 0, +5, +10} dB
- RT60 ∈ {0.3s, 0.8s, 1.5s, 2.5s}
- clipping ∈ {off, mild, heavy}

## 3) Metrics

### 3.1 Identification metrics

- Top-1 accuracy
- No-match rate (system returns nothing)
- Wrong-match rate

### 3.2 Latency

- Time to result for each query
- Variance across conditions

### 3.3 Calibration

If APIs return confidence scores:

- reliability plot (predicted confidence vs empirical accuracy)

## 4) Protocol details per system

### 4.1 ShazamKit

- Build a small harness app (iOS) or macOS tool to submit audio buffers.
- Ensure consistent audio format and avoid preprocessing that would advantage/disadvantage.

Log:

- query params (window length, condition)
- recognized track ID (if any)
- timestamp offset (if available)
- runtime

### 4.2 ACRCloud

- Use their SDK/API with an explicit query length.
- Rate limits + network latency should be recorded separately from pure recognition.

### 4.3 Chromaprint

- Use `fpcalc` to compute fingerprints and match against a local index (or AcoustID if allowed).
- Prefer local matching for reproducibility.

## 5) Analysis plan

- Plot accuracy vs SNR for each method.
- Plot accuracy vs RT60.
- Report interaction: low SNR + high reverb.
- Identify failure clusters:
  - songs repeatedly confused with each other
  - genres most affected

## 6) Expected outcome

Hypothesis: ShazamKit and ACRCloud will degrade sharply at -5 dB and high RT60; Chromaprint will be weak on short windows and under distortion.

This baseline becomes the acceptance target for Phase 2–3 embedding models.
