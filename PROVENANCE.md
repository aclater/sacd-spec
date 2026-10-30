# Provenance

This file records, for each part of this repository, which sources were consulted to write it.
It is part of the clean-room record: an implementer who reads only this repository can see
what crossed the wall.

## Sources

| Ref | Source | Version | Licence | How it was used |
|---|---|---|---|---|
| P-README | pysacd `README.md` | commit 8834da1 (2026-10-29) | GPL-2.0-or-later | outline of what a reader parses |
| P-LIB | pysacd `docs/library.md` ("What's parsed", "Image formats") | same | same | outline of specifications 1 to 3 |
| P-DST | pysacd `docs/dst-bitstream.md` (a prior clean-room prose description of DST, sections 1 to 8 with errata) | same | same | basis of specification 5 |
| P-PS3 | pysacd `docs/ps3-protocol.md` | same | same | basis of specification 6 |
| P-SEC | pysacd `docs/security.md` | same | same | starting point of specification 7 |
| P-CR | pysacd `docs/clean-room.md` | same | same | process description |
| P-TOC | pysacd `src/pysacd/toc.py` (TOC reader) | same | same | field offsets and checks of specifications 1 to 3; behaviour on malformed TOCs |
| P-DEMUX | pysacd `csrc/demux.c`, `csrc/demux.h`, `csrc/sacd.h` (audio sector demultiplexer) | same | same | behaviour of specification 4 |
| P-ENGINE | pysacd `csrc/sacdconv.c` (reader thread, frame staging, fill values, capacity and channel checks), `src/pysacd/_native.py` | same | same | frame timeline and track mapping of specification 4; fill values |
| P-DSTC | pysacd `csrc/dstdec.c`, `csrc/dstdec.h` (DST decoder, LGPL-2.1-or-later) | same | LGPL-2.1-or-later | checking specification 5 against the decoder that produced the corpus hashes: segment limits, frame-length rules, header overrun |
| P-REMOTE | pysacd `src/pysacd/remote.py` (PS3 client and test server) | same | GPL-2.0-or-later | wire encoding details and client limits of specification 6 |
| P-SBCOMPAT | pysacd `src/pysacd/sbcompat.py` (channel identifiers, speaker configuration names) | same | same | the informative channel order of specification 3 |
| P-GEN | pysacd `tests/sacdgen.py` (synthetic image builder, hostile-image corpus) | same | same | building the corpus images; the layouts it writes confirmed specifications 2 and 3 |
| P-DSTENC | pysacd `tests/dstenc.py`, `tests/native/dstenc.c` (test DST encoder), `tests/test_dst_full.py` (feature frames and invalid frames) | same | same | building the DST frames of the corpus |
| P-HOSTILE | pysacd `tests/test_hostile_iso.py` | same | same | expected behaviour of hostile images |
| P-CLAUDE | pysacd `CLAUDE.md` | same | same | constraints (clean room for DST, hostile input) |
| DSF, DSDIFF | the public DSF and DSDIFF file format specifications | | public documents | permitted, but not consulted: no section depends on them |
| Python | Python 3 standard library codec behaviour (`latin-1`, `shift_jisx0213`, `euc_kr`, `gb2312`, `big5`, `str.isspace`) | 3.14 | PSF | exact character-set behaviour of specification 2, 2.6 |

Not consulted: the source of sacd-ripper or `sacd_extract`, FFmpeg's source tree itself (but
FFmpeg-derived code was read, see the disclosure below), the Philips/ISO MPEG-4 reference DST
decoder, or any other SACD or DST implementation.

Every source is pinned to the version that was read. pysacd has changed since commit 8834da1;
a later change to this repository that consults pysacd again records, in its entry below, the
commit it read, and the table above keeps the commit of the original reading.

**Disclosure.** P-DSTC, pysacd's `csrc/dstdec.c`, which was read, is itself a port of FFmpeg's
DST decoder (LGPL-2.1-or-later), extended with parts written from P-DST. Reading it therefore
means reading FFmpeg-derived code. It was used only to check specification 5 against the
behaviour that produced the corpus hashes, and some normative rules were aligned with it: at
most 4 (shared) or 3 (per channel) explicit filter lengths, at most 7 explicit ptable lengths,
frames of 0 or 1 byte invalid, and header overrun (including inside a Rice code) invalid. No
code, identifier or comment was taken from it; those rules are stated as behaviour. P-DST, the
prose description specification 5 is based on, was itself written by a party that studied the
Philips/ISO reference decoder, according to P-CR; it contains no code. Whoever relies on this
clean room should weigh these two facts.

## Per section

### README.md, spec/00-conventions.md

Written new. Process description informed by P-CR and the task brief. Fill values (0.6) from
P-ENGINE (0x69 for missing frames, 0x55 for undecodable DST frames) and P-DST 3.11. Timecode
formula from P-TOC.

### spec/01-disc-image-layout.md

- 1.2 to 1.4 (sector formats, addressing, detection order): P-LIB "Image formats", P-TOC
  (layout table and detection loop), P-GEN (raw layouts of the corpus).
- 1.3 item 1 (a trailing partial stored sector does not exist) and item 3 (all-or-nothing
  reads): P-TOC (sector count and short-read behaviour).
- 1.5 (disc layout, master TOC copies at 520 and 530): P-GEN (writes three copies), P-TOC
  (reads LSN 510 only). 1.5.1 item 3 (what a master TOC copy comparison compares, reported once)
  was decided for this specification in answer to issue #4; pysacd does not compare the copies.
- 1.6 (largest disc): P-REMOTE comment on dual-layer sizes.

### spec/02-master-toc-and-text.md

- 2.1 to 2.3 (fields and offsets): P-TOC, cross-checked with P-GEN's writer.
- 2.2 (version check): P-TOC.
- 2.4 (genre tables): P-TOC (names), restated as a table.
- 2.5 (locales, which language): P-TOC, P-LIB ("only the first text language is decoded").
- 2.6.2 (character sets): P-TOC (code to codec mapping) and P-LIB. The mapping tables in
  `spec/charsets/` were generated by enumerating every one- and two-byte sequence through
  Python's `shift_jisx0213`, `euc_kr`, `gb2312` and `big5` codecs (the codecs P-TOC uses) and
  recording what decodes; they are data, not code. The divergent positions listed in 2.6.2 were
  found by comparing those codecs with `shift_jis_2004`, `cp932`, `cp949`, `gbk`, `gb18030`,
  `cp950` and `big5hkscs`.
- 2.6.3 (decoding and invalid sequences): new. The rule was checked to reproduce Python's
  replacement behaviour for those four codecs exactly on 800 000 random byte strings, apart from
  the EUC-KR composition sequences beginning with 0xA4D4. **Deviation:** Python's codec applies
  KS X 1001:1998 composition to those; this specification forbids it (decided after the fourth
  cold read, so that every reported value is deterministic). The corpus contains no such string.
- 2.6.4 (text strings: 4096-byte bound, controls to space, spaces kept): P-TOC.
- 2.6.5 (fixed text fields and the whitespace set W): P-TOC; W established from Python's
  definition of whitespace in the U+0000 to U+00FF range.
- 2.7: P-TOC.

### spec/03-area-toc.md

- 3.1 (slots, kind from signature, both slots allowed to yield the same kind): P-TOC; P-GEN
  hostile cases `master-areas-same-toc`, `area-mch-signature-in-stereo-slot`.
- 3.2 (TOC-1 and TOC-2, readability decides, no fallback on invalid content, 128-sector bound):
  P-TOC, P-SEC.
- 3.3 to 3.6 (header fields, loudspeaker configurations, frame formats, sample frequency):
  P-TOC, P-GEN; channel order: P-SBCOMPAT (informative).
- 3.7 (sub-structure walk and sizes; last `SACD_IGL`, `SACDTRL1`, `SACDTRL2` win; first
  `SACDTTxt` wins): P-TOC.
- 3.8 (track text items, zero skipping, the 4096-byte cut note): P-TOC.
- 3.9 (clamping to the play time, LSN warning): P-TOC, P-SEC.

### spec/04-audio-multiplexing.md

- 4.2 (scan the whole area, clipped to the image): P-ENGINE and P-DEMUX for the sector loop.
  **Deviation:** pysacd's extraction narrows the scan using track LSNs and fails the whole area
  with a read error when the audio range runs past the image; this specification instead
  defines the result as a whole-area scan clipped to the image (frames past the end are
  missing), so that results do not depend on which tracks are requested.
- 4.3 to 4.4 (sector header, packet and frame records, reassembly, completeness, malformed
  sectors): P-DEMUX. The rules for reporting `audio-sector-malformed` (4.4.5 items 1 to 3:
  processed sectors only, keyed by slot and LSN, optional cap with a count) were decided for
  this specification in answer to issue #5.
- 4.5 (timeline, next acceptable frame number): P-ENGINE (frame staging). The equivalence of
  early stopping (4.5 item 4) was derived for this specification.
- 4.6 (frame contents and fills): P-ENGINE, P-DST.
- 4.7 (tracks as frame ranges; pauses informative): P-TOC (track ranges with and without
  pauses), P-ENGINE.
- 4.8 (channel count 1 to 6, empty tracks, capacity of 7 frames per sector): P-ENGINE,
  P-HOSTILE, P-SEC. **Deviations:** pysacd applies the capacity bound to all requested outputs
  together, while this specification applies it per track; and pysacd's extraction tool stops
  with an error on a zero-length track, while this specification skips the track.

### spec/05-dst.md

Rewritten from P-DST. What changed relative to P-DST:

- removed every comparison with other decoders (P-DST 3.12 and 7) and every reference to
  implementation identifiers;
- folded the errata of P-DST section 8 into the normative rules: minimum segment length
  checked in bytes (`R x L >= Lmin/8`), at most 7 explicit ptable lengths in both modes, the
  end-of-frame check;
- aligned the rules with the decoder that produced the corpus hashes (P-DSTC): frames of 0 or
  1 byte are invalid; header overrun (including inside a Rice code) is invalid; explicit
  filter lengths at most 4 shared and 3 per channel;
- made the plain-frame zero-fill normative;
- moved the encoder notes (P-DST 6) to an informative annex;
- added rule 17 (initial code value 4095 is invalid) after the second cold read. **Deviation:**
  pysacd's decoder does not check this and decodes such a frame to an implementation-specific
  result; the corpus case `dst-code-all-ones` follows this specification.

The syntax tables, prediction coefficient tables, arithmetic decoder, history, half-probability
rule, X bit and output packing are those of P-DST, checked against P-DSTC.

### spec/06-ps3-protocol.md

Rewritten from P-PS3: the protobuf schema code block, the implementation names and the file
paths were replaced by prose and tables. The error classes of 6.7.1, the handling of repeated
fields, unexpected wire types and missing response types, and the server's behaviour before an
open were made explicit after the cold read (P-REMOTE behaviour where it defines them). Wire details (varint and key encoding, 10-byte
negatives, wire types accepted) and client limits (field size, field count, disc size, result
range, deadlines, reconnection checks) from P-REMOTE and P-SEC. Server read-size behaviour from
P-PS3 and P-REMOTE's test server. Not verified against a real PS3 (P-README says the same of
pysacd).

### spec/07-validation-and-limits.md

Consolidated from specifications 1 to 6. Starting points: P-SEC ("Hostile images" table,
"PS3 network protocol"), P-HOSTILE. Diagnostic codes are new names for conditions that P-TOC
reports as free-text warnings. 7.9 (real-disc survey) from P-SEC. Diagnostics in the
allocation bound of 7.1 item 2 were added in answer to issue #5.
The work bound of 7.1 item 3 (one pass per track or range requested) and 4.2 item 6 were
restated in answer to issue #18.

### corpus/

- Images: built with P-GEN (its disc builder, its raw-layout re-packer and its 95-image
  hostile corpus, taken unchanged) and P-DSTENC (feature frames and invalid frames at 2, 5 and 6
  channels, synthetic sigma-delta test signal). New cases (all with `origin: new`) were made by
  calling those builders with other parameters or by patching their output: character sets 0
  to 7, album set, no text, full text, TOC-2 fallback, coded DST in raw layouts, a 5-channel
  plain disc, invalid DST frames inside discs, truncated DST frames, invalid CJK text, and
  (after the first cold read) the CJK table positions where published mappings disagree and DST
  frames breaking only the arithmetic-code rules; and (after the second cold read) an area
  without `SACD_IGL`, frame format 3, track-text warnings in a skipped area, and a DST frame
  whose initial code value is 4095. For that last case the expected values of its track were
  set from specification 5 (0x55 fill, one invalid frame), because pysacd's decoder does not
  apply rule 17; every other expected value is pysacd's output.
- Case names of the malformed cases taken from the hostile-image corpus are the names that
  corpus uses, for traceability. The images' own text (for example the catalogue number `PYSACD-TEST`
  and titles naming pysacd) is test data written by that generator, not an identifier used by
  these specifications.
- Expected values: computed with P-TOC and the decoding engine (P-ENGINE, P-DEMUX, P-DSTC),
  configured to follow specification 4 where it deviates (see 4.2 and 4.8 above), mapping
  pysacd's warning texts to the codes of specification 7. Hashes of the DST feature cases were
  checked against the encoder's input signal.
- The generator scripts are kept outside this repository.

### benchmarks/README.md

Written new, after the specifications and corpus. The benchmark program's interface and the
method are new; the hash definition is referenced from corpus/README.md. The baseline table
and the single-thread DST figures are measurements published in pysacd `docs/performance.md`
(commit 8834da1), restated as numbers only: no implementation detail, profile or technique
from any reader is described. Disc descriptions are characteristics only. The baseline was
updated with the figures pysacd `docs/performance.md` publishes at commit 8a179b5 (2026-10-30),
again numbers only. Round 1 of the results was measured by the specification side against
compiled binaries only: `bench` from scarletbook-rs v0.1.0's release assets (its source was not
read), `sacd_extract`, pysacd 0.2.1 and 0.2.2 (commit 8a179b5 and later, release v0.2.2), and FFmpeg
8.1.3; numbers, conditions and the comparison with the targets only. Round 2 measured only `bench` from scarletbook-rs v0.2.0's release assets (source not read)
and reuses round 1's reference figures. The `bench` output for tracks without audio, area selection by slot,
`no-such-area` and track-number padding were decided for this document in answer to issue #9;
no source was consulted. The names of the checksum and CPU-specific assets, and other builds
of `bench` (Release binaries), were decided in answer to issue #14; no source was consulted.

## Open points

These could not be specified with full confidence. Each is specified deterministically, so
readers agree, but the choice may not match the Scarletbook standard, which was not available.

1. **Character set codes 0 and 7** have no known assignment; they decode as ISO 8859-1.
2. **Character set 3** ("Shift JIS for music", RIS-506) is decoded as Shift_JIS-X0213 (JIS X
   0213:2000), including 0x5C as the yen sign and 0x7E as overline. The real RIS-506 table may
   differ from JIS X 0213 in details, and some discs may have been authored with plain Shift JIS
   or CP932 in mind; the tables in `spec/charsets/` fix one answer.
3. **EUC-KR composition sequences** (beginning with 0xA4D4) are not composed; a disc that relies
   on them shows filler and jamo replacement characters instead of syllables.
4. **TOC-1 readable but invalid**: the area is skipped rather than falling back to TOC-2. This
   matches existing readers; a reader that falls back would be more forgiving.
5. **Area kind versus slot**: the kind comes from the signature, so a disc can report two areas
   of the same kind. The standard may require the 2-channel slot to hold a `TWOCHTOC`.
6. **Frame format field** (3.4.2) is informative; the per-sector DST flag decides decoding.
   "DSD 3 in 16" is assumed to use the same interleaved layout as "DSD 3 in 14"; the corpus case
   `dsd-3in16-format` follows that assumption, which is not confirmed by a real disc.
7. **Field meanings not specified**: track start and duration flag bytes, track offset, track
   attribute, area mute flags, maximum available channels, genre table 2 names. They are
   reported as stored (the track attribute as its low 4 bits).
8. **Multichannel channel order** is informative, taken from the channel labels existing tools
   write.
9. **DST segment limits**: the reference behaviour on a ninth ptable segment in shared mode was
   unusable, so it is invalid here (at most 8 ptable segments). Filter limits are 5 segments
   shared and 4 per channel. These match the decoder that produced the corpus.
10. **DST at rates above DSD64** is described only informatively.
11. **PS3 protocol** has been tested only against a simulator, not real hardware.
12. **Master TOC copies** at LSN 520 and 530 are not used, even when LSN 510 is damaged.

## Wall review

The cold read flagged sections whose step-by-step descriptions could resemble an
implementation's structure (3.7.0, 3.8.1, 4.4.3, 5.3.2, 5.5.7, annex 5.A.6). They were reviewed:
each describes the format's semantics (the order in which bytes are interpreted and the state
the format defines), uses names chosen for this specification, and none was transcribed from
code. The DST syntax tables follow the conventional MPEG syntax-table style. The quirks they
record (for example the item-skipping rule of 3.8.1) are normative behaviour, kept so that
readers agree on hostile input.

### Names (third cold read)

- DST syntax element names in specification 5 (`same_segm_all_ch`, `coded_pred_order` and so
  on) are those of P-DST, which states that they are descriptive names chosen for that
  document, in the style of the MPEG syntax, and not the identifiers of any implementation.
- Master and area TOC field names (album catalogue number, maximum byte rate, area mute flags,
  track attribute and so on) and the corresponding manifest keys are the format's own field
  names, written in snake case; the manifest uses the US spelling `catalog` as the format's
  field name does. Manifest keys that are not format field names (`toc_lsn`, `playtime_frames`,
  `audio_start_lsn`, `start_flags`, `duration_flags`, `length_sectors`, `first_frame`,
  `end_frame`, `missing_frames`, `invalid_dst_frames`) were chosen for this repository.
- Implementation-derived constants that are normative here (explicit segment-length limits,
  frames of 0 or 1 byte invalid, the 0x69 and 0x55 fills, the 65 536-byte frame limit, the
  128-sector area TOC limit, 7 frames per sector) are recorded as such in this file; they are
  facts about the agreed behaviour, not code.
- The PS3 server behaviour of specification 6, 6.5 (the block-size table and the need for a
  priming read) describes observable protocol behaviour that clients must handle.

## Licensing note

Specifications 5 and 6 are rewrites of documentation in pysacd, whose repository is licensed
GPL-2.0-or-later. They are published here under CC BY 4.0 by the owner of both repositories.
