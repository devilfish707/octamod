# IronOxide5 testing

## Commands and exact revision

All results below are for this draft's files on top of Octamod `main` at
`433fa32c0f5b381cf5a71930dc45e121609b7f4d` (2 Oct 2026), with the vendored
DSP56300 tree built from `sdk/octabam/scripts/vendor.sh dsp56300` (pin
`8ccdd843`, `tools/patches/dsp56300.patch` applied) on Linux x86-64.

| file | SHA-256 |
|---|---|
| `ironoxide5.asm` | `9968f2f90566dcf337cba7944f05b652591106480f7d2e7ff958c630da269d1a` |

### The render gate: `verify.py`

```sh
cd sdk/octabam
bash scripts/vendor.sh dsp56300
cmake -S vendor/dsp56300 -B vendor/dsp56300/build -DCMAKE_BUILD_TYPE=Release
cmake --build vendor/dsp56300/build --target dsp56kDisassemble dsp_asm dsp_host -j8
python3 ../drafts/ironoxide5/verify.py
```

No firmware is read. `verify.py` assembles `ironoxide5.asm` at P:0x2000 and
runs it in `dsp_host` as one instance from a synthetic memory image: a no-op
frame-context routine (`-ctx 40,41,42`), 16-sample blocks, r7 = 0x6200,
r6 = 0x506, page-1 knobs as value << 16. The reference is `reference.py`,
a line-for-line port of `IronOxide5Proc.cpp` at Flutter = Noise = 0,
limited to ±1 as the DSP's output store is.

Result on the revision above:

```
assembled 654 words; init P:2000 proc P:200b
[PASS] channel_l is straight-line, one rts
[PASS] channel_r is straight-line, one rts
[PASS] no mpysu anywhere []
[PASS] dearest settings: zero in, zero out
[PASS] peak error vs the plugin <= 0.001 worst 7.65e-04 at INPUT/HIGH/LOW/OUTPUT/MIX, signal ((64, 72, 72, 127, 127), 4000)
       meter: 184.7 instructions/sample (one instance, dsp_host)
[PASS] MIX 0 is dry at HIGH 72, LOW 72, INPUT/OUT 127 max deviation 1 LSB
[PASS] MIX 0 is dry at HIGH 0, LOW 127, INPUT/OUT 127 max deviation 1 LSB
[PASS] MIX 0 is dry at HIGH 127, LOW 0, INPUT/OUT 127 max deviation 1 LSB
[PASS] MIX 64 = 63/127 dry + 64/127 wet max deviation 3.19e-05
[PASS] stereo render == two mono renders, bit for bit
[PASS] split 7/9 blocks == unsplit render, bit for bit
all IRONOXIDE5 gates passed
```

The nine settings are the defaults, OUT at 127, everything at 127,
everything at 0, three mixed settings, MIX 64 and MIX 0; the signals are a
0.9 impulse, a 0.5 step and 50, 220, 1,000, 4,000 and 12,000 Hz at −2 dBFS.
The 7.7e-4 worst case is OUT at +18 dB, which multiplies the sine fit's
6.8e-5 error by about 8. With the degree-7 sine the whole grid was within
1.5e-4.

### The octabam source, for the record

The same renders against octabam's `ironoxide5.asm` (SHA-256
`ae5a5507…6e35c6`, with its own knob layout: OUT on slot 5, MIX on page 2)
measured a peak
error of 2.0 (OUT 127, 4 kHz) and 0.64 at the defaults. Its per-block
coefficients were read back with `dsp_host -peekx` and all matched the
plugin's formulas, so the per-block fits are kept unchanged. The
per-sample path had three defects:

1. **The saturators' square was mpysu.** `mpy y0,y1` is not a pair dsp_asm
   knows; it encodes as `mpysu`, which reads the second operand as unsigned.
   For a negative argument t it computed t·(t + 2) instead of t², so the
   negative half of both saturators was wrong. The disassembly of the
   octabam build had 8 `mpysu y0,y1`, 12 `mpysu x1,y1` and 1
   `mpysu x0,x1`.
2. **Every branchless min stage returned 2 × min** (no final `/2`), so both
   saturators ran at twice their input.
3. **The A/B selection used n7**, the frame count the dispatcher passes in,
   as an index register inside the loop.

The rewrite uses only signed pairs (`y1,y0`, `y0,y0`, `y1,x1`, `y0,x0`,
`x0,x0`, `mac y1,y0`), clamps with `cmp` + `tgt`/`tlt`, and points r4 at the
current copy and r5 at the other, swapped every sample (m4/m5 set linear,
as Character does). The flip is stored back as r4 − r7 at the end of each
call.

### Optimisation

| | instructions per 16-sample block, one instance |
|---|---:|
| octabam build | 4,455 |
| rewrite, degree-7 sine | 3,432 |
| this version | 2,954 |

The steps after the rewrite: a degree-5 minimax sine (`gen_constants.py`
prints it; error 6.8e-5 on [0, π/2]) saves one multiply-accumulate per
saturator; the highpass computes x − iir directly as (x − iir)·(1 − amount)
with 1 − amount precomputed per block; and two shift pairs fold away.
The grid's worst error went from 1.5e-4 to 7.7e-4, all of it from the
shorter sine at high OUT; at the defaults the composed image is within
1.8e-4.

### Static checks with the test remix

With the draft copied to `sdk/octabam/modules/ironoxide5/` and
`hardware-test-remix.py` as `sdk/octabam/remixes/ironoxide5-spring/remix.py`:

- `cycle_count.py`: **194 cycles/sample** (`bsr channel_l 85w/call, bsr
  channel_r 85w/call`); worst core 776 with four FX2 instances.
- `verify_menu.py`: all checks pass (chooser row 13, id 0x1e, knobs 0–4
  enabled and named).
- `verify_initregs.py`: init preserves r1.
- `verify_replaces.py --image`: every stock id is stock's.
- `label_fmt.py`: OK.
- `tools/remix/ledger.check` over the eleven modules, the TapeHead draft
  and IronOxide5: no conflicts.

## The hardware test image

```sh
cd sdk/octabam
make image REMIX=ironoxide5-spring BUILD=4
```

Built 2 Oct 2026 from the author's own OS 1.40C (MAIN OS section SHA-256
`164f3122…0a84e`, the fingerprint `stock_guard.py` expects):

```
IRONOXIDE5    P:0x01252..0x014e0 ( 654 words)  id 0x1e      (payload A)
IRONOXIDE5    P:0x01012..0x012a0 ( 654 words)  id 0x1e      (payload B)
region P:...  (1063 words)  used 654  FREE 409
out/mainos_bus.bin: 1,112,560 bytes, 3831 changed
```

| artifact (local only, never committed) | SHA-256 |
|---|---|
| `out/mainos_bus.bin` | `76d6290a098aa083787e07da77d1718f9b1a88a3e28e3b0df7d4141f74d20d19` |
| `out/OCTATRACK_OCTABAM4.bin` | `47c30f156bc40e991140961ef33a3365afad8b27ff3e18c1296b730ec9d25834` |
| `out/OCTATRACK_OS1.40C_OCTABAM4.syx` | `dc00a2541536ba2229c20adb66b945aa19285a04e736a96dc0d1da06e4433ca8` |

`make_bin.py` round-trips the card image (payload and checksum ok).
Composed-image render: `benchmark.py`'s IRONOXIDE5 dump, one instance at
the defaults, is within 1.8e-4 of `reference.py`, L = R for mono input.

## IronOxide5 against SPRING REV: `benchmark.py`

```sh
cd sdk/octabam
REMIX=ironoxide5-spring python3 ../drafts/ironoxide5/benchmark.py
```

It reuses `tools/harness/benchmark_reverbs.py`'s runner: both cores from
their real payloads (stock SPRING REV from the untouched 1.40C, IRONOXIDE5
from the image above), four FX2 slots per core at the real r7 stride,
16-sample blocks, 2,048 blocks per case, every active control moving every
block in the modulated cases, all 16 trigger-split positions. The unit is
**executed DSP instructions**, not hardware cycles.

| case | IronOxide5 | SPRING REV |
|---|---:|---:|
| one instance, fixed controls, per 16-sample block | 2,954 | 4,186 |
| one instance, per sample | 184.6 | 261.6 |
| four per core, fixed controls, peak per block | 11,816 | 16,744 |
| four per core, modulated, peak per block | 11,820 | 16,748 |
| four per core, worst of all splits, peak per block | 13,144 | 20,376 |
| per core, per sample, worst | 821.5 | 1,273.5 |
| init, per core | 44 | 380 |

SPRING REV's figures are identical to the TapeHead run and to the Mini Verb
benchmark's record for pristine 1.40C.

Memory:

| | IronOxide5 | SPRING REV |
|---|---:|---:|
| DSP program, per payload | 654 words | 1,063 words |
| FX2 instance buffer (allocator) | none | 16,384 words per instance slot |
| per-instance state | 28 words of its r7 block | not measured |
| ColdFire | cloned descriptor | stock descriptor |

## OT UI capture evidence

`scripts/capture-module-ui.py` on `ot_emu` (SHA-256 `2360ffb2…5115`), MKII
panel, empty scratch card, transport stopped, the image above. Plan and
hashes: `media/capture.json`. Reviewed:

- `media/ot-location.png`: FX2 SETUP after YES, IRONOXIDE5 highlighted in
  the row after PLATE REV, no SETUP controls.
- `media/ot-controls.png`: the FX2 main page, INPUT, HIGH, LOW on top, OUT
  and MIX below, footer `FX2▸IRONOXIDE5`.
- `media/ot-out.png`: OUT turned with encoder D, printing 74.

UI evidence only: not a hardware or audio test.

## Stress and audio quality

Not run. Required before publication: at least 60 minutes on real MKI or
MKII hardware with all eight audio tracks active, IronOxide5 on FX1 and FX2,
every knob swept by LFOs and p-locks, Part changes and FX reselection, with
INPUT and OUT at 127. Record tester, date, project fingerprint, local image
SHA-256, source SHA-256 and results.

## Resources

- DSP cycles: 194 cycles/sample per instance, static, the same at every
  setting. dsp_host meter: 184.7 instructions/sample.
- DSP program: 654 words (1,962 bytes at 24 bits) in each payload's donor
  region.
- DSP X: 28 words per instance (r7 + $00–$1b) inside the dispatcher's own
  256-word r7 block. No allocator buffer, no Y memory.
- ColdFire: none beyond the cloned menu descriptor.

## Hardware

2 Oct 2026, OCTABAM3 on the author's unit: with MIX at 0, HIGH and LOW still
changed the sound. Cause: OCTABAM3 kept the plugin's inv/dry/wet knob, where
0 is dry plus the wet signal inverted (dry was at 64). That matched the
plugin and `verify.py` then (its gate checked "MIX 0 = dry - wet"), so no
render could flag it; it is a control-design problem, not an arithmetic one.

Fixed in OCTABAM4: MIX is now the plugin's upper half only (G = 0.5 +
MIX/254), so 0 is dry and 127 fully wet. `verify.py` now checks MIX 0 is dry
within 1 LSB with INPUT and OUT at 127 and HIGH/LOW at both extremes. Only
the MIX setup changed: 3 words less, 3 instructions per block less, the
LCD captures pixel-identical. OCTABAM4 has not been flashed.

The octabam build was never heard on a unit.
