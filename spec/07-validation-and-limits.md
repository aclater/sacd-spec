# 7. Validation and limits

Disc images, and data from network servers, are hostile input. This specification lists, for
every length, count, offset and position a reader takes from an image, its valid range and the
required behaviour when it is out of range. It also defines the diagnostic codes. It is
normative. The outcome vocabulary (reject the image, skip the area, skip or refuse the track,
replace the frame, warn, clamp) is defined in [00-conventions.md](00-conventions.md), 0.8.

Where this document and specifications 1 to 6 disagree, this is an error in the
specifications; report it. Until it is fixed, the more detailed text of specifications 1 to 6
prevails over the summary tables here. The section references say where each rule is defined.

## 7.1 General requirements

1. A reader MUST NOT read outside the image, outside a buffer it has read, or outside a
   sector, whatever any value in the image says.
2. A reader MUST bound every allocation by a constant of these specifications or by the size
   of the image before making it. No value read from an image may, by itself, make a reader
   allocate more than: 128 sectors for each of the two copies of an area TOC, 65 536 bytes for a frame under assembly,
   `6 x 4704` bytes for a decoded frame, plus per-track output proportional to the track's
   length (and the same for a frame range, specification 4, 4.7.1), which 7.5 bounds by 7 frames of at most `6 x 4704` bytes per existing audio sector
   (about 100 times the size of the area's audio sectors), the frames held between delivery
   and output (specification 4, 4.5 item 5), saved state for beginning later in an area
   (specification 4, 4.2 item 7: a constant number of bytes per existing audio sector, plus at
   most one frame under assembly of 65 536 bytes for each saved state that holds one, with the
   number of such states a constant of the reader or chosen by the caller), and diagnostics: each warning is of bounded size,
   and their number is bounded by a constant per structure of these specifications plus at most
   one `audio-sector-malformed` per audio sector processed (specification 4, 4.4.5), which a
   reader MAY cap further.
3. A reader MUST terminate on every input, and the work per structure is bounded by
   constants of these specifications: the sub-structure walk takes at most 128 steps per area;
   disc text takes at most 16 strings of at most 4096 bytes; area text at most 4 strings; track
   text at most 255 tracks x 255 items x 4096 bytes per area (tracks may share one item list,
   so this bound is not proportional to the area TOC size, but it is a constant), plus one
   zero-skipping pass of at most `E` bytes per track; sector
   processing is at most one pass over the area's existing audio sectors (specification 4,
   4.2) for each track, frame range or stored frame list (specification 4, 4.6.1) the caller asks
   for, plus at most one further pass per area to establish saved state (specification 4, 4.2
   item 7), made once however many reads then use it; a read that begins after `S0` (4.2
   item 7), including the sectors it processes only to establish the frame under assembly,
   counts as one pass; a provisional reading (4.2 item 8) MAY also read, to choose `L`, at most
   a constant of the reader times `log2(n + 1)` sectors, where `n` is the number of the area's
   existing audio sectors (4.8 item 3); a reader that produces an
   area's tracks in increasing order of their start frames needs only one pass for all of them
   (specification 4, 4.5 item 4); and DST decoding of a frame is
   proportional to `NB x Nch` plus the frame length.
4. A reader MUST NOT crash or abort on any input. Errors are reported with the codes of 7.7.
5. A failure in one area MUST NOT affect the other area, and a bad frame MUST NOT affect any
   other frame.

## 7.2 Image and master TOC

| Value | Valid range | Out of range | Code | Spec |
|---|---|---|---|---|
| Image size | any | for each format, fewer than `510 x P + D + 8` bytes means that format cannot match; if no format matches | `not-scarletbook` (reject) | 1.4 |
| `SACDMTOC` at LSN 510 | present in one of the three formats | absent | `not-scarletbook` (reject) | 1.4 |
| Master TOC sectors (LSN 510 to 519) | all exist | any missing | `master-toc-truncated` (reject) | 2.2 |
| Version major (master offset 8) | 0 or 1 | 2 to 255 | `master-toc-version-unsupported` (reject) | 2.2 |
| Version minor | any | | | 2.2 |
| Album set size, sequence number | any | reported as stored | | 2.3.1 |
| Catalogue numbers | 16 bytes, any content | decoded as fixed text fields | | 2.6.5 |
| Genres | any | reported as stored | | 2.4 |
| Date | any | reported as stored | | 2.3.2 |
| Text channel count | 0 to 8 | above 8: list capped at 8 | | 2.5.2 |
| Character set code | 0 to 7 | high bits ignored | | 2.6.2 |
| `SACDText` at LSN 511 | present | absent and count > 0 | `master-text-missing` (warn) | 2.5.4 |
| Disc text offsets | 1 to 2047 | 0 or at least 2048: item absent | | 2.5.4 |
| Text string length | up to 4096 bytes, within the buffer | cut | | 2.6.4 |
| Invalid byte sequences in text | none | replaced by U+FFFD | | 2.6.3 |
| Code points below U+0020 in text | none | replaced by U+0020 | | 2.6.4 |
| `SACD_Man` at LSN 519 | present | absent | `manufacturer-missing` (warn) | 2.7 |
| Master TOC copies at 520 and 530 | identical to 510 | not used | | 1.5.1 |
| Area slots | at least one yields an area | none | `no-audio-areas` (reject) | 3.1 |

## 7.3 Area TOC

| Value | Valid range | Out of range | Code | Spec |
|---|---|---|---|---|
| Slot TOC-1 LSN, TOC size | both non-zero | either 0: slot unused (silent) | | 3.1 |
| Area TOC size | 1 to 128 sectors | above 128 | `area-toc-too-large` (skip the area) | 3.2 |
| TOC-1 sectors | all exist | any missing | `area-toc1-unreadable` (warn; use TOC-2 if it is set and readable, else skip the area with `area-toc-unreadable`) | 3.2 |
| TOC-2 LSN | 0 (no backup) or any | | | 3.2 |
| TOC-2 sectors (if LSN non-zero) | all exist | any missing | `area-toc2-unreadable` (warn) | 3.2 |
| TOC-1 and TOC-2 content | identical | different | `area-toc-copies-differ` (warn; use TOC-1) | 3.2 |
| Both copies | at least one readable | none | `area-toc-unreadable` (skip the area) | 3.2 |
| Area signature | `TWOCHTOC` or `MULCHTOC` | anything else | `area-signature-invalid` (skip the area) | 3.3 |
| Area kind versus slot | any combination | | | 3.1 |
| Area version | any | not validated | | 3.3 |
| Sample frequency code | 4 | others reported; audio still DSD64 | `sample-frequency-unknown` (SHOULD warn) | 3.4.2 |
| Frame format | 0, 2, 3 | others reported; not used for decoding | | 3.4.2 |
| Channel count | 1 to 6 for audio | others: TOC reported, no audio | `channel-count-unsupported` (refuse each track) | 3.4.1, 4.8 |
| Loudspeaker configuration | any | reported | | 3.4.1 |
| Total play time | any timecode | | | 3.3 |
| Track count | 0 to 255 | all valid | | 3.9.1 |
| Area text channel count (offset 80) | 0 to 10 | above 10: list capped at 10 | | 3.5 |
| Area text offsets | 1 to `E - 1` | 0 or at least `E`: absent | | 3.6 |
| Sub-structure walk | ends at an unknown signature or the end of the TOC | | | 3.7.0 |
| `SACDTRL1` and `SACDTRL2` | both found by the walk | either missing | `area-track-list-missing` (skip the area) | 3.7.0 |
| `SACD_IGL` extent | within the TOC (when there is at least one track) | extends past the end | `area-toc-malformed` (skip the area) | 3.7.3 |
| Track text offset | 0 (none), or `q + 4 <= E` with `q` the offset plus the position of the `SACDTTxt` structure | `q + 4 > E` | `track-text-offset-invalid` (warn; no text for the track) | 3.8.1 |
| Track text item count | 0 to 255 | items starting past `E - 2` do not exist | | 3.8.1 |
| Track text item type | 0x01 to 0x07, 0x81 to 0x87 | others (with a non-empty string) | `track-text-type-unknown` (warn; item ignored) | 3.8.1 |
| Audio LSN range | first <= last | last < first | `area-audio-range-reversed` (skip the area) | 3.9.2 |
| Audio LSN range versus image | any | sectors past the image are missing | | 4.2 |
| Track start + duration | at most the play time | above it | `track-past-playtime` (clamp; warn) | 3.9.2 |
| Track start LSN | within the audio range | outside it | `track-lsn-outside-area` (warn; value kept) | 3.9.2 |
| Track LSN length | any | reported only | | 3.7.1 |
| Track order, overlap, gaps | any | all valid | | 3.9.2 |
| ISRC | any 12 bytes | decoded as a fixed text field; empty means none | | 3.7.3 |

## 7.4 Audio sectors and frames

| Value | Valid range | Out of range | Code | Spec |
|---|---|---|---|---|
| Packet records per sector | 0 to 7 | cannot exceed | | 4.3.1 |
| Frame records per sector | 0 to 7 | cannot exceed | | 4.3.1 |
| Packets' total length | fits in the sector | a packet past the end: rest of sector ignored | `audio-sector-malformed` (SHOULD warn) | 4.4.3 |
| Frame start with no frame record left | does not occur | no frame starts | `audio-sector-malformed` (SHOULD warn) | 4.4.3 |
| Frame size during assembly | up to 65 536 bytes | larger: frame discarded | `audio-sector-malformed` (SHOULD warn) | 4.4.3 |
| Packet types | 2, 3, 7 | others ignored like 3 and 7 | | 4.3.2 |
| DST sector count | equals the frame's audio packet count | a frame is complete only while its remaining count is exactly 0 when the next frame starts or the area ends (4.4.2); an incomplete frame is discarded, and its timeline position stays missing unless another accepted frame fills it | | 4.4.2 |
| Plain frame size | exactly `C x 4704` | otherwise never complete (missing) | | 4.4.2 |
| Frame timecode | strictly increasing | a frame not above the previous accepted one is discarded | | 4.5 |
| Timeline position never filled | | filled with 0x69 | (counted as missing) | 4.6 |
| DST frame | valid by 5.6 | invalid | replace the frame with 0x55 (counted as invalid) | 4.6, 5.6 |

## 7.5 Tracks and output

| Value | Valid range | Out of range | Code | Spec |
|---|---|---|---|---|
| Area channel count | 1 to 6 | otherwise | `channel-count-unsupported` (refuse the track) | 4.8 |
| Track duration | at least 1 frame | 0 | `track-empty` (skip the track) | 4.8 |
| Track end frame | at most 7 x the number of the area's audio sectors that exist in the image | above it | `track-exceeds-capacity` (refuse the track) | 4.8 |

The checks are made in the order of the table; the first that applies decides.

## 7.6 DST frames

Every rule of specification 5, section 5.6, makes the frame invalid and replaces it (7.4).
The bounds that keep decoding safe:

| Value | Valid range | Spec |
|---|---|---|
| Frame length `FB` | at least 2 bytes; bounded by the reassembly limit | 5.2.1 |
| Header length | at most `8 x FB` bits | 5.2.1 |
| Segment resolution | 1 to 4576 (filters), 1 to 4700 (ptables) | 5.3.2 |
| Explicit segment lengths | at least `Lmin` and at most the remaining length | 5.3.2 |
| Explicit lengths per channel | filters: 4 (shared) or 3 (per channel); ptables: 7 | 5.3.2 |
| Table numbers | at most the running count | 5.3.3 |
| Tables of each kind | at most `2 x Nch` | 5.3.3 |
| Coding method | 0 to 2, and less than the order or length minus 1 | 5.3.5, 5.3.6 |
| Filter coefficients | -256 to 255 | 5.3.5 |
| Ptable entries | 1 to 128 | 5.3.6 |
| Unary part of a Rice code | ends within the frame | 5.3.8 |
| Unread code bits | at most 7 | 5.5.7 |
| Channels `Nch` | 1 to 6 (guaranteed by 7.5) | 5.1.1 |

## 7.7 Diagnostic codes

### 7.7.1 Errors that reject the image

| Code | Condition | Spec |
|---|---|---|
| `not-scarletbook` | no `SACDMTOC` at LSN 510 in any sector format (or in the format a caller chose, or over the network); over the network, also a disc size of at most 510 sectors | 1.4, 2.2, 6.7 |
| `master-toc-truncated` | the image ends before LSN 519 is complete; over the network, a disc size of 511 to 519 sectors | 2.2, 6.7 |
| `master-toc-version-unsupported` | master TOC version major above 1 | 2.2 |
| `no-audio-areas` | no area slot yields an area | 3.1 |

### 7.7.2 Warnings that skip an area

Each is reported with the slot (`2ch` or `multichannel`) it concerns.

| Code | Condition | Spec |
|---|---|---|
| `area-toc-too-large` | area TOC size above 128 sectors | 3.2 |
| `area-toc-unreadable` | neither TOC copy can be read | 3.2 |
| `area-signature-invalid` | the chosen copy's signature is neither `TWOCHTOC` nor `MULCHTOC` | 3.3 |
| `area-track-list-missing` | no `SACDTRL1` or no `SACDTRL2` in the walk | 3.7.0 |
| `area-toc-malformed` | a `SACD_IGL` extends past the end of the area TOC | 3.7.3 |
| `area-audio-range-reversed` | last audio LSN below the first | 3.9.2 |

If more than one applies to a slot, only the first in the order of specification 3 is
reported, because processing of the area stops there.

### 7.7.3 Other warnings

| Code | Reported with | Condition | Spec |
|---|---|---|---|
| `master-text-missing` | | no `SACDText` at LSN 511 and the text channel count is above 0 | 2.5.4 |
| `manufacturer-missing` | | no `SACD_Man` at LSN 519 | 2.7 |
| `area-toc1-unreadable` | slot | TOC-1 cannot be read | 3.2 |
| `area-toc2-unreadable` | slot | TOC-2 is set but cannot be read | 3.2 |
| `area-toc-copies-differ` | slot | both copies read and differ | 3.2 |
| `track-text-offset-invalid` | slot, track | a track text offset points too close to the end | 3.8.1 |
| `track-text-type-unknown` | slot, track | a text item of unknown type with a non-empty string | 3.8.1 |
| `track-past-playtime` | slot, track | a track ends after the area play time; clamped | 3.9.2 |
| `track-lsn-outside-area` | slot, track | a track start LSN lies outside the audio range | 3.9.2 |
| `audio-sector-malformed` | slot, LSN | (SHOULD) a malformed audio sector | 4.4.5 |
| `master-toc-copies-differ` | | (MAY) a master TOC copy at LSN 520 or 530 that can be read in full (one read of 10 sectors) differs from LSN 510 to 519 in any of its 20 480 bytes; reported at most once (a copy that cannot be read is not compared) | 1.5.1 |
| `sample-frequency-unknown` | slot | (SHOULD) sample frequency code other than 4 | 3.4.2 |

The track reported with a warning is the track **number** (1 to the track count, specification
3, 3.9.1). A warning of a given code is reported at least once per (slot, track) it applies to. A reader
MAY report repeated occurrences separately. `audio-sector-malformed` and
`master-toc-copies-differ` follow their own rules instead (specification 4, 4.4.5;
specification 1, 1.5.1 item 3).

A reader MUST NOT report a code of 7.7.1 to 7.7.3 when its condition does not hold. Warnings
for conditions these tables do not list MUST use other codes. The corpus therefore compares, for each image that is **not** rejected, the set of
(code, slot, track) triples a reader reports from 7.7.2 and 7.7.3, other than the optional
`audio-sector-malformed`, `master-toc-copies-differ` and `sample-frequency-unknown`, for
equality with the expected set. For a rejected image only the
error code of 7.7.1 is compared.

### 7.7.4 Per-track outcomes

These codes are reported per track (or per frame range, specification 4, 4.7.1 item 4) as
the reason a track has no audio. They are not
warnings: they are not part of the warning set of 7.7.3, and a reader that also reports them
through its warning mechanism must not let them count as warnings for comparison.

| Code | Outcome | Condition | Spec |
|---|---|---|---|
| `channel-count-unsupported` | refuse the track | the area's channel count is not 1 to 6 | 4.8 |
| `track-empty` | skip the track | duration 0 | 4.8 |
| `track-exceeds-capacity` | refuse the track | the track ends beyond 7 frames per existing audio sector | 4.8 |

### 7.7.5 Per-frame outcomes

Per-frame outcomes have no warning code requirement. A reader SHOULD count, per track (and per frame range), the
frames that were missing (filled with 0x69) and the DST frames that were invalid (filled with
0x55); the corpus manifest gives both counts.

## 7.8 Network input

Data from a PS3 SACD Ripper server is bounded by specification 6, section 6.7, and its
failures have the codes of 6.7.1 (`network-connection-lost`, `network-protocol-error`,
`network-disc-error`, `network-disc-too-large`, `network-disc-changed`). Sectors received over
the network are then subject to every rule of this specification.

## 7.9 Real-disc envelope (informative)

A survey of 110 commercial discs (made while writing an existing reader) found area TOCs of 5
to 41 sectors, at most 0.6 frames per audio sector, titles under 100 bytes, and no track ending
after its area's play time. The bounds above are far outside that envelope, so they never
affect a real disc.
