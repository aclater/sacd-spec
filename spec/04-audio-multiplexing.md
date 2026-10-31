# 4. Audio sector multiplexing

This specification defines how audio frames are carried in an area's audio sectors: the
sector header, packets and frame information, how frames are reassembled across sectors, how
frames are placed on the area's timeline, and how that timeline maps to tracks. It is
normative. Conventions are in [00-conventions.md](00-conventions.md); the area fields used
here are in [03-area-toc.md](03-area-toc.md).

## 4.1 Overview

1. An area's audio is a sequence of **frames** of 1/75 s (specification 0, 0.5). Each frame
   is either **plain DSD** (exactly `C x 4704` bytes in interleaved layout, `C` the area's
   channel count) or **DST-coded** (a variable number of bytes, decoded by specification 5).
2. Frames are carried in **audio sectors**, the logical sectors from the area's first audio
   LSN `S0` to its last audio LSN `S1`. A frame may be spread over several sectors, and a
   sector may carry pieces of several frames.
3. Each sector carries up to 7 **packets**. Each packet is a run of bytes of one kind (audio,
   supplementary data or padding). Audio packets carry frame bytes.
4. Each frame that begins in a sector has a **frame information** record in that sector,
   giving the frame's timecode.

## 4.2 Which sectors are read

1. Let `N` be the number of sectors in the image (specification 1, 1.3) and
   `S1' = min(S1, N - 1)`.
2. The result is defined as if the reader processes the logical sectors
   `S0, S0 + 1, ..., S1'` in increasing LSN order, each exactly once. If `S0 > S1'` no sector is processed.
3. Sectors past the end of the image are not an error: frames they would have carried are
   missing (4.5 item 3, 4.6).
4. The sectors in that range are processed as audio sectors whatever they contain. If the range
   covers TOC sectors or other non-audio data (possible in a hostile image), those bytes are
   parsed as audio sectors and yield whatever this specification defines for them.
5. The result is defined as if every sector in the range is processed. A reader MAY stop
   early once it can prove that no later sector can change any frame it still has to produce
   (4.5 item 4 gives one such case). The track start LSNs of specification 3 MUST NOT change
   the result: a reader that uses them to seek MUST still produce exactly the frames this
   specification defines.
6. A reader MAY process the sectors of an area more than once, for example once per track.
   Every complete pass gives the same frames, because 4.4 and 4.5 depend only on the sectors
   and their order. Specification 7, 7.1 item 3 bounds the number of passes.
7. A reader MAY begin processing at a sector `L` with `S0 < L <= S1' + 1`, instead of at `S0`,
   if it begins with the frame under assembly (4.4.1) that processing `S0` to `L - 1` would
   leave there. That frame under assembly depends only on `C` and the bytes of those sectors,
   so a reader can establish it during an earlier pass of its own over them (item 6), or by
   beginning at an earlier sector where it knows the frame under assembly to be none. Let
   `a_L` be the next acceptable frame number (4.5) that processing `S0` to `L - 1` would leave.
   From such a beginning, a reader produces only positions numbered `u` or more, for a `u`
   that it knows to be at least `a_L`, and it begins with a next acceptable frame number of
   its own choice that is at most `u`. It then produces exactly the frames this specification
   defines for those positions (4.5 item 6). Positions numbered below `a_L` are never produced
   from such a beginning. The stored frame list (4.6.1 item 2) is not produced this way.

   Informative: choosing `L` in any other way, for example from the track start LSNs
   (item 5), from timecodes found by searching, or by going back a fixed number of sectors,
   gives in general neither the frame under assembly nor a bound on `a_L`, because a single
   earlier sector can change either (4.5 item 2). Item 8 describes reading without that
   state.
8. A reader MAY, when its caller asks for it, begin processing at a sector `L` with
   `S0 < L <= S1'` without the state of item 7: with no frame under assembly and a next
   acceptable frame number of `u`, producing only positions numbered `u` or more. This is
   **provisional reading**. With `a_L` and the frame under assembly at `L` as in item 7, the
   positions it produces are exactly those of this specification when both:

   - (a) `a_L <= u`, and
   - (b) the frame under assembly at `L` is none or has a frame number below `u`.

   A reader MUST report the positions it produces this way as provisional, and MUST NOT
   report them as the positions of this specification unless it has established (a) and (b),
   for example by processing `S0` to `L - 1` afterwards. A reader that establishes (a) and (b)
   SHOULD then report the provisional positions as confirmed, and one that establishes that
   either fails SHOULD report them as possibly different. The reader chooses `L` as it likes
   (for example from timecodes found by searching, from the track start LSNs, in proportion
   to the play time, or from **locator samples**: pairs of a sector and the frame number of
   its first frame record, saved from an earlier read). The choice affects only how often (a)
   and (b) hold. A locator sample only steers that choice and never makes a position exact, so
   it needs no check beyond ignoring a sample whose sector is outside `S0` to `S1'`.

   A reader MAY also mark a provisional reading as **suspect** when it has reasons of its own
   to expect (a) or (b) to fail, before any pass has settled them. The mark is informative:
   a suspect reading is still provisional, is settled as confirmed or possibly different
   exactly as any other, and the mark has no effect on that outcome. The reasons are the
   reader's own; examples are frame numbers that do not increase between the sectors it read
   while choosing `L`, and an area whose TOC places its audio past the end of the image or
   over another area's TOC or audio sectors. A reader MAY also, for such an area or reading,
   not read provisionally at all and produce the positions as items 2 to 7 do.

   Informative:
   - Under (a) and (b), the two readers differ only in the frame under assembly at `L`, until
     the first start packet at or after `L` (or the end of the area). There, that frame is
     delivered or discarded with a frame number below `u`, which cannot fill a position
     numbered `u` or more. From there on, the frame under assembly is the same as for a reader
     beginning under item 7, and 4.5 item 6 applies.
   - (a) and (b) both hold whenever every frame that starts before `L` (4.4.3 item 3.3) has a
     frame number below `u`. On an image whose frames start in increasing frame number order,
     as on a well-formed disc, that is whenever `L` is at or before the sector where the
     first frame numbered `u` or more starts.
   - When (a) or (b) fails, the positions numbered `a*` or more are still exact, where `a*` is
     the next acceptable frame number of a reader processing from `S0` just after it has
     delivered or discarded the frame under assembly at `L`. Below `a*`, a provisional position
     can differ from this specification's whether or not that position is missing: a frame
     accepted before `L` is never seen by a provisional reader, and frames that this
     specification discards can be accepted.
   - No test of sectors at or after `L` can establish (a) or (b) (see the note to item 7).
9. What a pass over `S0` to `L - 1` establishes stays established for later readings,
   including after the reader has saved it and loaded it again, for example in another
   session. That covers the frame under assembly and `a_L` of item 7, and (a) and (b) of
   item 8. It holds only for an image whose sectors `S0` to `L - 1` are the bytes that pass
   processed. A reader MUST NOT rely on saved results for any other image. How it establishes
   that the bytes are the same is its own choice, for example by keeping the results with the
   image's identity and checks of its own; a check of sectors at or after `L` alone does not
   establish it (see the note to item 7). Once (a) and (b) are established for a sector `L`
   and a value `u_L`, they hold for every reading from `L` with `u >= u_L`, because both only
   bound values at `L` by `u` from above; the reader MAY report such readings as confirmed
   without processing `S0` to `L - 1` again.

## 4.3 Audio sector syntax

All positions are byte offsets within the 2048-byte logical sector.

### 4.3.1 Sector header

| Offset | Bits | Field | Meaning |
|---|---|---|---|
| 0 | bit 0 | DST flag | 1: frames that start in this sector are DST-coded; 0: plain DSD |
| 0 | bit 1 | reserved | |
| 0 | bits 2 to 4 | frame info count `F` | number of frame information records, 0 to 7 |
| 0 | bits 5 to 7 | packet info count `K` | number of packet information records, 0 to 7 |

### 4.3.2 Packet information records

`K` records of 2 bytes each, starting at offset 1. Taking the two bytes as a big-endian 16-bit
value `w`:

| Bits of `w` | Field | Meaning |
|---|---|---|
| 15 | frame start | 1: this packet holds the first bytes of a frame |
| 14 | reserved | |
| 11 to 13 | packet type | 2: audio; 3: supplementary data; 7: padding; others reserved |
| 0 to 10 | packet length | length of the packet's data in bytes, 0 to 2047 |

### 4.3.3 Frame information records

`F` records start at offset `1 + 2K`. Their size depends on the sector's DST flag:

| DST flag | Record size | Bytes |
|---|---|---|
| 0 | 3 | byte 0: minutes, byte 1: seconds, byte 2: frames (a timecode) |
| 1 | 4 | bytes 0 to 2: timecode; byte 3: bits 2 to 6 = **sector count** (0 to 31), other bits reserved |

The timecode's frame number (specification 0, 0.5) is the frame's position on the area's
timeline. The sector count of a DST frame is the number of audio packets that carry the frame
(on real discs, one packet per sector, so it is also the number of sectors the frame spans).

### 4.3.4 Packet data

1. Packet data starts at offset `H = 1 + 2K + F x r`, where `r` is the record size (3 or 4).
   `H` is at most 1 + 14 + 28 = 43.
2. The packets' data follow each other in record order with no gaps: packet 0 occupies
   `[H, H + len0)`, packet 1 the next `len1` bytes, and so on.
3. Bytes after the last packet, up to offset 2048, are ignored.

## 4.4 Frame reassembly

### 4.4.1 State

The reader keeps one **frame under assembly** for the area, which persists from one sector to
the next. It is either **none**, or a frame with:

- its timecode (frame number),
- its kind (DST or plain), fixed when the frame starts,
- its bytes so far,
- for a DST frame, a **remaining count**, a signed integer.

At the start of the area the frame under assembly is none.

### 4.4.2 Completeness

A frame under assembly is **complete** when it has at least 1 byte and:

- for a DST frame: its remaining count is exactly 0;
- for a plain frame: its size is exactly `C x 4704` bytes, where `C` is the area's channel
  count from the area TOC.

### 4.4.3 Processing one sector

For each sector, read the header (4.3.1), the `K` packet records and the `F` frame records.
Then process the packets in record order, keeping a **next frame record index** `x`, starting
at 0 for each sector, and a **position** starting at `H`:

1. If position + packet length > 2048, the sector is **malformed**: stop processing the
   sector here. This packet and every later packet of the sector are ignored. The frame under
   assembly keeps its state, including bytes added from earlier packets of this sector.
2. If the packet type is not 2 (audio), the packet is ignored apart from advancing the
   position (item 6).
3. If the packet is an audio packet with the frame start bit set:
   1. If a frame is under assembly and it is complete, **deliver** it (4.5).
   2. The frame under assembly becomes none (an incomplete frame is discarded).
   3. If `x < F`, a new frame starts: its timecode is that of frame record `x`, its kind is
      DST if the sector's DST flag is 1 and plain otherwise, its bytes are empty, and (for
      DST) its remaining count is the sector count of record `x`. Then `x = x + 1`.
   4. If `x >= F` (no record left), no frame starts, and the sector is malformed (processing
      continues).
4. If the packet is an audio packet and a frame is under assembly (including one just
   started in item 3):
   1. If its size plus the packet length would exceed 65 536 bytes, the frame is discarded
      (the frame under assembly becomes none) and the sector is malformed (processing
      continues).
   2. Otherwise the packet's bytes are appended to the frame and, for a DST frame, its
      remaining count decreases by 1.
5. An audio packet without the start bit when no frame is under assembly is ignored.
6. Advance the position by the packet length.

Frame records that no start packet consumed are ignored. The sector's DST flag affects only
the record size and the kind of frames that start in the sector: bytes appended to a frame
that started in an earlier sector are appended whatever this sector's flag is.

### 4.4.4 End of the area

After the last sector (4.2), if a frame is under assembly and it is complete, deliver it.

### 4.4.5 Malformed sectors

A reader SHOULD warn `audio-sector-malformed` for a malformed sector, with its LSN. The
corpus does not require this warning.

1. The warning covers the sectors the reader actually processes. A reader that stops early
   (4.2 item 5, 4.5 item 4) does not warn for sectors it did not process.
2. A sector SHOULD be reported once for its area, keyed by (slot, LSN). A reader that processes
   a sector more than once (for example one pass per track) MAY report it again.
3. Keeping one warning per malformed sector is an allocation proportional to the number of
   sectors processed, which specification 7, 7.1 item 2 allows. A reader MAY instead keep only
   the first `K` such warnings of an area in the order it processes sectors, for a constant
   `K` of its own (at least 1); it then SHOULD also report the number of malformed sectors it
   processed in that area, so that the omission is visible.
4. A reader that begins at a sector after `S0` (4.2 item 7) need not report the malformed
   sectors before that sector. It MAY report those that an earlier pass of its own found.
   Item 2 applies to every sector it reports again, including the sectors it processes only to
   establish the frame under assembly. A count reported under item 3 SHOULD say which sectors
   it covers.
5. A reader reading provisionally (4.2 item 8) reports the malformed sectors it processes from
   `L`, and SHOULD say that its counts begin at `L`. Once it has established the state at `L`,
   item 4 applies.

## 4.5 Delivery and the timeline

Delivered frames are placed on the area's **timeline**, which has one **position** for each frame
number. The reader keeps a **next acceptable frame number** `a`, starting at 0 for the area.

1. When a frame with frame number `t` is delivered:
   - if `t >= a`, it is **accepted**: it fills position `t`, and `a = t + 1`;
   - otherwise it is discarded.
2. Consequently each position holds at most one frame, the timeline is filled in increasing order,
   and a frame whose timecode goes backwards or repeats is discarded. After a frame with a very
   large timecode is accepted, every later frame with a smaller timecode is discarded.
3. Timeline positions that no accepted frame fills are **missing**.
4. Once a frame with number `t` has been accepted, no position below `t + 1` can change. A reader
   that needs only positions below some bound `u` MAY stop processing sectors as soon as it
   delivers a frame with number `t >= u`: the result for positions below `u` is the same.
5. Frames held between delivery and output need memory. A reader MUST bound it: an accepted
   frame is at most 65 536 bytes before decoding and `C x 4704` bytes after, and only frames
   inside a track or frame range (4.7.1) that is still to be produced need to be kept. A reader
   that produces the stored frame list (4.6.1 item 2) need keep only the frame it is delivering.
6. Informative: when processing begins at `S0`, `a` is 0 until the first frame is delivered
   and afterwards one more than the largest frame number delivered so far. In general, a
   delivered frame `t` is accepted exactly when `t` is at least the value of `a` at the
   beginning of processing and greater than every frame number delivered before it. So two
   readers that process the same sectors from the same frame under assembly, beginning with
   next acceptable frame numbers that are both at most `u`, accept exactly the same frames
   numbered `u` or more. 4.2 item 7 relies on this.

## 4.6 Frame contents

The content of timeline position `t` is `C x 4704` bytes in interleaved layout (specification 0, 0.6):

| Position state | Content |
|---|---|
| plain frame | the frame's bytes, unchanged (its size is exactly `C x 4704` by 4.4.2) |
| DST frame, decoded successfully | the decoder output (specification 5) with `Nch = C` and the frame's bytes as input (its byte count is the frame length `FB`) |
| DST frame, invalid | every byte 0x55 (**replace the frame**); the frame counts as an invalid DST frame |
| missing | every byte 0x69; the frame counts as missing |

A DST frame is decoded independently of every other frame. An invalid frame does not affect
its neighbours.

### 4.6.1 Stored content and DST equivalents

A reader MAY offer timeline positions without decoding them, for example to write DST frames
unchanged into a file format that carries them.

1. The **stored content** of position `t` is one of:
   - **DST**, with the bytes of the accepted DST frame (1 to 65 536 bytes, 4.4.3);
   - **plain**, with the `C x 4704` bytes of the accepted plain frame (4.4.2);
   - **missing**, when no frame was accepted at `t` (4.5 item 3).

   These are exactly the frames 4.4 and 4.5 accept; nothing is decoded. No stored content is
   offered for an area whose channel count is not 1 to 6 (4.8 item 1).
2. A reader offers stored content in one or both of these forms:
   - **The area's stored frame list:** every accepted frame of the area, in frame number
     order, with its frame number. It is defined by processing every sector of 4.2, with no
     early stop (4.2 item 5 does not apply), and it includes frames numbered `P` or more.
     Positions not in the list are missing. The list holds at most 7 frames per existing audio
     sector, because a sector can start at most 7 frames.
   - **The stored content of a track or frame range** `[a, b)` (4.7, 4.7.1): the stored content
     of each position from `a` to `b - 1`. The checks of 4.8 (or 4.7.1 item 4) apply first: a
     track or range they skip or refuse has no stored content, with the same code. Otherwise
     the reader SHOULD report how many of its positions are missing (4.7.1 item 6). It does
     not report invalid DST frames, because validity (specification 5, 5.6) is known only by
     decoding.
3. Each position has a **DST equivalent**: a DST frame (specification 5) that decodes with
   `Nch = C` to exactly the content 4.6 gives the position.
   - For DST, it is the stored bytes.
   - For plain, it is the byte 0x00 followed by the stored bytes.
   - For missing, it is the byte 0x00 followed by `C x 4704` bytes of 0x69.

   The byte 0x00 is a plain DST frame header (`DST_coded` = 0, stuffing 0), so by
   specification 5, 5.2.1 item 2 the frame decodes to the bytes that follow it.
4. A reader that writes the DST equivalents of a track's or a frame range's positions produces
   output whose decoding (specification 5, then 4.7 item 2) is exactly that track's or range's
   audio. A track or range that the checks of 4.8 skip or refuse has no DST equivalents written
   (item 2), so they cannot be used to produce fill beyond the area's capacity.
5. Informative: an invalid DST frame's stored bytes decode, by 4.6, to the 0x55 fill. Decoders
   outside these specifications may treat such frames differently (specification 5, 5.8
   item 1). A reader that writes DST equivalents for such a decoder MAY decode each DST frame
   and, if it is invalid, write the byte 0x00 followed by `C x 4704` bytes of 0x55 instead,
   which decodes to the same content.

## 4.7 Tracks

1. Track `i` covers the frame numbers `[s_i, s_i + d_i)`, where `s_i` and `d_i` are its start
   and duration after the checks of specification 3, 3.9.2 (so `s_i + d_i` is at most the
   area's play time).
2. The **audio of track `i` for channel `c`** is the concatenation, for each frame number `t`
   from `s_i` to `s_i + d_i - 1` in order, of the 4704 bytes of channel `c` in position `t`: bytes
   `c, c + C, c + 2C, ...` of the position's content. It is `d_i x 4704` bytes long, and each byte
   holds 8 samples, earliest in the MSB.
3. Tracks may overlap; overlapping tracks share the same timeline positions.
4. Frames between the end of one track and the start of the next (pauses), and frames outside
   every track, belong to no track. A reader that offers "tracks with pauses" uses the ranges
   of 4.7.1 item 5.

### 4.7.1 Frame ranges

1. The **play range** of an area is the frame numbers `[0, P)`, where `P` is the frame number
   of the total area play time (specification 3, 3.3). Accepted frames numbered `P` or more
   belong to no play range, and no range of this section produces them, but they still take
   part in acceptance (4.5).
2. A reader MAY produce the audio of any **frame range** `[a, b)` of an area with
   `0 <= a <= b <= P`, as the caller asks. Its audio for channel `c` is defined exactly as in
   4.7 item 2, with `a` and `b` in place of `s_i` and `s_i + d_i`. Track `i`'s own audio is the
   range `[s_i, s_i + d_i)`.
3. A request for a range with `a > b` or `b > P` is a caller error, not an outcome of the
   image. A reader MUST NOT produce audio for it; how it reports the error is its own choice.
4. The checks of 4.8 apply to a range as to a track, with `b - a` as the duration and `b` as
   the end frame: refuse with `channel-count-unsupported`; skip with `track-empty` when
   `a = b`; refuse with `track-exceeds-capacity` when `b > Q`.
5. The **track with pause** of track number `i` is the range `[a_i, b_i)`, where:
   - `a_i = 0` for track 1, and `a_i = s_i` for every other track;
   - `b_i = max(s_i + d_i, s_j)` when the track numbered `j = i + 1` exists, and
     `b_i = max(s_i + d_i, P)` for the last track (which is `P`, by specification 3, 3.9.2).

   Here `s_i` and `d_i` are as in 4.7 item 1, after the checks of 3.9.2. Tracks with pauses
   attach each pause to the track before it, and the frames before the first track to the
   first track. When tracks overlap or are out of order, the ranges given by these formulas
   may overlap; they are used as they are. Item 4 applies to each of them, so a track with
   duration 0 can still have a track with pause that produces audio.
6. A reader SHOULD report, for each range it produces, how many of its positions were
   missing and how many were invalid DST frames. These counts depend only on the range's
   positions (4.6), not on which sectors the reader processes or in how many passes (4.2
   items 5 to 7). Counts of provisional positions (4.2 item 8) are provisional too.

## 4.8 When no audio is produced

These checks apply to each track separately, in this order:

1. If the area's channel count `C` is not 1 to 6, **refuse the track** with
   `channel-count-unsupported`. (This applies to every track of such an area. Its TOC is
   still reported.)
2. If the track's duration `d_i` is 0, **skip the track** with `track-empty`.
3. Let `n = max(0, S1' - S0 + 1)` be the number of the area's audio sectors that exist
   (4.2), and the area's **capacity** `Q = 7 x n` frames. If `s_i + d_i > Q`, **refuse the
   track** with `track-exceeds-capacity`.

Item 3 bounds output by what the image could hold: a sector's header can start at most 7
frames, so an area cannot carry more frames than 7 per sector. Without it, a hostile TOC could
make a reader produce hours of fill from a few sectors. The bound applies to the track's end
frame number, not to its length.

Otherwise the track's audio is produced as 4.7 says, and the reader SHOULD report, per track,
how many of its frames were missing and how many were invalid DST frames, counted over the
track's positions as in 4.7.1 item 6. Frame ranges
(4.7.1) are checked in the same way (4.7.1 item 4).

## 4.9 Limits

| Item | Value | Behaviour when exceeded |
|---|---|---|
| Packets per sector | 0 to 7 (3-bit field) | cannot be exceeded |
| Frame records per sector | 0 to 7 (3-bit field) | cannot be exceeded |
| Packet length | 0 to 2047 (11-bit field) | packets past the end of the sector make it malformed (4.4.3 item 1) |
| Frame size | at most 65 536 bytes | the frame is discarded (4.4.3 item 4) |
| DST sector count | 0 to 31 | a frame whose packets do not match it is never complete |
| Plain frame size | exactly `C x 4704` | other sizes are never complete |
| Channels | 1 to 6 | no audio for the area's tracks |
| Track end frame | at most `7 x` existing audio sectors | the track is refused |

## 4.10 Informative notes

1. On real discs, frames are packed in order, a frame's timecodes increase by one from frame
   to frame, a DST frame spans as many sectors as its sector count says, and padding packets
   (type 7) fill the unused end of a sector.
2. "DSD 3 in 14" packs three 2-channel plain frames (3 x 9408 = 28 224 bytes) into 14 sectors,
   each carrying 2016 bytes of frame data after its header and records.
3. The two fill values let a test distinguish frames the disc lacks (0x69) from frames the
   disc has but that cannot be decoded (0x55).
