# Benchmarks

How readers built from these specifications are compared for speed, the interface a reader
offers so that it can be measured, and the targets. This document is informative: it adds
no requirement to specifications 0 to 7, and a reader can conform without meeting any
target here.

## The wall

Benchmarks are run by the specification side, against compiled programs only. What crosses
the wall in either direction is limited to:

- **to the measuring side:** release binaries of the benchmark program described below, from
  the implementation's own releases. Its source is not read;
- **to the implementation side:** this document and the results recorded in it: machines,
  conditions, figures and targets. Never profiles, explanations of why another reader is
  faster, or suggestions about how to decode faster.

Comparison programs (other SACD readers) are run as compiled binaries only, and only on the
measuring side.

## The benchmark program

An implementation provides a command-line program for measurement. Its name is free; this
document calls it `bench`. It MUST support:

| Invocation | Behaviour |
|---|---|
| `bench --version` | prints the implementation name and version on one line and exits 0 |
| `bench decode IMAGE --area A --threads N --sink S` | decodes every track of area `A` of image `IMAGE` and exits 0, or exits non-zero with an error code on stderr (below) |

- `IMAGE` is a path to a disc image in any of the sector formats of specification 1.
- `A` is `2ch` or `mc` and selects an area by its **slot** (the 2-channel or the multichannel
  slot, specification 3, 3.1), not by its kind: a slot can yield an area of the other kind.
- `N` is the number of worker threads, 1 or more; `0` means one per logical CPU. With `N = 1`
  the program MUST do all decoding on a single thread.
- `S` is the sink:
  - `hash`: for every track, in track number order, print one line to stdout: the track
    number in decimal, a tab, then the SHA-256 of each channel's track audio, in channel
    order, separated by tabs. A track with no track audio (an outcome of specification 7,
    7.7.4: `track-empty`, `channel-count-unsupported` or `track-exceeds-capacity`) prints the
    track number, a tab and that outcome code instead; this is not an error and the exit
    status stays 0. The
    track audio and its hash are exactly as defined for the `sha256` field in
    [corpus/README.md](../corpus/README.md) (`(end_frame - first_frame) x 4704` bytes per
    channel, earliest sample in the most significant bit).
  - `null`: decode everything and discard the result. This measures decoding alone.
  - `raw DIR`: write each channel's track audio to `DIR/TT-C.dsd`, where `TT` is the track
    number in decimal, zero-padded to at least two digits (`01` to `99`, then `100` to `255`),
    and `C` the zero-based channel index in decimal. A track with no track audio writes no
    file. This measures decoding and writing.
- **Errors.** When the program cannot decode the area it prints one error code on a line to
  stderr and exits non-zero. The code is the one of specification 7, 7.7.1 when the image is
  rejected; `no-such-area` (a code of this document only, not of specification 7) when the
  image is accepted but slot `A` yields no area, because the slot is unused or the area was
  skipped (7.7.2); and a code of the program's own for anything else, such as an unreadable
  file or bad arguments. Warnings MAY also be printed to stderr.
- The program reads the image only through ordinary file reads, and SHOULD print nothing
  else to stdout.

**Correctness gate.** Every timed run is preceded by a `hash` run on the same image, whose
hashes must equal the reference hashes for that disc. A run that fails the gate is reported
as a failure, not as a time.

**Release binaries.** For each release tag, the implementation's CI publishes `bench` for
`x86_64-unknown-linux-gnu` as a release asset named
`bench-<version>-x86_64-linux.tar.gz`, built in release mode for a generic x86-64 target
(no `target-cpu=native`), with a SHA-256 checksum asset alongside it. The checksum asset is named `<asset>.sha256` and
holds one line in the format of `sha256sum` (lower-case hex digest, two spaces, the asset's
file name).

An implementation MAY also publish, for each measuring machine named under Method, one asset
built for that machine's CPU, named `bench-<version>-x86_64-linux-<cpu>.tar.gz`, where
`<cpu>` is the compiler's name for that CPU (for example `znver5`), with its own checksum
asset. The measuring side runs the generic asset and, on each machine, that machine's asset
if there is one, and reports which asset each figure comes from.

A release MAY carry other builds of `bench` (for example for a microarchitecture level such
as `x86-64-v3`). They are not benchmark assets: their names MUST NOT match either pattern
above, and the measuring side ignores them.

## Method

- **Images:** real discs owned by the measuring side, copied to local storage and read
  from the page cache. They are not distributed. Each is described by its characteristics
  only (area, channels, frame format, duration). The set:

  | Disc | Area | Channels | Frame format | Duration |
  |---|---|---|---|---|
  | A | 2-channel | 2 | plain DSD | about 78 min |
  | B | 2-channel | 2 | DST | about 50 min |
  | C | multichannel | 5 | DST | about 46 min |
  | D | 2-channel and multichannel | 2 and 5 or 6 | DST | about 45 min each |

  A raw-sector image (2064-byte sectors, specification 1) is added to the set to check that
  sector-format handling costs nothing measurable.
- **Machines:** a 16-core/32-thread desktop CPU (AMD Ryzen AI Max+ 395, 128 GB) and a
  10-core/12-thread mobile CPU (Intel Core i5-1235U, 94 GB), Linux, each otherwise idle and
  used by one benchmark at a time.
- **Runs:** programs alternated (A, B, C, A, B, C, ...), at least 3 runs each, mean and
  spread reported. Output, where written, goes to local storage and is deleted between runs.
- **Measured:** wall-clock time, CPU time (user + system) and peak resident memory, per run.
- **Thread scaling:** `N` = 1, 8, 16 and all logical CPUs.
- **Two comparisons**, kept separate:
  1. **Decoding:** `bench ... --sink null` against the decoding work of the other readers,
     reported as frames per second (single thread) and as wall time (all threads).
  2. **Extraction:** the full job of reading an area and writing it out, against other
     readers writing DSF files. `bench` uses `--sink raw`; the difference in file format
     (raw DSD against DSF) is recorded with the figures.

## Baseline

Measured before any implementation from these specifications existed. Wall-clock seconds,
CPU seconds in brackets, all threads, writing DSF.

**10-core machine**, mean of 3 runs, programs alternated:

| | `sacd_extract` | reader P 0.2.1 | reader P after its DST work |
|---|---|---|---|
| Plain DSD, 2-channel (78 min) | 7.9 (6.0) | 2.2 (4.7) | 2.2 (4.7) |
| DST, 2-channel (50 min) | 43.4 (467) | 33.9 | **18.0** (193) |
| DST, 2-channel + 5-channel | 181.8 (1989) | 137.2 | **69.5** (758) |

**16-core machine**, mean of 3 runs, programs alternated (reader P 0.2.2 is the version with
its DST work):

| | `sacd_extract` | reader P 0.2.1 | reader P 0.2.2 |
|---|---|---|---|
| Plain DSD, 2-channel (78 min) | 3.67 (3.7) | 1.00 (2.9) | **1.00** (2.8) |
| DST, 2-channel (50 min) | 7.83 (244) | 6.07 (189) | **2.87** (87) |
| DST, 2-channel + 6-channel (50 min each) | 35.02 (1096) | 27.31 (854) | **11.75** (368) |

**One core**, 16-core machine, DST 2-channel area of 50 min (226 497 frames), mean of 3:

| | ms per frame |
|---|---|
| `sacd_extract` | 0.680 |
| FFmpeg 8.1.3 DST decoder (estimate: its DST time minus its DSD-to-PCM time on the same area) | 0.669 |
| reader P 0.2.2 | **0.227** |

**Single thread, decoding only** (ms per frame, mean of 3):

| | reader P 0.2.1 | reader P after its DST work |
|---|---|---|
| 16-core machine, 2 channels | 0.410 | **0.225** |
| 16-core machine, 6 channels | 1.152 | **0.670** |
| 10-core machine (performance core), 2 channels | 1.19 | **0.74** |
| 10-core machine (performance core), 6 channels | 3.02 | **2.09** |

`sacd_extract` is a generic x86-64 build of sacd-ripper's extractor, and reader P is pysacd
(GPL). On DST discs every reader keeps all cores busy, so the cost per frame decides the
result. These figures are updated when the baseline readers change.

## Targets

For a reader built from these specifications, measured as above, with every correctness
gate passed. "The fastest baseline reader" is whichever reader in the baseline is fastest on
that workload when the measurement is made.

| Workload | Target |
|---|---|
| DST, all threads, wall time | at least 1.5 times faster than `sacd_extract`; aim: at least as fast as the fastest baseline reader (today 2.4 to 2.6 times `sacd_extract`) |
| DST, single thread | at least as many frames per second as the fastest baseline reader |
| Plain DSD, wall time | at least 2.5 times faster than `sacd_extract` |
| Thread scaling (DST) | at least 12 times faster at 16 threads than at 1 |
| Memory | peak resident memory bounded independently of disc duration |

## Results

Each round of measurements is appended here, newest first, with the implementation version,
the machines, the conditions and every figure, and the distance to each target.

### Round 2: scarletbook 0.2.0 (2026-10-30)

**Program:** `bench` from the scarletbook-rs v0.2.0 release assets, generic and `znver5`
builds; checksums verified, build provenance attestation verified.

**Machine and conditions:** as round 1 (16-core machine, the same five discs, local copies,
output in a RAM file system, mean of 3 runs, thread scaling at 1 thread: 1 run). One virtual
machine was running but idle: it used 99 CPU seconds over the 925-second round (about 0.1 of
a core). Only `bench` was measured; the figures of `sacd_extract` and reader P 0.2.2 are
round 1's, on the same machine and discs.

**Correctness gate:** passed. Both builds produced the reference hashes for every track and
channel of all 7 areas.

**Extraction, all threads** (wall seconds, CPU seconds in brackets; round 1 figure in
italics):

| | `sacd_extract` | reader P 0.2.2 | `bench` generic | `bench` znver5 |
|---|---|---|---|---|
| A | 3.67 | 1.00 | 1.19 (1.2) *1.31* | 1.03 (1.0) *1.19* |
| B, 2-channel | 7.83 | 2.87 | 3.11 (97) *5.76* | **2.58** (80) *5.10* |
| B, both areas | 35.02 | 11.75 | 14.27 (445) *26.08* | **11.30** (352) *23.03* |
| C, both areas | 18.43 | 6.96 | 8.63 (250) *14.60* | **6.60** (188) *12.98* |
| D | n/a | 2.35 | 2.60 (81) *4.66* | **2.22** (68) *4.26* |
| E | n/a | 4.23 | 4.83 (150) *8.43* | **4.02** (124) *7.69* |

**Decoding only** (`--sink null`, znver5): A 0.48 s, B 2-channel 2.59 s, B both areas 11.23 s,
C both areas 6.25 s, D 2.20 s, E 4.00 s.

**One core** (DST 2-channel, 50 min, 226 497 frames): `bench` 0.215 ms per frame (znver5) and
0.239 (generic), against 0.278 for `bench` 0.1.0, 0.227 for reader P 0.2.2 and 0.680 for
`sacd_extract`.

**Thread scaling** (disc B, both areas, znver5, extraction):

| Threads | 1 | 8 | 16 | 32 |
|---|---|---|---|---|
| `bench` wall | 183.2 s | 24.3 s (7.5×) | 13.4 s (13.7×) | 11.6 s (15.8×) |
| `bench` CPU | 183 s | 194 s | 213 s | 353 s |

**Peak memory:** `bench` 6 to 33 MB, flat with disc length.

**Against the targets** (znver5 build; generic in brackets):

| Target | Result |
|---|---|
| DST at least 1.5× `sacd_extract`, all threads | met: 2.8× to 3.1× (generic 2.1× to 2.5×) |
| DST aim: as fast as the fastest baseline reader | met: 1.04× to 1.11× reader P 0.2.2 (generic 0.81× to 0.92×) |
| DST single thread: frames per second of the fastest baseline reader | met: 0.215 against 0.227 ms per frame (generic 0.239: not met) |
| Plain DSD at least 2.5× `sacd_extract` | met: 3.6× (generic 3.1×) |
| Thread scaling at least 12× at 16 threads | met: 13.7× |
| Memory bounded independently of duration | met |

Reader P 0.2.2 was measured as a build for the measuring machine's CPU, so it compares with
the znver5 build; the generic build compares with portable builds of other readers.

### Round 1: scarletbook 0.1.0 (2026-10-30)

**Program:** `bench` from the scarletbook-rs v0.1.0 release assets, both the generic x86-64
build and the build for the measuring machine's CPU (`znver5`), checksums verified.

**Machine and conditions:** the 16-core machine, otherwise idle (no virtual machines
running), Linux; images on local storage, in the page cache; output in a RAM file system;
programs alternated, mean of 3 runs (thread scaling at 1 thread: 1 run).

**Discs** (real, not distributed):

| | Areas | Frame format | Sector format | Duration |
|---|---|---|---|---|
| A | 2-channel | plain DSD | 2048 | 79 min |
| B | 2-channel + 6-channel | DST | 2048 | 50 min each |
| C | 2-channel + 5-channel | plain DSD + DST | 2048 | 46 min each |
| D | 2-channel | DST | 2064 | 41 min |
| E | 2-channel | DST | 2064 | 76 min |

**Correctness gate:** passed. Both builds produced the reference hashes for every track
and channel of all 7 areas.

**Extraction, all threads** (wall seconds, CPU seconds in brackets; `bench` writes raw DSD,
the others DSF; `sacd_extract` does not read 2064-byte sector images):

| | `sacd_extract` | reader P 0.2.2 | `bench` generic | `bench` znver5 |
|---|---|---|---|---|
| A | 3.67 (3.7) | 1.00 (2.8) | 1.31 (1.3) | 1.19 (1.2) |
| B, 2-channel | 7.83 (244) | 2.87 (87) | 5.76 (144) | 5.10 (130) |
| B, both areas | 35.02 (1096) | 11.75 (368) | 26.08 (660) | 23.03 (607) |
| C, both areas | 18.43 (515) | 6.96 (205) | 14.60 (344) | 12.98 (324) |
| D | n/a | 2.35 (71) | 4.66 (118) | 4.26 (107) |
| E | n/a | 4.23 (131) | 8.43 (215) | 7.69 (196) |

**Decoding only** (`bench --sink null`, all threads, znver5 build): A 0.45 s, B 2-channel
4.57 s, B both areas 20.94 s, C both areas 11.40 s, D 3.77 s, E 6.89 s.

**One core** (DST 2-channel, 50 min, 226 497 frames): `bench` 0.278 ms per frame (znver5)
and 0.298 (generic), against 0.227 for reader P 0.2.2 and 0.680 for `sacd_extract`.

**Thread scaling** (disc B, both areas, znver5 build, extraction):

| Threads | 1 | 8 | 16 | 32 |
|---|---|---|---|---|
| `bench` wall | 263.8 s | 38.0 s (6.9×) | 23.4 s (11.3×) | 23.2 s (11.4×) |
| `bench` CPU | 263 s | 284 s | 331 s | 610 s |
| reader P 0.2.2 wall | 203.4 s | 26.6 s (7.7×) | 14.5 s (14.0×) | 11.8 s (17.2×) |

**Peak memory:** `bench` 6 to 32 MB, flat with disc length; reader P 86 to 181 MB;
`sacd_extract` 6 to 50 MB.

**Against the targets:**

| Target | Result |
|---|---|
| DST at least 1.5× `sacd_extract`, all threads | met narrowly: 1.52× (B both areas), 1.54× (B 2-channel), 1.42× (C both areas, not met) |
| DST aim: as fast as the fastest baseline reader | not met: reader P 0.2.2 is 1.8 to 2.0× faster on every DST case |
| DST single thread: frames per second of the fastest baseline reader | not met: 0.278 ms per frame against 0.227 (reader P is 1.22× faster) |
| Plain DSD at least 2.5× `sacd_extract` | met: 3.1× (znver5), 2.8× (generic) |
| Thread scaling at least 12× at 16 threads | not met: 11.3× |
| Memory bounded independently of duration | met |

**Real-world survey (same round, the 10-core machine).** Every SACD image of the measuring
side's collection: 113 images, 128 areas, 100 hours of audio (100 2-channel plain DSD areas,
13 2-channel DST, 15 multichannel DST; 14 of the areas on 2064-byte sector images). One run per
program per area, order rotated per disc, all threads, each image read from the collection's
disks once first (6.4 min for all of them) and then from the page cache. Reader P from its
0.2.2 wheel, `bench` generic build. **Correctness gate: passed on all 128 areas.**

| Areas | Count | Audio | reader P vs `sacd_extract`, median [range] | `bench` vs `sacd_extract`, median [range] | reader P vs `bench` |
|---|---|---|---|---|---|
| 2-channel, plain DSD | 100 | 74.6 h | 3.6× [2.9–4.0] | 2.5× [2.0–2.7] | 1.5× [1.2–1.8] |
| 2-channel, DST | 13 | 13.1 h | 2.3× [2.3–2.3] | 1.4× [1.4–1.5] | 1.6× [1.5–1.7] |
| multichannel, DST | 15 | 12.3 h | 2.4× [2.3–2.6] | 1.5× [1.3–1.6] | 1.7× [1.6–1.8] |

Whole collection, the 114 areas `sacd_extract` reads: `sacd_extract` 41.3 min, reader P 16.2 min
(2.5×), `bench` 26.5 min (1.6×). All 128 areas: reader P 19.0 min, `bench` 31.1 min.

**Notes for the next round:**
- From 16 to 32 threads, `bench`'s wall time is unchanged while its CPU time nearly
  doubles (331 s to 610 s); reader P gains 23% over the same step.
- The single-thread figure on real disc frames (0.278 ms) differs from figures on corpus
  frames; only real-disc figures are used for the targets.
- The release assets carry checksums but no build provenance attestation.
