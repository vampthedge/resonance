# ARCHITECTURE — Setlist-based, on-device concert song recognition (CoreML)

Resonance is a **setlist-aware**, **on-device** song recognition system designed for live concerts.

**Core mission: replace the Front of House (FOH) operator.**
Today, a human at FOH manually cues lyrics by watching the stage and pressing buttons. Resonance eliminates that workflow: it recognizes what song is playing **in real time** from a stage/FOH microphone and **automatically triggers lyric sync events** for AUGE’s Lyrical platform — with **no human intervention**.

The key product reframing is **closed-world recognition**: at a given concert, the candidate catalog is the setlist (typically **20–50 songs**), not “all music.” That enables a tiny offline micro-catalog and reliable temporal gating.

---

## Design principles

- **Micro-catalog retrieval beats global search**: With 20–50 candidates, exact cosine similarity search is cheap and stable, and we can rely on temporal consistency logic.
- **Offline-first (CoreML)**: At-concert recognition must work with no network and low latency.
- **System > model**: The defensibility is the end-to-end pipeline (setlist ingestion → studio embedding catalog → offline inference → automatic event triggering), not just a single embedding network.

---

## 1) Two pipelines: pre-concert build vs at-concert inference

### 1.1 Pre-concert pipeline (before the show)

**Goal:** build a tiny, show-specific **micro-catalog** from the setlist using **studio recordings** (clean, isolated), then package it for offline use on iOS.

1. **Input: setlist (20–50 songs)**
   - Song titles + artists (and optional alternate versions if known)

2. **Fetch studio recordings (clean, isolated)**
   - Apple Music / other clean sources, or user-provided files
   - Normalize sample rate + loudness

3. **Segment into overlapping windows**
   - e.g., **3.0s windows** with **50% overlap**

4. **Embed reference audio**
   - log-mel frontend → embedding model (CoreML-compatible) → L2-normalized vectors

5. **Store as an on-device micro-catalog package (<10MB)**
   - reference embeddings + metadata
   - CoreML model (or pinned version reference)
   - lightweight index/representation optimized for cosine similarity

**Output:** a downloadable/offline “concert pack” that the device can use fully offline during the show.

### 1.2 At-concert pipeline (during the show, fully offline + automated)

**Goal:** from a stage/FOH mic stream, identify the current song and **trigger lyric sync** automatically.

1. **Mic input (stage mic or near-FOH position)** → **noise suppression**
2. **Log-mel spectrogram** → **embedding model (CoreML)**
3. **Cosine similarity** against the micro-catalog
4. **Temporal consistency gate**: require **2 of the last 3** windows to agree
5. **Output:** matched song + confidence → **automatically triggers lyric sync event** in the Lyrical app
6. **NO human intervention required**

---

## 2) Validation with VEEP (test harness, not training data)

VEEP concert MP4s are used strictly for **validation/testing** under real concert conditions.

- Download concert MP4s from VEEP where the **setlist is known**
- Extract audio → run Resonance end-to-end → measure recognition accuracy under real crowd noise/reverb/distortion

Example extraction:

```bash
ffmpeg -i concert.mp4 -vn -ac 1 -ar 16000 -c:a pcm_s16le concert.wav
```

**Important:** VEEP is a **held-out benchmark** and reliability check. The model’s learning signal comes from **studio recordings** plus augmentation that simulates concert degradation.

---

## 3) ASCII architecture diagram

```
                 ┌──────────────────────────────────────────────────────┐
                 │                 PRE-CONCERT (build)                   │
                 └──────────────────────────────────────────────────────┘

   Setlist (20–50)     Fetch studio recordings        Segment (3s, overlap)
 (title+artist) ───▶ (clean, isolated sources) ───▶  sliding windows
         │                      │                           │
         v                      v                           v
  track metadata        normalized audio                log-mel windows
         │                      │                           │
         └──────────────────────┴───────────────┬───────────┘
                                                 v
                                        CoreML embedding model
                                                 │
                                                 v
                                  reference embeddings (L2-normalized)
                                                 │
                                                 v
                           micro-catalog "concert pack" (<10MB)
                           (embeddings + metadata + model version)


                 ┌──────────────────────────────────────────────────────┐
                 │              AT-CONCERT (offline inference)           │
                 └──────────────────────────────────────────────────────┘

   Mic near stage/FOH ─▶ noise suppression ─▶ log-mel (3s windows) ─▶ CoreML
                                                                     embed
                                                                       │
                                                                       v
                                                     cosine similarity vs
                                                     show micro-catalog
                                                                       │
                                                                       v
                                              temporal consistency (2/3)
                                                                       │
                                                                       v
                                      matched song + confidence + time
                                                     │
                                                     v
                                   auto lyric-sync trigger in Lyrical
```

---

## 4) What changes vs Shazam-style global fingerprinting

- We **do not** need internet-scale indexing, collision handling, or global uniqueness.
- The task becomes **robust retrieval within a tiny candidate set** under severe degradation.
- The system can afford strong temporal gating (2/3 windows) because latency requirements are measured in seconds, not milliseconds.

