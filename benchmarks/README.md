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
(no `target-cpu=native`), with a SHA-256 checksum asset alongside it. A second asset built
for the measuring machine's CPU MAY be added, labelled as such.

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

**16-core machine** (measured with reader P 0.2.1; the DST rows predate its DST work):

| | `sacd_extract` | reader P 0.2.1 |
|---|---|---|
| Plain DSD, 2-channel (78 min) | 3.2 (2.9) | 1.3 (2.0) |
| DST, 2-channel (50 min) | 7.9 (245) | 7.8 (243) |
| DST, 2-channel + 5-channel | 35.6 (1094) | 33.4 (1020) |

**Single thread, decoding only** (ms per frame, mean of 3):

| | reader P 0.2.1 | reader P after its DST work |
|---|---|---|
| 16-core machine, 2 channels | 0.410 | **0.225** |
| 16-core machine, 6 channels | 1.152 | **0.670** |
| 10-core machine (performance core), 2 channels | 1.19 | **0.74** |
| 10-core machine (performance core), 6 channels | 3.02 | **2.09** |

On one core of the 16-core machine, a DST 2-channel disc took 154.7 s with `sacd_extract` and
51.6 s with reader P after its DST work.

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

No rounds yet.
