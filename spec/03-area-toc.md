# 3. Area TOCs

This specification defines the two area TOCs: how they are located and chosen (TOC-1 and
TOC-2), their header fields, speaker configurations, frame formats, the track list,
timecodes, LSNs, ISRCs, per-track genres and per-track text. It is normative. Conventions are
in [00-conventions.md](00-conventions.md); text decoding is in
[02-master-toc-and-text.md](02-master-toc-and-text.md), section 2.6.

## 3.1 Area slots

The master TOC (specification 2, 2.3) has two **area slots**:

| Slot | TOC-1 LSN | TOC-2 LSN | TOC size (sectors) |
|---|---|---|---|
| 2-channel | u32 at 64 | u32 at 68 | u16 at 84 |
| multichannel | u32 at 72 | u32 at 76 | u16 at 86 |

1. A slot is **used** if its TOC-1 LSN is not 0 and its TOC size is not 0. An unused slot is
   ignored silently (no warning).
2. A reader processes the 2-channel slot first, then the multichannel slot. Each used slot
   yields either an area or a skipped area (with a warning). The TOC lists the areas in slot
   order.
3. The kind of an area (2-channel or multichannel) comes from its area TOC signature (3.3),
   not from its slot. A slot can therefore yield an area of the other kind, and both slots can
   yield areas of the same kind (for example when both point at the same TOC). Readers MUST
   accept this and report each area with both its slot and its kind.
4. If no used slot yields an area, **reject the image** with `no-audio-areas`. This includes a
   disc where neither slot is used.

## 3.2 Reading an area TOC: TOC-1 and TOC-2

For a used slot with TOC size `T` sectors, TOC-1 LSN `A` and TOC-2 LSN `B`:

1. If `T > 128`, **skip the area** with `area-toc-too-large`. Nothing is read.
2. Read `T` sectors at `A` (specification 1, 1.3). If the read fails, **warn**
   `area-toc1-unreadable`.
3. If `B` is not 0, read `T` sectors at `B`. If the read fails, **warn**
   `area-toc2-unreadable`.
4. Choose the copy:
   - If the TOC-1 read succeeded, use TOC-1. If the TOC-2 read also succeeded and its bytes
     differ from TOC-1's in any position, **warn** `area-toc-copies-differ`.
   - Otherwise, if the TOC-2 read succeeded, use TOC-2.
   - Otherwise **skip the area** with `area-toc-unreadable`.
5. The chosen copy is the area TOC buffer `Z` of `E = T x 2048` bytes. Its **TOC LSN** is the
   LSN it was read from (`A` or `B`); readers report it.
6. Only readability selects the copy. If TOC-1 is readable but invalid (for example its
   signature is wrong), the area is skipped as 3.3 says; the reader MUST NOT fall back to
   TOC-2 in that case.
7. The value at offset 10 of the area TOC (3.3) is not used to size the read.

## 3.3 Area TOC header

Sector 0 of `Z`:

| Offset | Size | Type | Field | Meaning |
|---|---|---|---|---|
| 0 | 8 | chars[8] | signature | `TWOCHTOC` (2-channel area) or `MULCHTOC` (multichannel area) |
| 8 | 1 | u8 | version major | not validated |
| 9 | 1 | u8 | version minor | not validated |
| 10 | 2 | u16 | area TOC length | in sectors; not used |
| 12 | 4 | | reserved | |
| 16 | 4 | u32 | maximum byte rate | bytes per second; reported only |
| 20 | 1 | u8 | sample frequency code | 4 = 64 x 44 100 Hz |
| 21 | 1 | u8 | frame format | low 4 bits, see 3.4.2; high 4 bits not interpreted |
| 22 | 10 | | reserved | |
| 32 | 1 | u8 | channel count | number of audio channels, see 3.4.1 |
| 33 | 1 | u8 | area configuration | low 3 bits: loudspeaker configuration, see 3.4.1 |
| 34 | 1 | u8 | maximum available channels | reported only |
| 35 | 1 | u8 | area mute flags | reported only |
| 36 | 12 | | reserved | |
| 48 | 1 | u8 | track attribute | low 4 bits reported; high 4 bits not interpreted |
| 49 | 15 | | reserved | |
| 64 | 3 | tc | total area play time | the area's length in frames |
| 67 | 1 | | reserved | |
| 68 | 1 | u8 | track offset | reported only |
| 69 | 1 | u8 | track count | number of tracks, 0 to 255 |
| 70 | 2 | | reserved | |
| 72 | 4 | u32 | first audio LSN | first sector of the area's audio |
| 76 | 4 | u32 | last audio LSN | last sector of the area's audio (inclusive) |
| 80 | 1 | u8 | text channel count | |
| 81 | 7 | | reserved | |
| 88 | 40 | 10 x locale | locales | 10 locale entries (specification 2, 2.5.1) |
| 128 | 16 | | reserved | |
| 144 | 2 | u16 | area description offset | text, see 3.6 |
| 146 | 2 | u16 | area copyright offset | text |
| 148 | 2 | u16 | area description, phonetic, offset | text |
| 150 | 2 | u16 | area copyright, phonetic, offset | text |
| 152 | 1896 | | not interpreted | |

1. If the signature is neither `TWOCHTOC` nor `MULCHTOC`, **skip the area** with
   `area-signature-invalid`.
2. All other header fields are read and reported without validation, except as 3.4 and 3.9
   state.

## 3.4 Audio format fields

### 3.4.1 Channels and loudspeaker configuration

1. The channel count (offset 32) is the number of channels in every frame of the area. It is
   reported as stored, whatever its value. Audio can be produced only when it is 1 to 6
   (specification 4, 4.8).
2. The loudspeaker configuration is the low 3 bits of offset 33. Readers report the whole
   byte. The combinations below are defined; any other combination is reported as stored and
   has no defined loudspeaker layout:

   | Channel count | Configuration | Name | Channel order in a frame |
   |---|---|---|---|
   | 2 | 0 | stereo | left, right |
   | 5 | 3 | 5-channel (3/2) | front left, front right, centre, surround left, surround right |
   | 6 | 4 | 5.1-channel | front left, front right, centre, low-frequency effects, surround left, surround right |

   The channel order is informative. It is the order existing tools use when they label the
   channels of extracted audio; these specifications do not otherwise depend on it.
3. Neither field changes how audio is decoded, apart from the channel count.

### 3.4.2 Frame format and sample rate

1. The frame format is the low 4 bits of offset 21:

   | Value | Name | Frames |
   |---|---|---|
   | 0 | DST | DST-coded (specification 5) |
   | 2 | DSD 3 in 14 | plain DSD; three 2-channel frames fill 14 sectors |
   | 3 | DSD 3 in 16 | plain DSD; three frames fill 16 sectors |

   Other values are reported as stored and are not an error.
2. The frame format is **informative**. Whether a given frame is DST-coded or plain is decided
   by the audio sector that starts it (specification 4, 4.3), not by this field. A reader MUST
   decode audio by the sector flag even when it contradicts the frame format.
3. The sample frequency code 4 means 64 x 44 100 = 2 822 400 Hz. No other code is defined. A
   reader reports the stored code; whatever its value, the frame structure (4704 bytes per
   channel per 1/75 s) is unchanged and audio is decoded as DSD64. A reader SHOULD warn when the
   code is not 4; the corpus does not require that warning.

## 3.5 Locales and the text language

1. The reported area locale list is the first `min(n, 10)` of the 10 entries at offset 88,
   where `n` is the text channel count at offset 80. If `n` is 0 the list is empty.
2. All text in the area TOC (area text, 3.6, and track text, 3.8) is decoded with the
   character set of **area locale entry 0** (offset 90, low 3 bits), even if the text channel
   count is 0. The master TOC's locales do not apply to area text.
3. Only the first text channel is used (3.7, `SACDTTxt`).

## 3.6 Area text

1. Each of the four offsets at 144 to 151 is relative to the start of `Z` (byte 0 of sector
   0).
2. An offset of 0, or an offset of `E` or more, means the item is absent. This is not
   reported.
3. Otherwise the item is the text string (specification 2, 2.6.4) at that offset, with the
   containing buffer being all of `Z`. An empty result is absent.

## 3.7 Sub-structures

The sectors after sector 0 hold a chain of sub-structures, each starting at a sector boundary
with an 8-byte signature:

| Signature | Size | Content |
|---|---|---|
| `SACDTTxt` | 1 sector | track text for one text channel (3.8) |
| `SACD_IGL` | 2 sectors | ISRCs and per-track genres (3.7.3) |
| `SACD_ACC` | 32 sectors | access list (not interpreted) |
| `SACDTRL1` | 1 sector | track start LSNs and lengths (3.7.1) |
| `SACDTRL2` | 1 sector | track start timecodes and durations (3.7.2) |

### 3.7.0 Walking the chain

1. Start at position `p = 2048` in `Z`.
2. While `p + 8 <= E`, read the signature at `p`:
   - `SACDTTxt`: if no `SACDTTxt` has been seen yet in this chain, record position `p` as the
     track text. Later ones (other text channels) are ignored. Advance `p` by 2048.
   - `SACD_IGL`: record `p` as the ISRC and genre list, replacing any earlier one. Advance by
     4096.
   - `SACD_ACC`: advance by 65 536 (32 sectors). Its content is ignored.
   - `SACDTRL1`: record `p` as the LSN list, replacing any earlier one. Advance by 2048.
   - `SACDTRL2`: record `p` as the timecode list, replacing any earlier one. Advance by 2048.
   - anything else: stop walking.
3. Walking also stops when `p + 8 > E`. Advancing past the end of `Z` is not an error.
4. If no `SACDTRL1` or no `SACDTRL2` was recorded, **skip the area** with
   `area-track-list-missing`.
5. The order of sub-structures is not fixed. Real discs use the order text, ISRC and genre,
   access list, LSN list, timecode list, but readers MUST accept any order.
6. Each signature is examined only at sector boundaries reached by the walk. A structure
   that the walk does not reach (because an unknown signature stopped it first) does not
   exist.

Because every recorded `SACDTRL1`, `SACDTRL2` and `SACDTTxt` position satisfies
`p + 2048 <= E`, those structures always fit in `Z`. A `SACD_IGL` may not fit (3.7.3).

### 3.7.1 `SACDTRL1`: track LSNs

At the recorded position `t1`:

| Offset from `t1` | Size | Type | Content |
|---|---|---|---|
| 0 | 8 | chars[8] | `SACDTRL1` |
| 8 + 4i | 4 | u32 | start LSN of track i (i = 0 to 254) |
| 1028 + 4i | 4 | u32 | length of track i, in sectors |

Only entries `i < track count` are read. Track start LSNs and lengths are reported as stored;
3.9 checks them.

### 3.7.2 `SACDTRL2`: track timecodes

At the recorded position `t2`:

| Offset from `t2` | Size | Type | Content |
|---|---|---|---|
| 0 | 8 | chars[8] | `SACDTRL2` |
| 8 + 4i | 3 | tc | start of track i |
| 11 + 4i | 1 | u8 | start flags of track i |
| 1028 + 4i | 3 | tc | duration of track i |
| 1031 + 4i | 1 | u8 | duration flags of track i |

1. The start is the frame number (specification 0, 0.5) of the track's first frame within the
   area. The duration is its length in frames.
2. The two flag bytes are reported as stored. Their meaning is not defined by these
   specifications.
3. Starts and durations are not validated here; 3.9 bounds them.

### 3.7.3 `SACD_IGL`: ISRCs and genres

At the recorded position `g`:

| Offset from `g` | Size | Type | Content |
|---|---|---|---|
| 0 | 8 | chars[8] | `SACD_IGL` |
| 8 + 12i | 12 | chars[12] | ISRC of track i, fixed text field (specification 2, 2.6.5) |
| 3068 | 4 | | reserved |
| 3072 + 4i | 4 | genre | genre of track i (specification 2, 2.4) |

1. The ISRC is the fixed text field of the 12 bytes. If it is empty, the track has no ISRC.
   An ISRC is reported as decoded; its syntax (`CCXXXYYNNNNN`) is not validated.
2. If a `SACD_IGL` was recorded and the track count is at least 1, the structure MUST lie
   entirely within `Z`: if `g + 4096 > E`, **skip the area** with `area-toc-malformed`.
3. If no `SACD_IGL` was recorded, tracks have no ISRC and no genre (absent, not 0).

## 3.8 Track text (`SACDTTxt`)

At the recorded position `b` of the first `SACDTTxt`:

| Offset from `b` | Size | Type | Content |
|---|---|---|---|
| 0 | 8 | chars[8] | `SACDTTxt` |
| 8 + 2i | 2 | u16 | offset of track i's text, relative to `b`; 0 = no text |

At `q = b + offset`, a track's text is a list of items:

| Offset from `q` | Size | Type | Content |
|---|---|---|---|
| 0 | 1 | u8 | item count |
| 1 | 3 | | reserved |
| 4 | | | first item |

Each item is:

| Offset from item start | Size | Type | Content |
|---|---|---|---|
| 0 | 1 | u8 | item type (table below) |
| 1 | 1 | | reserved (0x20 on real discs) |
| 2 | | | text string, ending with 0x00 |

Items are separated by zero or more 0x00 bytes of padding. The next item starts at the first
non-zero byte after the string's terminating 0x00.

| Type | Item | Type | Item |
|---|---|---|---|
| 0x01 | title | 0x81 | title, phonetic |
| 0x02 | performer | 0x82 | performer, phonetic |
| 0x03 | songwriter | 0x83 | songwriter, phonetic |
| 0x04 | composer | 0x84 | composer, phonetic |
| 0x05 | arranger | 0x85 | arranger, phonetic |
| 0x06 | message | 0x86 | message, phonetic |
| 0x07 | extra message | 0x87 | extra message, phonetic |

### 3.8.1 Parsing track text

For each track i (0 to track count - 1), all positions being in `Z`:

1. Read the u16 offset at `b + 8 + 2i`. If it is 0 the track has no text.
2. Let `q = b + offset`. If `q + 4 > E`, **warn** `track-text-offset-invalid` for that track;
   the track has no text.
3. Let `k` be the byte at `q` (the item count). Set `q = q + 4`.
4. Repeat for item j = 0 to k - 1:
   1. If `q + 2 > E`, stop: the remaining items do not exist.
   2. The item type is the byte at `q`. Set `q = q + 2`.
   3. Take the text string's bytes at `q` (specification 2, 2.6.4 item 1, with the whole of
      `Z` as the containing buffer). Call their count `r` (0 to 4096).
   4. If `r > 0`:
      - if the type is in the table, decode the string (2.6.4) and store it as that item of
        the track, replacing any earlier item of the same type for the track;
      - otherwise **warn** `track-text-type-unknown` for the track and ignore the item.
      If `r = 0` the item is ignored silently, whatever its type.
   5. If this is not the last item (`j < k - 1`), set `q` to the first position at or after
      `q + r` whose byte is not 0x00, or to `E` if there is none.
5. The track's text is the set of stored items. An item whose decoded string is empty is
   absent (2.6.4 item 4).

Notes:

1. Step 4.5 starts the search at `q + r`, the position of the string's terminator. When a
   string was cut at 4096 bytes there is no terminator there, so the search starts inside the
   string and the next "item" begins with a byte of that string. This is the normative result:
   it keeps the work for one track's items bounded by about `E` bytes for any input.
2. Strings are bounded by the end of `Z`, not by the end of the `SACDTTxt` sector: a string
   that is not terminated within the sector continues into the following sectors.
3. Text offsets may point anywhere in `Z`, including outside the `SACDTTxt` sector.

## 3.9 Track list and area checks

### 3.9.1 Tracks

1. The area has exactly `track count` tracks (offset 69), numbered 1 to `track count` in the
   order of their entries. Track number `i + 1` uses entry `i` of every list. The track offset
   (offset 68) does not change this numbering.
2. A track count of 0 is valid: the area has no tracks.
3. Each track reports: number, start LSN, length in sectors, start (frames), duration
   (frames), start flags, duration flags, ISRC (or none), genre (or none) and text items.

### 3.9.2 Checks after parsing

After 3.3 to 3.8 have succeeded, with `P` the frame number of the total area play time
(offset 64) and `[S0, S1]` the audio range (offsets 72 and 76):

1. If `S1 < S0`, **skip the area** with `area-audio-range-reversed`.
2. For each track with start `s` and duration `d` (frame numbers), let `e = s + d`. If
   `e > P`:
   - **clamp**: the reported start becomes `s' = min(s, P)` and the reported duration becomes
     `d' = min(d, P - s')`;
   - **warn** `track-past-playtime` for that track.
   Otherwise the start and duration are unchanged. After this step every track satisfies
   `start + duration <= P`.
3. For each track, if its start LSN is not in `[S0, S1]`, **warn** `track-lsn-outside-area`
   for that track. The LSN is still reported unchanged.
4. No other relation is checked. Tracks may overlap, be out of order, have zero duration, or
   leave gaps; the result of each is defined in specification 4.

### 3.9.3 Fields outside the buffer

Every position used in 3.3 to 3.8 has been bounded above; the only structure that can extend
past `E` is a `SACD_IGL`, handled by 3.7.3 item 2. A reader MUST NOT read outside `Z` while
parsing an area TOC.

## 3.10 Limits

| Item | Range | Out-of-range behaviour |
|---|---|---|
| Area TOC size | 1 to 128 sectors | 0: slot unused; above 128: skip the area |
| Track count | 0 to 255 | (all values valid) |
| Area locales reported | at most 10 | count capped |
| Timecode frame number | 0 to 1 166 880 | (all values valid) |
| Track start + duration | at most the play time | clamped, with a warning |
| Text string | at most 4096 bytes, within `Z` | cut |
| Track text items per track | 0 to 255 | items past `E` do not exist |
| Sub-structure walk | at most `E / 2048` steps | stops at an unknown signature or at `E` |
