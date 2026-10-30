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
2. The reader processes the logical sectors `S0, S0 + 1, ..., S1'` in increasing LSN order,
   each exactly once. If `S0 > S1'` no sector is processed.
3. Sectors past the end of the image are not an error: frames they would have carried are
   missing (4.7).
4. The sectors in that range are processed as audio sectors whatever they contain. If the range
   covers TOC sectors or other non-audio data (possible in a hostile image), those bytes are
   parsed as audio sectors and yield whatever this specification defines for them.
5. The result is defined as if every sector in the range is processed. A reader MAY stop
   early once it can prove that no later sector can change any frame it still has to produce
   (4.6 item 4 gives one such case). The track start LSNs of specification 3 MUST NOT change
   the result: a reader that uses them to seek MUST still produce exactly the frames this
   specification defines.

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

## 4.5 Delivery and the timeline

Delivered frames are placed on the area's **timeline**, which has one slot for each frame
number. The reader keeps a **next acceptable frame number** `a`, starting at 0 for the area.

1. When a frame with frame number `t` is delivered:
   - if `t >= a`, it is **accepted**: it fills slot `t`, and `a = t + 1`;
   - otherwise it is discarded.
2. Consequently each slot holds at most one frame, the timeline is filled in increasing order,
   and a frame whose timecode goes backwards or repeats is discarded. After a frame with a very
   large timecode is accepted, every later frame with a smaller timecode is discarded.
3. Slots that no accepted frame fills are **missing**.
4. Once a frame with number `t` has been accepted, no slot below `t + 1` can change. A reader
   that needs only slots below some bound `u` MAY stop processing sectors as soon as it
   delivers a frame with number `t >= u`: the result for slots below `u` is the same.

## 4.6 Frame contents

The content of slot `t` is `C x 4704` bytes in interleaved layout (specification 0, 0.6):

| Slot state | Content |
|---|---|
| plain frame | the frame's bytes, unchanged (its size is exactly `C x 4704` by 4.4.2) |
| DST frame, decoded successfully | the decoder output (specification 5) with `Nch = C` and the frame's bytes as input (its byte count is the frame length `FB`) |
| DST frame, invalid | every byte 0x55 (**replace the frame**); the frame counts as an invalid DST frame |
| missing | every byte 0x69; the frame counts as missing |

A DST frame is decoded independently of every other frame. An invalid frame does not affect
its neighbours.

## 4.7 Tracks

1. Track `i` covers the frame numbers `[s_i, s_i + d_i)`, where `s_i` and `d_i` are its start
   and duration after the checks of specification 3, 3.9.2 (so `s_i + d_i` is at most the
   area's play time).
2. The **audio of track `i` for channel `c`** is the concatenation, for each frame number `t`
   from `s_i` to `s_i + d_i - 1` in order, of the 4704 bytes of channel `c` in slot `t`: bytes
   `c, c + C, c + 2C, ...` of the slot's content. It is `d_i x 4704` bytes long, and each byte
   holds 8 samples, earliest in the MSB.
3. Tracks may overlap; overlapping tracks share the same slots.
4. Frames between the end of one track and the start of the next (pauses), and frames outside
   every track, belong to no track. Informative: a reader that offers "tracks with pauses" can
   extend track `i` to start at frame 0 (for the first track) or at its own start (for the
   others), and to end at the next track's start (or at the play time for the last track),
   but never before its own end. The corpus checks only the ranges of item 1.

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
how many of its frames were missing and how many were invalid DST frames.

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
