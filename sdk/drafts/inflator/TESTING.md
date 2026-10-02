# Inflator testing

## Commands and exact revision

All results below are for this draft's files on top of Octamod `main` at
`433fa32c0f5b381cf5a71930dc45e121609b7f4d` (2 Oct 2026), with the vendored
DSP56300 tree built from `sdk/octabam/scripts/vendor.sh dsp56300` (pin
`8ccdd843`, `tools/patches/dsp56300.patch` applied) on Linux x86-64.

| file | SHA-256 |
|---|---|
| `inflator.asm` | `fcb1db5edf5f93f3b0f952098cc9963f08fa932258336f0975f7bfdad2acd8ba` |

The source is generated: `python3 gen_asm.py` rewrites `inflator.asm`
byte for byte from `gen_constants.py`.

### Source of the algorithm

`reference.py` is `JClones_OInflator.jsfx` from
https://github.com/JClones/JSFXClones at
`88a1503d668c378ced4c166e772378272f3b72ea` (MIT), line for line at
44.1 kHz. RCInflator 2 (Oxford Edition) by lewloiwc, which octabam's own
inflator ported, was read at ReaTeam/JSFX
`9a9b6449ffd4cd8cbaeb98b73125b54afe2ddee7`: neither the file nor the
repository carries a licence, so it is not used.

### The render gate: `verify.py`

```sh
cd sdk/octabam
bash scripts/vendor.sh dsp56300
cmake -S vendor/dsp56300 -B vendor/dsp56300/build -DCMAKE_BUILD_TYPE=Release
cmake --build vendor/dsp56300/build --target dsp56kDisassemble dsp_asm dsp_host -j8
python3 ../drafts/inflator/verify.py
```

No firmware is read. `verify.py` assembles `inflator.asm` at P:0x2000 and
runs it in `dsp_host` as one instance from a synthetic memory image: a no-op
frame-context routine (`-ctx 40,41,42`), 16-sample blocks, r7 = 0x6200,
r6 = 0x506, page-1 knobs as value << 16.

Result on the revision above:

```
assembled 576 words; init P:2000 proc P:2031
[PASS] ch_one is straight-line, one rts
[PASS] ch_split is straight-line, one rts
[PASS] no mpysu anywhere []
[PASS] SPLIT 0, dearest settings: zero in, zero out
[PASS] SPLIT 1, dearest settings: zero in, zero out
[PASS] peak error vs the JSFX <= 0.001 worst 2.09e-04 at INPUT/EFFCT/CURVE/CLIP/SPLIT/OUT, signal ((42, 127, 64, 1, 1, 127), 'noise')
       meter: 89.9 (single band) / 258.9 (band split) instructions/sample
[PASS] EFFECT 0, 0 dB in and out: passthrough within 32 LSB (-108 dBFS) max deviation 15 LSB (-115 dBFS)
[PASS] CLIP ON, +12 dB in: output within full scale peak 1.0000
[PASS] band split: stereo render == two mono renders, bit for bit
[PASS] band split: split 7/9 blocks == unsplit render, bit for bit
all INFLATOR gates passed
```

The eleven settings are the defaults, EFFECT 127, every extreme with clip
on and off, and mixed settings, in both modes; the signals are a 0.9
impulse, a 0.5 step, 50, 240, 1,000, 2,400 and 8,000 Hz at −2 dBFS, and
white noise.

The 15-LSB passthrough residue is the headroom: every band is carried at
the JSFX's x / 2 (and the JSFX itself works at half scale), which costs two
bits of the 24-bit input at 0 dB.

### Design and scaling

The JSFX runs internally at half scale (input × 0.5, clip at 0.5, output
× 2). With CLIP off and +12 dB in, its internal signal reaches 1, a band
1.7 and the three-band sum 3.2 (measured in `reference.py` with square
waves, noise and tones), so here every band value is carried as x / 2, each
shaped band as y / 4, and the output is (y / 4) × OUTPUT × 8. The filter
states are kept in the x / 2 scale and the JSFX's fixed ×2 / ×4 gains are
folded into the coefficients (`gen_constants.py`). The knob curves are
four-segment degree-5 fits (error under 1.3e-8).

Every multiply uses a pair dsp_asm encodes as signed (`y1,y0`, `y0,x0`,
`x0,y1`, `y1,x1`, `x0,x0`); clamps are `cmp` + `tgt`/`tlt`; the sample
routines never branch. The first assembly hit the label-prefix trap
(`fit_in` is a prefix of `fit_in_s0`, and the assembler resolved it as
`$..._s0`); the labels were renamed so none prefixes another.

### Optimisation

| version | single band, per block | band split, per block |
|---|---:|---:|
| first working version | 1,650 | 5,141 |
| tables + parallel moves | 1,462 | 4,166 |
| band-split constants in init | 1,436 | 4,140 |

The sample loop walks the shaper's constants (r5 → r1) and the band-split
coefficients (r3 → r2) with post-increment parallel moves and the filter
states with r4, so nearly every instruction is one word. Accuracy was
unchanged (2.09e-4).

### Static checks with the test remix

With the draft copied to `sdk/octabam/modules/inflator/` and
`hardware-test-remix.py` as `sdk/octabam/remixes/inflator-spring/remix.py`:

- `cycle_count.py`: **258 cycles/sample** (`worst of 2 mode loops
  (89/258)`); worst core 1,032 with four FX2 instances.
- `verify_menu.py`: all checks pass.
- `label_fmt.py`: CLIP and SPLIT print OFF / ON.
- `verify_initregs.py`: init preserves r1.
- `verify_replaces.py --image`: every stock id is stock's.
- `tools/remix/ledger.check` over the eleven modules and the three drafts:
  no conflicts.

## The hardware test image

```sh
cd sdk/octabam
make image REMIX=inflator-spring BUILD=5
```

Built 2 Oct 2026 from the author's own OS 1.40C (MAIN OS section SHA-256
`164f3122…0a84e`):

```
INFLATOR      P:0x01252..0x01492 ( 576 words)  id 0x1b      (payload A)
INFLATOR      P:0x01012..0x01252 ( 576 words)  id 0x1b      (payload B)
INFLATOR      slot 3  CLIP  prints OFF|ON
INFLATOR      slot 4  SPLIT prints OFF|ON
out/mainos_bus.bin: 1,112,560 bytes, 3477 changed
```

| artifact (local only, never committed) | SHA-256 |
|---|---|
| `out/mainos_bus.bin` | `4ec513e79009dca117ec054312a2265e7b76f3b689fdbb960920e4be91eb5cc6` |
| `out/OCTATRACK_OCTABAM5.bin` | `e7d24b09ee969d656bbd20dfd91d524dccfaf17c6a0d1eb3c3e3044e5eba5d6e` |
| `out/OCTATRACK_OS1.40C_OCTABAM5.syx` | `34bcef3a946bcc4bbc1acc1cb8a08ff1da507265a676fb3fbcdb4255f1ce4b57` |

`make_bin.py` round-trips the card image (payload and checksum ok).
Composed-image render, one instance: within 1.2e-6 of `reference.py` at the
defaults and 1.8e-4 at the dearest settings (band split, +12 dB, clip off).

## Inflator against SPRING REV: `benchmark.py`

```sh
cd sdk/octabam
REMIX=inflator-spring python3 ../drafts/inflator/benchmark.py
```

It reuses `tools/harness/benchmark_reverbs.py`'s runner: both cores from
their real payloads, four FX2 slots per core at the real r7 stride,
16-sample blocks, 2,048 blocks per case, every active control (SPLIT
included) moving every block in the modulated cases, all 16 trigger-split
positions. The default knobs leave SPLIT off, so two extra cases run at the
manifest's dearest settings (band split on). The unit is **executed DSP
instructions**, not hardware cycles.

| case | Inflator | SPRING REV |
|---|---:|---:|
| one instance, defaults (single band), per block | 1,436 | 4,186 |
| one instance, band split, per block | 4,143 | — |
| four per core, defaults, peak per block | 5,744 | 16,744 |
| four per core, band split, peak per block | 16,572 | — |
| four per core, modulated, peak per block | 16,572 | 16,748 |
| four per core, worst of all splits, peak per block | 17,272 | 20,376 |
| init, per core | 144 | 380 |

Memory:

| | Inflator | SPRING REV |
|---|---:|---:|
| DSP program, per payload | 576 words | 1,063 words |
| FX2 instance buffer (allocator) | none | 16,384 words per instance slot |
| per-instance state | 43 words of its r7 block | not measured |
| ColdFire | cloned descriptor, two 52 B label caves | stock descriptor |

## OT UI capture evidence

`scripts/capture-module-ui.py` on `ot_emu` (SHA-256 `2360ffb2…5115`), MKII
panel, empty scratch card, transport stopped, the image above. Plan and
hashes: `media/capture.json`. Reviewed:

- `media/ot-location.png`: FX2 SETUP after YES, INFLATOR highlighted in the
  row after PLATE REV, no SETUP controls.
- `media/ot-controls.png`: the FX2 main page at the defaults, CURVE drawn as
  a centred bipolar dial, CLIP and SPLIT as selects.
- `media/ot-split.png`: EFFCT at 90 and SPLIT printing ON.

UI evidence only: not a hardware or audio test.

## Stress and audio quality

Not run. Required before publication: at least 60 minutes on real MKI or
MKII hardware with all eight audio tracks active, Inflator on FX1 and FX2
with SPLIT on, every knob swept by LFOs and p-locks, SPLIT and CLIP toggled
by scenes, Part changes and FX reselection, INPUT at 127 and CLIP off.
Record tester, date, project fingerprint, local image SHA-256, source
SHA-256 and results.

## Resources

- DSP cycles: 89 (single band) / 258 (band split) cycles/sample per
  instance, static. dsp_host meter: 89.9 / 258.9 instructions/sample.
- DSP program: 576 words (1,728 bytes at 24 bits) in each payload's donor
  region.
- DSP X: 43 words per instance inside the dispatcher's own 256-word r7
  block. No allocator buffer, no Y memory.
- ColdFire: the cloned menu descriptor and two label caves.

## Hardware

Untested.
