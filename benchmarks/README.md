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
| `bench decode IMAGE --area A --threads N --sink S` | decodes every track of area `A` of image `IMAGE` and exits 0, or exits non-zero with an error class from specification 7 on stderr |

- `IMAGE` is a path to a disc image in any of the sector formats of specification 1.
- `A` is `2ch` or `mc` (the 2-channel or the multichannel area, specification 3).
- `N` is the number of worker threads, 1 or more; `0` means one per logical CPU. With `N = 1`
  the program MUST do all decoding on a single thread.
- `S` is the sink:
  - `hash`: for every track, print one line to stdout: the track number, a tab, then the
    SHA-256 of each channel's track audio, in channel order, separated by tabs. The
    track audio and its hash are exactly as defined for the `sha256` field in
    [corpus/README.md](../corpus/README.md) (`(end_frame - first_frame) x 4704` bytes per
    channel, earliest sample in the most significant bit).
  - `null`: decode everything and discard the result. This measures decoding alone.
  - `raw DIR`: write each channel's track audio to `DIR/TT-C.dsd`, where `TT` is the
    two-digit track number and `C` the zero-based channel index. This measures decoding
    and writing.
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

Measured before any implementation from these specifications existed, on the machines above.
Wall-clock seconds, CPU seconds in brackets, all threads, writing DSF:

| | 16-core: `sacd_extract` | 16-core: reader P | 10-core: `sacd_extract` | 10-core: reader P |
|---|---|---|---|---|
| Plain DSD, 2-channel (78 min) | 3.2 (2.9) | 1.3 (2.0) | 7.9 (6.0) | 2.2 (4.7) |
| DST, 2-channel (50 min) | 7.9 (245) | 7.8 (243) | 42.0 (457) | 39.7 (451) |
| DST, 2-channel + 5-channel | 35.6 (1094) | 33.4 (1020) | 175.2 (1958) | 159.3 (1817) |

`sacd_extract` is a generic x86-64 build of sacd-ripper's extractor; reader P is pysacd.
Single-thread DST decoding cost on the 16-core machine: about 0.41 ms per 2-channel frame
and 1.15 ms per 6-channel frame. On DST discs both readers keep every core busy, so the cost
per frame decides the result. These figures are updated when the baseline readers change.

## Targets

For a reader built from these specifications, measured as above, with every correctness
gate passed:

| Workload | Target |
|---|---|
| DST, all threads, wall time | at least 1.5 times faster than `sacd_extract` |
| DST, single thread | at least as many frames per second as the fastest baseline reader |
| Plain DSD, wall time | at least 2.5 times faster than `sacd_extract` |
| Thread scaling (DST) | at least 12 times faster at 16 threads than at 1 |
| Memory | peak resident memory bounded independently of disc duration |

## Results

Each round of measurements is appended here, newest first, with the implementation version,
the machines, the conditions and every figure, and the distance to each target.

No rounds yet.
