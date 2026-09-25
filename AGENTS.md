# pydelossifier — AGENTS.md

## What This Is

Open-source Python reimplementation of the Delossifier and Spectral Hole Filler modules from Stereo Tool (Jan Plugins). Reverse-engineered from the packed DLL via Enigma Protector unpacking → Scylla memory dump → Ghidra decompilation. No ML. Pure DSP.

Processes audio files from the CLI. Detects lossy compression artifacts and fills spectral holes using harmonic inference across a multi-frame circular buffer.

## Algorithm — What We Know From Ghidra

### Pipeline Order (mirrors Stereo Tool internal chain)
```
Input → FFT → DR (de-ringing) → DL (Delossifier) → FH (Spectral Hole Filler) → IFFT → Output
```

### FFT
- Intel IPP `ippsDFTInitAlloc_R_32f` / `ippsFFTInitAlloc_R_32f` in the original.
- We use `numpy.fft.rfft` / `numpy.fft.irfft` as equivalent.
- Frame size: 0xcc00 = 52224 samples (from `lVar71 = lVar96 * 0xcc00`).
- Circular buffer: 0x3300 = 13056 bins, ~10 frames of history.

### DR — De-ringing / BIYF (`FUN_7ffacca116e0`, inner: `FUN_7ffacc0ff430`)
- Internally called BIYF ("Bandlimited Inverse YF"). Strings: `s_BIYF_YES`, `s_BIYF_NO`, `s_BIYF_YES_3RD`.
- Three-band spectral shaping system, not simple subtraction. `bVar55 < 3` outer loop = 3 bands.
- Core per-band op (triple product with window coefficients): `output[i] = spectrum[i] * reference[i] * window[i]`
- Four overlapping window arrays at `param_1 + 0x140`, `0x160`, `0x180`, `0x1a0`.
- Energy accumulator = MPEG detector: `energy += (sample[i] * ref[i])²`, fast invsqrt → inverse RMS table at `param_1 + 0x12a1c0`.
- Pre-ringing frame count `iVar53 = (int)(frame_count * sample_rate * DAT_7ffaceaac968 + 0.5)`.
- Gated by MPEG detection flag at `param_1 + 0x750` / `0x751`.
- Handles stereo channel swap internally.
- `FUN_7ffacc102c30` is the core inner processing call (still unread — low priority, DR is secondary to DL).

### DL — Delossifier (`FUN_7ffacc0f7ca0`, AVX2 path)
- Per-bin spectral magnitude via fast inverse sqrt (Newton-Raphson, 1 iteration):
  `r = vrsqrt(x); r = -(r*r*x) + r + r; mag = x * r`
- Maintains 10-frame circular history buffer at offset `0x665c0`, size `0x3300`.
- Harmonic prediction: each bin's expected energy estimated from weighted sum of 10 surrounding harmonic partials across multiple prior frames.
- Hole detection: `vmaxps`/`vminps` chains compare current magnitude against prediction envelope.
- Reconstruction: missing energy filled from harmonic series interpolation.
- Two code paths internally: `iVar62` flag selects between codec-aware fill and generic fill.
- Three SIMD variants (AVX2 / SSE4 / scalar) selected at runtime; we implement scalar equivalents.

### FH — Spectral Hole Filler (`FUN_7ffaccf26c40` + `FUN_7ffaccf26de0`)
- `FUN_7ffaccf26c40`: interleaver only — merges separate real/imag arrays into complex pairs before second FFT pass.
- Second-pass spectral repair, lighter than DL; operates on already-filled spectrum.

### MPEG Detection
- Flag at `param_1 + 0x750` / `param_1 + 0x751` gates the Delossifier.
- Confidence score stored at `param_1 + 0x7a4`.
- Detection is pre-pass; we implement heuristic detection (spectral flatness + comb pattern analysis) before the DSP pipeline runs.

### Key Constants (verified from Ghidra float dump, `0x7ffacdf02d00`–`0x7ffacdf03400`)

**Newton-Raphson fast sqrt coefficients:**
- `0x2d00`: `1/9 ≈ 0.1111` (x8) — Newton-Raphson denominator correction
- `0x2d20`: `0.25` (x8) — harmonic weight decay factor
- `0x2d40`: `3.0` (x8) — the `3 - x*r*r` coefficient in `r * (3 - x*r*r)`
- `0x2d60`: `~1.4e-45` = `Float.MIN_VALUE` (x8) — denormal / epsilon guard
- `0x2d80`: `2.0` (x8) — harmonic energy scaling

**Per-harmonic triplet pattern (repeats x10 for 10 harmonics, starting at `0x2de0`):**
Each 0x60-byte block: `[3.0 x8], [-0.5 x8], [Float.MIN_VALUE x8], [NaN-mask x8]`
- `3.0` = Newton-Raphson coefficient (same formula, applied to harmonic magnitudes)
- `-0.5` = bias subtracted from reconstruction estimate
- `Float.MIN_VALUE` = comparison floor for hole detection threshold
- `NaN` = bitmask used in `vcmpps` to gate reconstruction (disables output when magnitude above threshold)

**BIYF / DR constants:**
- `0x32e0`: `π ≈ 3.1415927` (x8) — used in cosine window for BIYF band shaping
- `0x3300`: `0.5` (x8) — BIYF amplitude scale
- `0x3320`: `[1,2,3,4,5,6,7,8]` as floats — harmonic index series

**Comb pattern detection (for MPEG identification):**
- `0x3344`: `2.5625` — inter-bin ratio for MP3 quantization comb
- `0x338c`: `1.875` — second comb ratio (AAC / lower-quality pattern)

**Harmonic weights (extracted from `0x7ffacea99450`–`0x7ffacea994dc`):**
Weights decay non-linearly across 10 harmonics (harmonic 1 is fundamental, 2–10 are partials):
`[0.1, 0.9, 0.8, 0.7, 0.6, 0.4, 0.3, 0.2, 0.1]`
(Harmonic 1 = 0.1 is low because it's the direct-copy fundamental; harmonics 2–9 are the actual fill sources)

**Windowing (confirmed from `0x7ffacea984e0`–`0x7ffacea98560`):**
Window type: **sqrt-Hann** (sine window). Key values: `0.5` and `0.35355338` (= `1/sqrt(8)` = `sin(π/8)`).
This is `w(n) = sin(π·n/N)` — standard MPEG/overlap-add sine window.
Four overlapping window arrays at `param_1+0x140/0x160/0x180/0x1a0` correspond to the 4 SIMD lanes.

**Overlap-add stride (confirmed from buffer offsets):**
- Frame size: `0xcc00` = 52224 samples
- Hop size: `0x4200` = 16896 samples
- Overlap: ~67.6% (~2/3)
- This is standard 3:1 overlap-add (hop ≈ frame/3.09, near-integer approximation of frame/3)

**DR/BIYF angle constants (from `0x7ffacea98930`):**
- `π/32 = 0.09817477` — band boundary angle for BIYF cosine shaping
- `π/4 = 0.78539816` — second band boundary

**Practical summary for implementation:**
- Hole threshold: a bin is declared a hole when `current_mag < Float.MIN_VALUE` (effectively zero after codec quantization)
- Fill formula: `fill = harmonic_prediction * (3*r - 0.5)` where `r = vrsqrt(energy)`
- Reconstruction is gated by the NaN-mask: holes with `energy == 0` skip the fill
- Frame size: 52224 samples; circular buffer: 13056 bins; history: 10 frames; harmonics per bin: 10
- Windowing: sine window `w(n) = sin(π·n/N)` applied before FFT and after IFFT
- Hop: 16896 samples (~frame/3)

## Repo Structure (planned, not yet built)

```
pydelossifier/
  pydelossifier/
    __init__.py
    detect.py        # MPEG artifact detection + confidence scoring
    dr.py            # De-ringing module
    delossifier.py   # Core DL spectral reconstruction
    fh.py            # Spectral hole filler (second pass)
    pipeline.py      # Orchestrates the full chain
    progress.py      # CLI progress bar (mimics Stereo Tool UI)
  cli.py             # Entry point: argument parsing, file I/O
  tests/
    test_detect.py
    test_delossifier.py
    test_pipeline.py
  AGENTS.md
  README.md
  requirements.txt
  plan.md
```

## CLI Behaviour (planned)

```
pydelossifier input.mp3 output.wav
```

Progress display:
```
Analyzing...       [████████░░]
MPEG detected      (confidence: 87%)
Pre-ringing found  (43% of frames)
Processing...      [████████████] 100%  2m14s
```

## Dependencies (planned)

- numpy — FFT and array ops
- soundfile — audio I/O
- librosa — optional, resampling
- tqdm — progress bars
- scipy — signal processing utilities

## What Is Not Yet Known / Still To Investigate

- Exact pre-ringing detection threshold for DR gating (confirmed as runtime computed from sample rate; tunable parameter in implementation).
- `FUN_7ffacc102c30` (DR innermost processing call) not yet read — low priority since DR is secondary to DL/FH.

All other unknowns resolved via second Ghidra float dump (`0x7ffacea98000`–`0x7ffacea9a000`):
- Windowing: confirmed sine window (sqrt-Hann), `w(n) = sin(π·n/N)`.
- Overlap: confirmed 3:1 (~67.6%), hop = 16896 = frame/3.09.
- Harmonic weights: `[0.1, 0.9, 0.8, 0.7, 0.6, 0.4, 0.3, 0.2, 0.1]` for harmonics 1–9.

All critical threshold constants extracted via Ghidra Java script — see Key Constants above.

## Version

0.0.0.4
