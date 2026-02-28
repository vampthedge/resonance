# Experiment plan: simulate concert audio from clean recordings

Goal: build a reproducible pipeline that transforms clean studio tracks into **concert-like recordings**.

## 1) Degradation components

### 1.1 Impulse response convolution (venue reverb)

- Convolve clean audio with real or simulated RIRs.
- Control parameter: RT60 (reverb time).

Sources:

- Public RIR datasets (to curate)
- Synthetic RIRs with `pyroomacoustics` (shoebox + absorption)

### 1.2 Crowd noise mixing

- Use real crowd recordings (FreeSound).
- Mix at target SNR values.

Notes:

- Use diverse noise types: applause, chanting, cheering, whistles.
- Ensure non-stationary behavior is represented.

### 1.3 PA / mic coloration

- Random EQ curves to simulate:
  - bass boost
  - “smile” EQ
  - harsh mid boost
  - device mic roll-off

### 1.4 Dynamic compression and limiting

- Apply compressor with randomized parameters.
- Apply limiter to mimic live capture chains.

### 1.5 Mic saturation / clipping

- Soft saturation: `tanh(gain * x)`
- Hard clipping: `clip(x, -t, t)`

## 2) Proposed Python implementation outline

### 2.1 Dependencies

- `librosa` or `torchaudio` for loading/resampling
- `soundfile` for writing
- `numpy` for mixing
- `pyroomacoustics` for RIR synthesis

### 2.2 Pseudocode

```python
import numpy as np
import soundfile as sf
import librosa


def rms(x):
    return np.sqrt(np.mean(x**2) + 1e-12)


def mix_at_snr(signal, noise, snr_db):
    # scale noise so that 20log10(rms(signal)/rms(noise_scaled)) = snr_db
    s = rms(signal)
    n = rms(noise)
    noise_scaled = noise * (s / (n + 1e-12)) * 10 ** (-snr_db / 20)
    return signal + noise_scaled


def soft_clip(x, drive=2.0):
    return np.tanh(drive * x)


def hard_clip(x, thresh=0.8):
    return np.clip(x, -thresh, thresh)


def apply_rir(x, rir):
    y = np.convolve(x, rir, mode="full")
    return y[: len(x)]


# pipeline
x, sr = librosa.load(path, sr=16000, mono=True)
# 1) reverb
x = apply_rir(x, rir)
# 2) crowd noise
noise, _ = librosa.load(noise_path, sr=sr, mono=True)
noise = noise[: len(x)]
x = mix_at_snr(x, noise, snr_db=0)
# 3) compression (placeholder; implement with a DSP lib)
# 4) saturation/clipping
x = soft_clip(x, drive=3.0)
# write
sf.write(out_path, x, sr)
```

## 3) Dataset outputs

For each clean track segment, produce:

- `clean.wav`
- `degraded_{snr}_{rt60}_{clip}.wav`

Also produce a manifest (JSON/CSV):

- track_id
- segment_start
- snr_db
- rt60
- eq_params
- clip_params
- noise_source_id

## 4) Validation checks

- Confirm resulting SNR matches target (within tolerance).
- Confirm no major clipping unless intended.
- Visualize spectrograms to sanity-check reverb/noise effects.

## 5) Why simulation matters

- Provides perfect labels.
- Allows controlled sweeps (SNR, RT60).
- Enables fair baseline comparisons and model evaluation.

Later, augment with real concert snippets to reduce simulation-to-reality gap.
