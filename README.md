# pydelossifier

Open-source Python reimplementation of the **Delossifier** and **Spectral Hole Filler** from [Stereo Tool](https://www.thimeo.com/stereo-tool/) by Jan Plugins.

Repairs spectral damage from lossy audio codecs (MP3, AAC, etc.) **without machine learning** — pure DSP, runs standalone from the command line, no DAW required.

## What it does

Lossy codecs discard or heavily quantize frequency content to save space. The result is spectral "holes" — bins with missing or heavily attenuated energy. Pydelossifier:

1. **Detects** MPEG compression artifacts using spectral flatness and comb pattern analysis
2. **De-rings** pre-ringing transients introduced by the codec
3. **Fills spectral holes** by inferring missing energy from the harmonic series across a multi-frame history buffer
4. **Runs a second-pass** spectral repair for any remaining gaps

## How it was built

The algorithm was reverse-engineered from the Stereo Tool VST DLL:

- Unpacked Enigma Protector via x64dbg + Scylla memory dump
- Decompiled with Ghidra (NSA's open-source RE tool)
- Identified the DSP pipeline via internal string labels (`delossifier_input`, `Chain_HQ_PC_DL`, etc.)
- Reconstructed the algorithm from AVX2 SIMD decompiler output

The original uses Intel IPP for FFT and hand-vectorized AVX2/SSE4 for the DSP inner loops. This implementation uses NumPy equivalents.

## Usage

```bash
pip install pydelossifier
pydelossifier input.mp3 output.wav
```

Progress output:
```
Analyzing...       [████████░░]
MPEG detected      (confidence: 87%)
Pre-ringing found  (43% of frames)
Processing...      [████████████] 100%  2m14s
```

## Status

🚧 **In active development** — algorithm planning and architecture complete, implementation in progress.

## Algorithm overview

```
Input → FFT → DR (de-ringing) → DL (Delossifier) → FH (Spectral Hole Filler) → IFFT → Output
```

- **Frame size:** 52,224 samples
- **History buffer:** 10 frames × 13,056 bins
- **Harmonic window:** 10 surrounding partials per bin
- **Sqrt method:** Newton-Raphson fast inverse sqrt (1 iteration, matching original)

## License

MIT
