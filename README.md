# sacd-spec

An implementation-free specification of reading Super Audio CD (Scarletbook) disc images, and
a conformance corpus to test readers against. It is written so that someone who has never seen
any SACD code can write a correct reader from this repository alone.

## Contents

| | |
|---|---|
| [spec/00-conventions.md](spec/00-conventions.md) | terms, notation, byte and bit order, outcome vocabulary |
| [spec/01-disc-image-layout.md](spec/01-disc-image-layout.md) | 2048, 2054 and 2064-byte sector formats, detection, LSN addressing, where the TOCs are |
| [spec/02-master-toc-and-text.md](spec/02-master-toc-and-text.md) | the master TOC, album set, catalogue numbers, genres, date, locales, text channels, character sets |
| [spec/03-area-toc.md](spec/03-area-toc.md) | 2-channel and multichannel area TOCs, TOC-1 and TOC-2, tracks, timecodes, ISRCs, track text |
| [spec/04-audio-multiplexing.md](spec/04-audio-multiplexing.md) | audio sectors, packets, frame reassembly, frames on the timeline, tracks |
| [spec/05-dst.md](spec/05-dst.md) | the complete DST frame syntax and decoding process |
| [spec/06-ps3-protocol.md](spec/06-ps3-protocol.md) | the PS3 SACD Ripper network protocol (optional) |
| [spec/07-validation-and-limits.md](spec/07-validation-and-limits.md) | every limit and check, the required behaviour, diagnostic codes |
| [spec/charsets/](spec/charsets/) | normative mapping tables for the CJK character sets of specification 2 |
| [corpus/](corpus/README.md) | synthetic images, `manifest.json` with expected TOCs and decoded-audio hashes |
| [benchmarks/](benchmarks/README.md) | how readers are compared for speed: the benchmark program's interface, method, baseline, targets and results (informative) |
| [PROVENANCE.md](PROVENANCE.md) | what was consulted to write each part |

Start with specification 0, then 1 to 4 in order. Specification 5 is needed to decode DST
frames, which any area can contain (the audio sectors, not the area TOC, say whether a frame is
DST-coded), and 6 only for reading from a PS3. Specification 7 collects every check in
one place.

## The clean-room process

This repository is the hand-off document of a two-party clean-room process.

1. **The specification side** (this repository) was written by a party that was allowed to
   read an existing SACD implementation: pysacd, a GPL-licensed toolkit, including its own
   prior clean-room description of DST and its protocol notes, plus the public DSF and DSDIFF
   specifications. [PROVENANCE.md](PROVENANCE.md) lists, for each section, exactly what was
   consulted.
2. **The implementation side** writes a new, permissively licensed reader using only this
   repository (and general, non-SACD references). It never reads any SACD or DST
   implementation. When something here is ambiguous or missing, it asks for a fix to this
   repository rather than looking elsewhere.

To keep the wall intact, everything in this repository obeys these rules:

- **No source code** in any language, and no pseudo-code that mirrors the structure of an
  existing implementation. The specifications use normative prose, syntax tables (field,
  offset, size, type, meaning), state descriptions and mathematical formulas.
- **No identifiers, comments or structure copied** from pysacd, sacd-ripper, FFmpeg or any
  other implementation, beyond the format's own field names and signatures.
- **Recorded provenance** for every section.
- The **corpus generators are not here**: only their output (images and expected values) is.

Before release, the specifications were cold-read by a reviewer who had access to this
repository only, and who listed every ambiguity, undefined term, missing limit and place where
behaviour would have to be guessed. Each finding was fixed, and the review repeated until a
pass found nothing material.

## Using the corpus

[corpus/README.md](corpus/README.md) describes the manifest and how to compare a reader's
results with it: TOC equality, warning sets, and per-channel SHA-256 hashes of each track's
decoded DSD.

## Status

Version 1.0-draft. The behaviour specified for malformed input is deliberately conservative
and deterministic: for every image that is not rejected, two conforming readers produce the
same TOC, the same set of required warnings (the codes of specification 7 that are not marked
optional) and the same audio; for a rejected image they produce the same error code. Not
covered by that promise: a sector format chosen by the caller instead of detected
(specification 1, 1.4); optional (SHOULD or MAY) warnings, warnings under codes of the reader's
own, and warnings emitted before a rejection; and, over the network, the cases specification 6
declares unspecified, and the choice of network error code and retry policy, which it only
recommends. Frame counts
are recommended (SHOULD) rather than required, and a reader that reports them reports the same
values. Points that could not be specified with full confidence are listed in
[PROVENANCE.md](PROVENANCE.md#open-points).

## Licences

- The specifications and all other documentation: **Creative Commons Attribution 4.0
  International** (CC BY 4.0), see [LICENSE](LICENSE).
- The conformance corpus (`corpus/`, images and manifest) and the character-set tables
  (`spec/charsets/`): **CC0 1.0 Universal** (public
  domain dedication), see [corpus/LICENSE](corpus/LICENSE).

The test data is CC0 so that implementations under any licence can copy it into their own
test suites.
