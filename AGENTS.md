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

### Key Constants
- Frame size: 52224 samples
- Circular buffer length: 13056 bins
- History depth: 10 frames
- Harmonic window: 10 surrounding partials per bin
- Newton-Raphson for fast sqrt: `r = -(r*r*x) + r + r`

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

- Exact windowing function used before FFT (Hann assumed, not confirmed).
- Exact threshold constants for hole detection (the `fRam00007ffacdf02d*` values from Ghidra — need extraction).
- Whether DL operates on overlapping frames (overlap-add assumed).
- Exact pre-ringing detection threshold for DR gating.
- `FUN_7ffacc102c30` (DR innermost processing call) not yet read — low priority since DR is secondary to DL/FH.

## Version

0.0.0.2
