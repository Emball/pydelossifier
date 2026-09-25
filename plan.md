# Implementation Plan

## Phase 1 — Foundation

### 1.1 Audio I/O + CLI skeleton (`cli.py`)
- argparse: `input`, `output`, optional `--no-dr`, `--no-fh`, `--verbose`
- soundfile load/save
- normalize to float32
- wire up progress display

### 1.2 Progress display (`progress.py`)
- Mimics Stereo Tool's popup progress bar, but CLI
- Stages: Analyzing / MPEG detected / Pre-ringing found / Processing / Done
- tqdm-backed with stage labels and elapsed time
- Detection confidence printed inline

### 1.3 FFT layer (`pipeline.py`)
- `numpy.fft.rfft` with frame size 52224
- Hann window (assumed — confirm against Ghidra constants later)
- Overlap-add framing (50% overlap assumed)
- Circular buffer class: 10-frame history, 13056 bins, ring indexing

---

## Phase 2 — Detection (`detect.py`)

Goal: reproduce the MPEG detection gate from `param_1 + 0x750`.

### Heuristics to implement
1. **Spectral flatness** — encoded audio has characteristically flat high-frequency rolloff
2. **Comb pattern detection** — MP3 quantization leaves comb-shaped artifacts at predictable bin intervals
3. **High-frequency energy ratio** — ratio of energy above ~15kHz to total energy, normalized by bitrate estimate
4. **Frame-to-frame coherence** — lossy codecs produce characteristic inter-frame correlation patterns

### Output
- `confidence: float` (0.0–1.0)
- `codec_hint: str` ("mp3" | "aac" | "unknown")
- `pre_ringing_frames: float` (fraction of frames with detected pre-ringing)

---

## Phase 3 — De-ringing (`dr.py`)

From `FUN_7ffacca116e0`:

Core operation per bin:
```python
output[i] = (input[i] - hiyf[i]) + prev[i]
```

Where `hiyf` is a high-frequency copy buffer used to isolate ringing artifacts.

### Steps
1. Compute `hiyf` buffer: high-pass filtered copy of input spectrum
2. Subtract ringing component: `input - hiyf`
3. Add back previous clean frame: `+ prev`
4. Update `prev` buffer
5. Gate on pre-ringing detection flag from Phase 2

### Still to resolve
- Need to read `FUN_7ffacc0ff3f0` fully — it's the inner DR function called from `FUN_7ffacca116e0`. Go to `7ffacc0ff3f0` in Ghidra next session.
- Stereo handling: original swaps L/R channels internally in some cases (the `param_3 + 0x4200` swap block)

---

## Phase 4 — Delossifier (`delossifier.py`)

From `FUN_7ffacc0f7ca0` (AVX2 path).

### 4.1 Per-bin magnitude computation
```python
# Fast inverse sqrt (Newton-Raphson, matching original's vrsqrtps + refinement)
def fast_invsqrt(x):
    r = 1.0 / np.sqrt(x + 1e-30)   # scalar equivalent of vrsqrtps
    r = -(r*r*x) + r + r             # Newton-Raphson refinement
    return r

magnitude = x * fast_invsqrt(x)     # = sqrt(x)
```

### 4.2 Circular history buffer
```python
# 10 frames × 13056 bins
history = np.zeros((10, N_BINS), dtype=np.float32)
write_ptr = 0  # ring index, matches 0x665c0 offset logic
```

### 4.3 Harmonic prediction
For each bin `k`, the expected energy is estimated from 10 surrounding harmonic partials:
```python
harmonics = [k * n for n in range(1, 11) if k*n < N_BINS]
predicted = weighted_sum(history[recent_frames, harmonics])
```
Weights come from the `fRam00007ffacdf02d*` table constants in Ghidra — these need extraction.

### 4.4 Hole detection + fill
```python
if magnitude[k] < threshold * predicted[k]:
    # bin is a hole — fill from harmonic prediction
    spectrum[k] = predicted[k] * phase_from_neighbors(k)
```

### 4.5 Two fill modes (from `iVar62` flag in Ghidra)
- **Mode 0** (codec-aware): uses codec-specific comb pattern to target fill precisely
- **Mode 1** (generic): fills all bins below prediction threshold

### Constants to extract from Ghidra
The `fRam00007ffacdf02d*` / `fRam00007ffacdf02e*` constants are harmonic weights and thresholds. Need to dump them from the memory image. Next Ghidra session: open the dump, navigate to `0x7ffacdf02d00`, export 0x400 bytes as float32 array.

---

## Phase 5 — Spectral Hole Filler (`fh.py`)

From `FUN_7ffaccf26c40`.

Lighter second pass — operates on already-DL-processed spectrum.

### Steps
1. Interleave real/imag into complex pairs (the `vunpcklps`/`vperm2f128` pattern)
2. Second FFT pass
3. Same hole-fill logic as DL but with looser thresholds
4. Inverse interleave back to separate real/imag

---

## Phase 6 — Pipeline + Integration (`pipeline.py`)

Wire everything together:
```python
for frame in frames:
    spec = rfft(frame * window)
    spec = dr(spec)
    spec = delossifier(spec)
    spec = fh(spec)
    out = irfft(spec)
    overlap_add(out)
```

---

## Phase 7 — Tests

- `test_detect.py`: feed known MP3 and WAV frames, assert confidence thresholds
- `test_delossifier.py`: synthesize a tone, zero out bins, assert fill restores energy
- `test_pipeline.py`: end-to-end input→output smoke test, assert no clipping, assert SNR improvement

---

## Open Questions (block implementation until resolved)

| Question | Where to look |
|---|---|
| Exact windowing function | Ghidra: function before first FFT call |
| Harmonic weight constants | Ghidra: dump `0x7ffacdf02d00` → `0x7ffacdf03300` as float32 |
| Overlap-add percentage | Ghidra: frame advance stride vs frame size |
| DR inner function (`0xff3f0`) | Ghidra: navigate there, read decompiler |
| Pre-ringing threshold exact value | Ghidra: `DAT_7ffaceaabdb8` context |

---

## Order of Implementation

1. cli.py + progress.py (skeleton, no DSP)
2. pipeline.py FFT layer only (passthrough)
3. detect.py
4. delossifier.py (core, no DR/FH yet)
5. dr.py (after reading `0xff3f0`)
6. fh.py
7. Wire pipeline + tests
8. Extract Ghidra constants, tune thresholds
