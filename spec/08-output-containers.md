# 8. Output containers

This specification defines how a reader writes the audio of a track, a frame range or a track
with pause (specification 4, 4.7 and 4.7.1) as a DSF file, optionally with an ID3v2 tag, or
as a DSDIFF file holding either plain DSD or DST frames, optionally as an edited master with
markers and text. 8.7 lists, without allowing them, departures from [DSDIFF] that some
existing programs make. A reader MAY write any of these files. A reader that does MUST follow this
specification. Conventions are in [00-conventions.md](00-conventions.md).

It restates, in this repository's own terms, the parts of four published documents that a
writer needs. Where those documents leave more than one valid output, this specification says
**writer's choice** and lists the point in 8.6 instead of choosing. Where a document is silent
or contradicts another one, the text says so.

| Ref | Document | Edition | Source, accessed 2026-10-31 |
|---|---|---|---|
| [DSF] | Sony Corporation, *DSF File Format Specification* | version 1.01 (change history dated 11 November 2005) | https://dsd-guide.com/sites/default/files/white-papers/DSFFileFormatSpec_E.pdf |
| [DSDIFF] | Philips, *Direct Stream Digital Interchange File Format, DSDIFF* | version 1.5, 2004-04-27 | https://dsd-guide.com/sites/default/files/white-papers/DSDIFF_1.5_Spec.pdf |
| [ID3] | M. Nilsson, *ID3 tag version 2.4.0 - Main Structure* and *ID3 tag version 2.4.0 - Native Frames* | 1 November 2000 | https://raw.githubusercontent.com/id3/ID3v2.4/master/id3v2.40-structure.txt and id3v2.4.0-frames.txt in the same directory |
| [ID3v2.3] | M. Nilsson, *ID3 tag version 2.3.0* (informal standard) | 3 February 1999 | https://id3.org/id3v2.3.0 (accessed 2026-11-01) |

## 8.1 What is written

1. A **writer** is a reader that writes one of the files of this specification.
2. A file holds the positions of one range `[a, b)` of one area, with `N = b - a` positions:
   a track (4.7 item 1), a frame range (4.7.1 item 2) or a track with pause (4.7.1 item 5).
   The checks of specification 4, 4.8 (or 4.7.1 item 4) apply first. A range that they skip
   or refuse is not written, and the writer reports the same code. So `N >= 1` and the
   area's channel count `C` is 1 to 6 in every file. A DSF file MAY instead hold a byte
   range (8.2.7), which is built from ranges of positions and checked the same way.
3. The audio of a position is its content by specification 4, 4.6: `C x 4704` bytes in the
   interleaved layout of 0.6 item 2, with the fills of 0.6 item 3 for missing and invalid
   positions. DSF and DSDIFF with plain DSD hold this content. DSDIFF with DST holds the DST
   equivalents of 4.6.1 item 3 instead, which decode to the same content (4.6.1 item 4).
4. The sample rate written is 2 822 400 Hz whatever the area's stored sample frequency code,
   because the audio is DSD64 in every case (specification 3, 3.4.2 item 3).
5. The **layout** of an area is that of specification 3, 3.4.1 item 2: stereo for channel
   count 2 with configuration 0, 3/2 for 5 with 3, and 5.1 for 6 with 4. Every other
   combination is **unspecified**. 8.2.4 and 8.4.4 map layouts to each format's values.
6. Text written into a file comes from the strings a reader reports (specification 0, 0.7):
   sequences of Unicode scalar values. Those come from the first text channel only
   (specification 2, 2.5.3 for disc text; specification 3, 3.5 for area and track text).
7. The **source track** of a file is the track it holds when it holds a track or a track with
   pause. A file holding a frame range has no source track, unless the writer names one whose
   frames `[s_i, s_i + d_i)` overlap `[a, b)` (8.6).
   The per-track items of 8.3.5 and the one-track markers of 8.4.10 item 3 come from the
   source track; a file without one has neither.
8. Besides the types of 0.2 item 4, tables here use `u64`, an unsigned 8-byte integer, and
   `s32`, a signed 4-byte two's complement integer. Byte order is stated per format: least
   significant byte first in DSF (8.2.1), most significant byte first in ID3 (8.3.1) and in
   DSDIFF (8.4.1).

## 8.2 DSF

### 8.2.1 File structure

A DSF file is four regions in this order, with nothing between them: the `DSD ` chunk, the
`fmt ` chunk, the `data` chunk, and an optional metadata region at the end. Every integer is
unsigned and stored least significant byte first. Each signature is 4 ASCII bytes and ends in
a space where shown.

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 4 | signature | `DSD ` |
| 4 | 8 | size of the `DSD ` chunk | 28 |
| 12 | 8 | total file size | `92 + n + m` (8.2.5) |
| 20 | 8 | metadata pointer | `92 + n` if the file has a tag, 0 if not |
| 28 | 4 | signature | `fmt ` |
| 32 | 8 | size of the `fmt ` chunk | 52 |
| 40 | 4 | format version | 1 |
| 44 | 4 | format ID | 0 (DSD raw) |
| 48 | 4 | channel type | 8.2.4 |
| 52 | 4 | channel count | `C` |
| 56 | 4 | sampling frequency | 2 822 400 |
| 60 | 4 | bits per sample | 1 or 8 (8.2.3) |
| 64 | 8 | sample count | `8 x S` (8.2.2) |
| 72 | 4 | block size per channel | 4096 |
| 76 | 4 | reserved | 0 |
| 80 | 4 | signature | `data` |
| 84 | 8 | size of the `data` chunk | `n + 12` |
| 92 | `n` | sound data | 8.2.2 |
| `92 + n` | `m` | metadata | an ID3v2 tag (8.2.6), or nothing (`m = 0`) |

1. Each chunk size counts the whole chunk, including its 12-byte signature and size. This is
   unlike DSDIFF (8.4.1).
2. [DSF] describes the `fmt ` chunk's size as "usually 52 bytes". A writer writes 52; the
   table above has no other field.
3. The sample count counts one channel's samples, which are bits: `S` DSD bytes of 8
   samples each (`N x 37 632` samples for a range of positions). The padding of 8.2.2 is not
   counted in it.

### 8.2.2 Sound data

Let `S = N x 4704` be the number of DSD bytes per channel. Channel `c`'s **stream** is its DSD
bytes in time order: byte `k` (0 to `S - 1`) of the stream is byte `(k mod 4704) x C + c` of
the content of position `a + floor(k / 4704)`. For a byte range, 8.2.7 item 2 gives `S` and
the stream instead; everything below is the same.

1. Each stream is cut into blocks of 4096 bytes. There are `B = ceil(S / 4096)` blocks per
   channel. Block `j` holds stream bytes `4096 j` to `min(4096 j + 4095, S - 1)`, followed,
   in the last block only when `S` is not a multiple of 4096, by bytes of 0x00 up to 4096. The pad bytes are
   0x00 whatever the bits per sample, and they are not samples.
2. The sound data is block 0 of channel 0, block 0 of channel 1, ..., block 0 of channel
   `C - 1`, then block 1 of each channel in the same order, and so on to block `B - 1`. Its
   size is `n = 4096 x B x C`.
3. Each stream byte is written as 8.2.3 says. Pad bytes are written unchanged.

[DSF] gives the block size per channel (4096) and requires that the unused part of a block be
filled with 0x00. That each channel's blocks alternate, one block of each channel in channel
order, is how this specification reads [DSF] together with its channel-order annotation; [DSF]
has no figure of it.

### 8.2.3 Bit order

[DSF] allows two values of bits per sample, and the bit order of a stored byte depends on it.

| Bits per sample | Stored byte | Earliest sample in |
|---|---|---|
| 1 | the DSD byte with its bits reversed: bit `i` of the DSD byte is bit `7 - i` of the stored byte | bit 0 |
| 8 | the DSD byte unchanged | bit 7 (as in 0.6 item 1) |

Which value to write is the writer's choice (8.6). [DSF] gives the reversal as an example:
DSD bytes 0x00, 0x01, 0x02, 0x03, 0x04 are stored as 0x00, 0x80, 0x40, 0xC0, 0x20 when bits per
sample is 1.

### 8.2.4 Channel type

| Layout (8.1 item 5) | Channel type | Channel order in [DSF] |
|---|---|---|
| stereo | 2 | front left, front right |
| 3/2 | 6 | front left, front right, centre, back left, back right |
| 5.1 | 7 | front left, front right, centre, low frequency, back left, back right |

1. In each case the order is that of specification 3, 3.4.1, so channel `c` of the content
   is channel `c` of the file. [DSF] names the last two channels "back" where 3.4.1 says
   "surround"; the channels are the same.
2. [DSF] defines no value for an unspecified layout. For one, the writer chooses a channel
   type whose channel count is `C`, or refuses to write the file with an error code of its
   own (8.6). The types [DSF]
   defines, by count, are:

   | `C` | Types with that count |
   |---|---|
   | 1 | 1 (mono) |
   | 2 | 2 (stereo: front left, front right) |
   | 3 | 3 (front left, front right, centre) |
   | 4 | 4 (quad: front left, front right, back left, back right) and 5 (front left, front right, centre, low frequency) |
   | 5 | 6 |
   | 6 | 7 |

   The channel type and the channel count MUST agree.

### 8.2.5 Sizes

Every field of the file follows from `C`, `S` and the tag's size `m`, where `S = N x 4704`
for a range of positions and `S = q - p` for a byte range (8.2.7):

- `B = ceil(S / 4096)` and `n = 4096 x B x C`;
- sample count `= 8 x S`;
- `data` chunk size `= n + 12`;
- metadata pointer `= 92 + n` when `m > 0`, and 0 when `m = 0`;
- total file size `= 92 + n + m`.

A writer can therefore state the file's exact size, and write its header, before it reads any
audio, once it has built the tag.

### 8.2.6 Metadata

1. The metadata region is optional (writer's choice, 8.6). When present, it is one ID3v2 tag
   that starts right after the sound data and ends the file. [DSF] says only "ID3v2", so
   either of these is valid, as the writer chooses (8.6):
   - an ID3v2.4 tag that the writer builds by 8.3; or
   - an ID3v2.3 tag ([ID3v2.3]) whose bytes the caller supplies (item 4).
2. When there is no tag, the metadata pointer is 0 and the file ends with the sound data.
3. An ID3v2.4 tag has at least one frame ([ID3] section 4). A writer with nothing to tag writes
   no tag.
4. **A caller-supplied ID3v2.3 tag** is a sequence of `m` bytes `T[0]` to `T[m - 1]`. The
   writer writes it unchanged as the metadata region if all of the following hold, and
   otherwise refuses to write the file, with an error code of its own:
   - `m >= 10`, and `T[0..3]` is `ID3` (0x49 0x44 0x33);
   - `T[3] = 0x03` and `T[4] = 0x00` (version 2.3.0);
   - the low five bits of the flags byte `T[5]` are 0 ([ID3v2.3] section 3.1 defines only
     bits 7, 6 and 5);
   - `T[6]` to `T[9]` are each less than 0x80, and the 28-bit size they hold,
     `T[6] x 2^21 + T[7] x 2^14 + T[8] x 2^7 + T[9]`, equals `m - 10`. [ID3v2.3] has no
     footer, so the size covers everything after the 10-byte header.

   This specification checks only the header. It does not define, check or change the
   frames, and builds no ID3v2.3 tag of its own.

### 8.2.7 Byte ranges

A DSF file MAY hold a **byte range** of each channel instead of whole positions, as the writer
chooses (8.6). This is DSF only. It allows "gapless" files whose boundaries fall inside a
position, so that each file except the last holds whole 4096-byte blocks.

1. The **byte timeline** numbers one channel's DSD bytes across the area: byte `k` of channel
   `c` is byte `(k mod 4704) x C + c` of the content (4.6) of position `floor(k / 4704)`.
   Frame `f` starts at byte `4704 f`.
2. A file holding the byte range `[p, q)`, with `p < q`, has `S = q - p`, and byte `k` (0 to
   `S - 1`) of channel `c`'s stream is byte `p + k` of channel `c`'s byte timeline. 8.2.2 to
   8.2.5 then apply unchanged.
3. Byte ranges are made from a **chain**: ranges of positions `[a_1, b_1)`, ...,
   `[a_K, b_K)` with `K >= 1` and `b_i = a_{i+1}` for `i < K`, for example the tracks with
   pause of consecutive tracks (4.7.1 item 5) when those meet end to end. Each `[a_i, b_i)`
   is checked by 8.1 item 2. A writer that skips or refuses one splits the chain there: the
   ranges before it and the ranges after it form separate chains. File `i` of the chain holds
   the byte range `[p_i, p_{i+1})`, where:
   - `p_1 = 4704 a_1`;
   - `p_i = p_1 + 4096 x floor(4704 (a_i - a_1) / 4096)` for `2 <= i <= K`;
   - `p_{K+1} = 4704 b_K`.
4. So file `i`, for `i >= 2`, starts with the last `4704 a_i - p_i` bytes (0 to 4095) of
   each channel of the range before it, and every file except the last ends at a block
   boundary relative to `p_1`, leaving the rest of its range to the next file. Its stream is a
   whole number of blocks with no pad: `S` is a multiple of 4096 and at least 4096, because
   each range is at least one position (4704 bytes) long. The last file ends at `4704 b_K`,
   with the pad of 8.2.2 when its `S` is not a multiple of 4096. Concatenating the files'
   streams of a channel, in order, gives bytes `4704 a_1` to `4704 b_K - 1` of its byte
   timeline, unchanged and with no pad between files.
5. Every size of every file in the chain follows from `C`, the frame numbers `a_1` and the
   `b_i`, and the tags (8.2.5), with no audio read.
6. The source track (8.1 item 7) of file `i` is that of the range `[a_i, b_i)`, although the
   file also holds bytes of its neighbours.

## 8.3 ID3v2.4 as DSF carries it

### 8.3.1 Tag structure

A tag is a 10-byte header, the frames, then either padding or a 10-byte footer, or neither.
Integers in a tag are stored most significant byte first. A **synchsafe integer** of 4 bytes
stores a value below 2^28 in 7 bits per byte, most significant byte first, with bit 7 of each
byte 0.

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 3 | identifier | `ID3` |
| 3 | 1 | major version | 4 |
| 4 | 1 | revision | 0 |
| 5 | 1 | flags | 8.3.2 |
| 6 | 4 | tag size, synchsafe | the size of the frames and padding, excluding the header and the footer |

The footer, when present, is the header with identifier `3DI` instead of `ID3`, and is the
last 10 bytes of the tag.

1. **Padding** is optional. It is bytes of 0x00 after the last frame, counted in the tag size.
   There is no padding between frames or before the first frame. A tag with a footer has no
   padding.
2. **Footer.** [ID3] section 3.4 requires a footer on a tag placed after all audio data, which
   a DSF tag is. [DSF] does not mention a footer. Whether to write one is listed in 8.6
   because the two documents differ. A writer that follows [ID3] writes the footer and so no
   padding.
3. The tag's total size `m` is `10 +` tag size, plus 10 with a footer.

### 8.3.2 Header flags

| Bit | Meaning | Value written |
|---|---|---|
| 7 | unsynchronisation applied | 0 |
| 6 | extended header present | 0 |
| 5 | experimental | 0 |
| 4 | footer present | 1 with a footer, else 0 |
| 3 to 0 | | 0 |

Unsynchronisation and the extended header are optional in [ID3]. A writer of this
specification uses neither, because nothing in a DSF file needs them; [ID3] says
unsynchronisation is useful only for MPEG and AAC files.

### 8.3.3 Frames

Each frame is a 10-byte header followed by its body:

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 4 | frame ID | 4 characters from `A` to `Z` and `0` to `9` (8.3.5) |
| 4 | 4 | body size, synchsafe | the body's size, excluding this header; at least 1 |
| 8 | 2 | flags | 0x00 0x00 |

1. With both flag bytes 0, no frame is compressed, encrypted, unsynchronised or grouped, and
   no field precedes the body.
2. The order of frames is the writer's choice (8.6). [ID3] suggests the most identifying
   frames first.
3. A tag has at most one text frame (ID starting with `T`, other than `TXXX`) of each ID, and
   at most one `COMM` frame with a given language and description.

### 8.3.4 Text in frames

A text frame's body is one **encoding byte** and then one or more strings in that encoding.
Several strings are separated by the encoding's terminator.

| Encoding byte | Encoding | Terminator |
|---|---|---|
| 0x00 | ISO 8859-1 | 0x00 |
| 0x01 | UTF-16 with a byte order mark before each string, including an empty one; all strings of a frame in the same byte order | 0x00 0x00 |
| 0x02 | UTF-16BE without a byte order mark | 0x00 0x00 |
| 0x03 | UTF-8 | 0x00 |

1. **Control characters** are the code points U+0000 to U+001F and U+007F to U+009F. A string
   holds none, except U+000A (a line break) in the comment text of `COMM` (8.3.6). A reported
   string that holds others is written with them removed or replaced, as the writer chooses
   (8.6).
2. The encoding is the writer's choice per frame (8.6), with one limit: 0x00 MAY be used only
   when every code point of the frame's strings, after item 1, is U+000A or U+0020 to U+00FF.
   ISO 8859-1 then maps each code point to the byte of the same value.
3. [ID3] does not say whether the last string of a text frame, the value of `TXXX`, or the
   text of `COMM` is followed by a terminator. Writing it or not is the writer's choice (8.6).
4. Several values (for example several performers) are several strings of one frame. The TOC
   gives each item as one string (specification 3, 3.8). Splitting it into several values is
   the writer's choice (8.6); without splitting, the frame has one string.
5. A numeric string holds only the characters `0` to `9`.

### 8.3.5 Frames for TOC items

A writer writes any subset of the frames below. Each frame is written only when its item is
present (not absent) in what the reader reports. "Choice" marks a mapping [ID3] does not fix
(8.6).

| Item | Source | Frame | Body |
|---|---|---|---|
| track title | 3.8, type 0x01 | `TIT2` | the string |
| track performer | 3.8, 0x02 | `TPE1` | the string |
| songwriter | 3.8, 0x03 | `TEXT` | the string. [ID3] defines `TEXT` as the lyricist or text writer, the nearest frame |
| composer | 3.8, 0x04 | `TCOM` | the string |
| arranger | 3.8, 0x05 | `TIPL` | two strings, a function and a name: `arranger`, then the string. Choice: `TIPL` or `TXXX` (8.3.6) |
| message, extra message | 3.8, 0x06 and 0x07 | `COMM` | 8.3.6. Choice: one frame each, with different descriptions, or both in one frame joined by a separator of the writer's choice |
| phonetic title, phonetic performer | 3.8, 0x81 and 0x82 | `TSOT`, `TSOP` | the string. Choice: these frames or none |
| track number and count | 3.9.1 | `TRCK` | numeric string of the source track's number, `/`, numeric string of the area's track count, for example `4/9`. Choice: with or without `/count` |
| ISRC | 3.7.3 | `TSRC` | the 12-character ISRC |
| track genre | 3.7.3 | `TCON` | 8.3.7 |
| album or disc title | 2.5.4, items 0 and 8 | `TALB` | Choice: the disc title, the album title, or the disc title falling back to the album title (2.5.4, informative note) |
| album or disc artist | 2.5.4, items 1 and 9 | none | [ID3] defines no album-artist frame. Choice: no frame, `TPE2` (which [ID3] defines as band, orchestra or accompaniment), or `TXXX` (8.3.6) |
| publisher | 2.5.4, items 2 and 10 | `TPUB` | the string. Choice: album or disc publisher, as for `TALB` |
| copyright | 2.5.4, items 3 and 11 | `TCOP` | [ID3] requires the string to begin with a 4-digit year and a space. Choice: album or disc copyright, as for `TALB`; and no frame, or the year of 2.3.2 (only when 1 to 9999, as 4 digits), a space and the string |
| album set | 2.3.1 | `TPOS` | when the sequence number `q` and set size `z` satisfy `1 <= q <= z`: numeric string `q`, `/`, numeric string `z`, for example `1/2`. Otherwise, choice: no frame, or `q` alone when `q >= 1` |
| date | 2.3.2 | `TDRC` or `TDRL` | 8.3.8. Choice of frame: [ID3] defines `TDRC` as the recording time and `TDRL` as the release time |
| album genres, disc genres | 2.3, offsets 40 and 104 (entries as in 2.4) | `TCON` | 8.3.7. Choice: which of the track genre, the album genres and the disc genres to use; they are taken in that order |

`TXXX` (user defined text) and the description strings it needs are in 8.3.6.

### 8.3.6 `COMM` and `TXXX`

| Field | Size | Value |
|---|---|---|
| encoding | 1 | 8.3.4 |
| language | 3 | three ISO 639-2 letters in lower case, or `XXX` if unknown |
| description | variable | a string and its terminator; may be empty |
| text | variable | the comment; line breaks allowed |

The language is the writer's choice. The TOC gives two-letter ISO 639-1 codes: for track
text, area locale entry 0 (specification 3, 3.5, with the entry format of specification 2,
2.5.1); for disc text, master TOC locale entry 0 (2.5.1). The writer may map the code to
ISO 639-2 or write `XXX`.

A `TXXX` body is an encoding byte (8.3.4), a description string and its terminator, then the
value string. A tag has at most one `TXXX` with a given description. The description
strings, for example for the arranger or the album artist, are the writer's choice (8.6).

### 8.3.7 `TCON`

1. For an entry of genre table 1 (specification 2, 2.4) with index 2 to 29, the string is the
   name in the table of 2.4.
2. Entries with genre table 0, index 0 or index 1, entries of table 2, and entries outside the
   tables give no string.
3. [ID3] also allows ID3v1 genre numbers as numeric strings. Those numbers are a different
   list from the SACD indexes and are not used.
4. Several genres are several strings of one frame (8.3.4 item 4), in the order of the
   entries: the track genre, then the album genres, then the disc genres, as far as the
   writer uses them (8.3.5). Each name appears once, at its first place.
5. When no entry gives a string, no `TCON` frame is written.

### 8.3.8 Date

[ID3] timestamps are `yyyy`, `yyyy-MM` or `yyyy-MM-dd`, with the year as 4 digits and month
and day as 2. From the year, month and day of specification 2, 2.3.2:

1. If the year is 0, or above 9999 (2.3.2 stores it as a u16, so it can be up to 65 535), no
   date frame is written.
2. Otherwise, if the month is 1 to 12 and the day is a day of that month in that year of the
   Gregorian calendar (February has 29 days when the year is divisible by 4 and not by 100,
   or by 400), the timestamp is `yyyy-MM-dd`.
3. Otherwise, if the month is 1 to 12, it is `yyyy-MM`.
4. Otherwise it is `yyyy`.

[ID3] says timestamps are in UTC; the TOC date has no time zone.

## 8.4 DSDIFF

### 8.4.1 Chunks

1. A DSDIFF file is a sequence of **chunks**. A chunk is a 4-byte ID, an 8-byte data size `D`,
   then `D` bytes of data, then one **pad byte** 0x00 if `D` is odd. The pad byte is not
   counted in `D`. A chunk therefore occupies `12 + D + (D mod 2)` bytes.
2. Every integer is stored most significant byte first. Integers are unsigned, except the
   marker offset of 8.4.7, which is a signed 32-bit two's complement number.
3. An ID is 4 bytes from 0x20 to 0x7E. No space comes before a non-space, so a short name is
   padded with spaces at the end: `DSD `, `C   `.
4. **Text bytes** are bytes from 0x20 to 0x7E. All text in DSDIFF is text bytes (8.4.9).
5. [DSDIFF] tells readers to skip chunks they do not recognise. A writer of this specification
   writes only the chunks below and the caller chunks of 8.4.2 item 5.

### 8.4.2 File structure

The file is one `FRM8` chunk whose data is the ID `DSD ` followed by the **local chunks**:

| Local chunk | ID | Required | Position |
|---|---|---|---|
| format version | `FVER` | yes | first |
| property | `PROP` | yes | before the sound data chunk |
| sound data, plain DSD | `DSD ` | one of these two | after `PROP` |
| sound data, DST | `DST ` | | after `PROP` |
| DST sound index | `DSTI` | no; only with `DST ` | 8.4.6 |
| comments | `COMT` | no | not constrained |
| edited master information | `DIIN` | no | not constrained |

1. Each local chunk of the table appears at most once.
2. The positions of `COMT`, `DIIN` and `DSTI` among the local chunks after `FVER` are the
   writer's choice (8.6).
3. [DSDIFF] also defines a manufacturer specific chunk (`MANF`), which a writer of this
   specification does not write.
4. The `FRM8` data size is 4 plus the space all local chunks occupy (8.4.1 item 1), caller
   chunks included. It equals the file size minus 12 and is always even.
5. **Caller chunks.** After the last of the local chunks above, a writer MAY write any number
   of **caller chunks** (8.6): chunks whose ID and data bytes its caller supplies, in the
   caller's order. Each is written by 8.4.1 item 1: `D` is the number of data bytes, and a
   pad byte 0x00 follows when `D` is odd, not counted in `D`. Its ID meets 8.4.1 item 3 and is
   none of `FRM8`, `FVER`, `PROP`, `DSD `, `DST `, `DSTI`, `COMT`, `DIIN` and `MANF`. The same
   ID may appear more than once. [DSDIFF] defines no such chunks and tells readers to skip
   them (8.4.1 item 5); this specification does not define or check their data. For example,
   a caller may supply an `ID3 ` chunk holding an ID3 tag, one in a track file, or one per
   track in an edited master. Caller chunks are allowed in an edited master (8.4.11).

`FVER` has `D = 4` and the value 0x01050000 (version 1.5.0.0).

### 8.4.3 Property chunk

`PROP` data is the ID `SND ` followed by these chunks, in an order that is the writer's choice
(8.6):

| Chunk | ID | `D` | Data | Required |
|---|---|---|---|---|
| sample rate | `FS  ` | 4 | u32 2 822 400 | yes |
| channels | `CHNL` | `2 + 4C` | u16 `C`, then `C` channel IDs (8.4.4) | yes |
| compression type | `CMPR` | `5 + L` | the compression ID, a u8 `L`, then `L` text bytes of name | yes |
| absolute start time | `ABSS` | 8 | u16 hours, u8 minutes, u8 seconds, u32 samples | no (8.4.8) |
| loudspeaker configuration | `LSCO` | 2 | u16 configuration (8.4.4) | no |

| Sound data | Compression ID | Name | `L` | `CMPR` `D` |
|---|---|---|---|---|
| plain DSD | `DSD ` | `not compressed` | 14 | 19, so one pad byte follows |
| DST | `DST ` | `DST Encoded` | 11 | 16 |

The `PROP` data size is 4 plus the space its chunks occupy.

### 8.4.4 Channel IDs and loudspeaker configuration

| Layout (8.1 item 5) | Channel IDs, in order | `LSCO` value |
|---|---|---|
| stereo | `SLFT` `SRGT` | 0 |
| 3/2 | `MLFT` `MRGT` `C   ` `LS  ` `RS  ` | 3 |
| 5.1 | `MLFT` `MRGT` `C   ` `LFE ` `LS  ` `RS  ` | 4 |

1. The channel IDs give the order of the channels in the sound data. For these layouts, the
   order above is the order [DSDIFF] requires and also that of specification 3, 3.4.1, so
   channel `c` of the content is channel `c` of the file.
2. For an unspecified layout, [DSDIFF] offers `C000` to `C999` ("context specific channel
   contents") and the configuration value 65 535 ("undefined channel set-up"). The writer
   chooses `C` distinct channel IDs from `C000` to `C999` (for example `C000` to `C00`*k*),
   and whether to write `LSCO` with 65 535 or no `LSCO` at all (8.6).
3. The configuration values are those of specification 3, 3.4.1 for the same layouts.

### 8.4.5 Plain DSD sound data

The `DSD ` chunk's data is the content of positions `a` to `b - 1`, in that order, unchanged.
[DSDIFF] stores a channel's 8 samples in a byte with the earliest sample in the most
significant bit, and interleaves channel bytes in channel order. That is the layout of 0.6.

- `D = N x C x 4704`, which is even, so no pad byte is needed.

### 8.4.6 DST sound data

The `DST ` chunk's data is the following chunks only, with no ID before them; its `D` is the
space they occupy (8.4.1 item 1):

1. one `FRTE` chunk (frame information), first: `D = 6`, a u32 frame count `N`, and a u16
   frame rate of 75;
2. then, for each position from `a` to `b - 1`, in order, one `DSTF` chunk whose data is that
   position's DST equivalent (specification 4, 4.6.1 item 3): `D = L_t`, the equivalent's
   length, with a pad byte when `L_t` is odd;
3. optionally, right after each `DSTF`, one `DSTC` chunk (8.4.6.1). A file has a `DSTC` after
   every `DSTF` or after none.

For an invalid DST frame, the writer writes either its stored bytes or, as specification 4,
4.6.1 item 5 allows, the byte 0x00 followed by `C x 4704` bytes of 0x55. This is the writer's
choice (8.6). Both decode by specifications 4 and 5 to the same content.

**Decoding.** By specification 4, 4.6.1 item 4, decoding the `DSTF` frames in order with
`Nch = C` gives exactly the range's audio. The container adds only the framing above.

**Sizes.** With `L_t` known for every position, every size follows: `FRTE` occupies 18 bytes,
each `DSTF` occupies `12 + L_t + (L_t mod 2)`, each `DSTC` 16, and the `DST ` data size is their
sum. `L_t` is the stored DST frame's length (from the stored frame list of 4.6.1 item 2, which
the writer reads without decoding), or `1 + C x 4704` for plain and missing positions and for
an invalid frame replaced as above. So a writer can compute the whole file's layout, and the
index of 8.4.6.2, from the lengths alone, with no audio bytes and no decoding. There are two
exceptions: the values of the `DSTC` chunks need decoding, and so does knowing which DST
frames are invalid when the writer replaces them.

#### 8.4.6.1 Frame CRC

A `DSTC` chunk has `D = 4` and holds a CRC of the position's content (4.6: the decoded
`C x 4704` bytes, interleaved), stored most significant byte first.

1. Treat the content as a sequence of `C x 4704 x 8` bits, starting with the most significant
   bit of its first byte.
2. Append 32 zero bits.
3. Divide by the polynomial `x^32 + x^31 + x^4 + 1` in modulo-2 arithmetic. The CRC is the
   32-bit remainder.

There is no initial value, no final exclusive-or and no bit reflection. Writing `DSTC` chunks
is the writer's choice (8.6). It requires decoding every position.

Test vectors (hexadecimal, most significant byte first):

| Content | CRC |
|---|---|
| `C = 1`, 4704 bytes of 0x69 | 2159B5D9 |
| `C = 2`, 9408 bytes of 0x69 | 2B3AB9F8 |
| `C = 2`, 9408 bytes of 0x55 | 380CA058 |

#### 8.4.6.2 Sound index

A `DSTI` chunk holds one 12-byte entry per `DSTF`, in order: a u64 offset and a u32 length. So
`D = 12 N`, which is even.

1. `DSTI` is optional. [DSDIFF] recommends it and allows it only with DST sound data. Writing
   it, and where among the local chunks (8.4.2 item 2), are the writer's choices (8.6).
2. The length is `L_t`, without the pad byte.
3. [DSDIFF] is ambiguous about the offset. Its overview calls the entries "file positions of
   the DST Frame Data Chunks", while the field's own description says "offset in the file [in
   bytes] of the sound in the DST Sound Data Chunk". The offset is therefore either the file
   position of the `DSTF` chunk's first header byte, or that position plus 12, where the
   frame's bytes begin. Both are measured from the start of the file. The writer chooses one
   (8.6).

### 8.4.7 Markers

A marker chunk (`MARK`, inside `DIIN`, 8.4.8) has this data:

| Offset | Size | Type | Field |
|---|---|---|---|
| 0 | 2 | u16 | hours, 0 to 23 |
| 2 | 1 | u8 | minutes, 0 to 59 |
| 3 | 1 | u8 | seconds, 0 to 59 |
| 4 | 4 | u32 | samples, 0 to 2 822 399 |
| 8 | 4 | s32 | offset, in samples |
| 12 | 2 | u16 | marker type |
| 14 | 2 | u16 | marker channel |
| 16 | 2 | u16 | track flags |
| 18 | 4 | u32 | text length `k` |
| 22 | `k` | text bytes | marker text, then 0x00 if `k` is odd |

1. **Time.** A marker's position is its time plus its offset. Times are absolute, on the
   same axis as the absolute start time (8.4.8), and lie at frame boundaries. This
   specification writes offset 0. The time of a frame number `f` is `hours = floor(f / 270
   000)`, `minutes = floor(f / 4500) mod 60`, `seconds = floor(f / 75) mod 60` and `samples =
   (f mod 75) x 37 632`.
2. **Types:** 0 track start, 1 track stop, 2 program start, 4 index. 3 is obsolete.
3. **Channel:** 0 means all channels. This specification writes 0.
4. **Track flags** are meaningful only on a track start with channel 0, and are 0 otherwise.
   The TOC fields of specifications 2 and 3 do not define these flags, so this specification
   writes 0. A writer with a mapping of its own may set them (8.6).
5. **Text:** the writer's choice, for example empty or the track title (8.4.9).
6. [DSDIFF] does not say whether `D` counts the pad byte after the marker text. The bytes in
   the file are the same either way; only `D` differs. The writer chooses (8.6). The same
   holds for `DIAR` and `DITI` (8.4.8).

### 8.4.8 Edited master information, absolute start time and comments

**Absolute start time.** `ABSS` gives the time of the first byte of sound data. The writer
chooses a frame count `A` with `A + N <= 6 479 999` (8.6), so that every time below has hours 0
to 23, and writes `ABSS` as the time of frame `A` by the formula of 8.4.7 item 1. Writing no
`ABSS` means `A = 0`. The usual choices are `A = 0`, or `A = a` so that marker times equal
timeline positions.

**Marker times.** A marker for frame `f` of the area's timeline (`a <= f <= b`) has the time of
frame `A + f - a`, by the formula of 8.4.7 item 1.

**`DIIN`** data is the following chunks only, with no ID before them, in an order that is the
writer's choice; its `D` is the space they occupy (8.4.1 item 1):

| Chunk | ID | Data |
|---|---|---|
| edited master ID | `EMID` | text bytes; content and length are the writer's choice, and [DSDIFF] recommends something unique per file |
| marker | `MARK` | 8.4.7; any number |
| artist | `DIAR` | u32 `k`, `k` text bytes, 0x00 if `k` is odd |
| title | `DITI` | u32 `k`, `k` text bytes, 0x00 if `k` is odd |

`EMID`, `DIAR` and `DITI` appear at most once each. `DIIN` may hold any of these chunks without
the others; for example `MARK`, `DIAR` and `DITI` without `EMID`. Only an edited master
requires `EMID` (8.4.11 item 1), and whether to write it in another file is the writer's choice
(8.6). `DIIN` may also hold none of them, in a file that is not an edited master: no `EMID`, no
markers, and no `DIAR` or `DITI`. [DSDIFF] makes every chunk inside `DIIN` optional and does not
require it to hold one. Writing such a `DIIN` with `D = 0` (12 bytes) or leaving `DIIN` out is
then the writer's choice (8.6). Both say the same: the file has no edited master information. A text length `k` of 0 is valid in
`DIAR`, `DITI`, a comment and a marker; when a string is empty, or becomes empty under 8.4.9,
writing the chunk or comment with `k = 0` or leaving it out is the writer's choice (8.6). Whether to write `DIIN`, `DIAR` and
`DITI`, and which reported strings fill `DIAR` and `DITI` (for example the album or disc
artist and title of 2.5.4), are the writer's choices (8.6). In a file with a source track (8.1 item 7), these
strings may also be that track's performer and title (specification 3, 3.5, item types 0x02
and 0x01).

**`COMT`** data is a u16 comment count followed by the comments, so its `D` is 2 plus the
comments' sizes, each `14 + k + (k mod 2)`. Each comment is:

| Size | Type | Field |
|---|---|---|
| 2 | u16 | year, 0 to 65 535 |
| 1 | u8 | month, 0 to 12 |
| 1 | u8 | day, 0 to 31 |
| 1 | u8 | hour, 0 to 23 |
| 1 | u8 | minutes, 0 to 59 |
| 2 | u16 | type: 0 general, 1 channel, 2 sound source, 3 file history |
| 2 | u16 | reference (below) |
| 4 | u32 | text length `k` |
| `k` | text bytes | text, then 0x00 if `k` is odd (counted in the comment, not in `k`) |

The reference is 0 for a general comment. For a channel comment it is 0 (all channels) or a
channel number from 1 to `C`. For a sound source it is 0 (DSD), 1 (analogue) or 2 (PCM). For
file history it is 0 (general remark), 1 (operator), 2 (machine), 3 (time zone) or 4
(revision). Every comment is an even number of bytes. Whether to write comments, and their
time stamps, types and text, are the writer's choices (8.6). [DSDIFF] describes time stamps
for a device without a clock: the year counts the comments from 0000 to 1999, with month,
day, hour and minutes 0.

### 8.4.9 Text in DSDIFF

All DSDIFF text is text bytes (0x20 to 0x7E). A reported string is written only as text bytes:
code points U+0020 to U+007E map to the byte of the same value. How other code points are
written (replaced, transliterated or removed) is the writer's choice (8.6). A writer MUST NOT
write a byte outside 0x20 to 0x7E as text.

### 8.4.10 Tracks as markers

Markers describe a **program**. It starts with a program start marker and is followed by
tracks, each from a track start to the next track start or track stop. Before the first track
start is pause 1; between a track stop and the next track start is a pause; after the last
track stop is the post-roll. The last track ends with a track stop.

A file holding the play range `[0, P)` (4.7.1 item 1) of an area with tracks 1 to `T`
(specification 3, 3.9.1, after 3.9.2) has these markers, in this order:

- a program start at frame 0;
- for each track `i` in order: a track start at `s_i`; then a track stop at `s_i + d_i` if
  `i = T`, or if `s_i + d_i < s_{i+1}`.

When `s_1 = 0`, the program start and the first track start coincide, which [DSDIFF] allows.
Pause 1 is `[0, s_1)`, the pauses between tracks are those of 4.7.1 item 5, and the post-roll
is `[s_T + d_T, P)`.

1. An area with no tracks (`T = 0`) has no program, and the writer writes no markers.
2. This represents the tracks exactly only when they are in increasing order, do not overlap
   and are not empty: `0 < d_i` and `s_i + d_i <= s_{i+1}` for every `i`. For tracks that
   overlap or are out of order (4.7 item 3), or have duration 0, no marker list does. A writer
   then writes no markers, or markers of its own choice that it labels as approximate (8.6).
3. A file with a source track (8.1 item 7) MAY carry markers: [DSDIFF] allows `DIIN` and
   `MARK` in any DSDIFF file. A program of one track `i` is a program start at frame `a`, a
   track start at `max(a, s_i)` and a track stop at `min(b, s_i + d_i)`, with times by 8.4.8.
   It is written only when `max(a, s_i) < min(b, s_i + d_i)`; otherwise (for example for the
   track with pause of a track of duration 0) the file has no markers.
4. A file holding a frame range `[a, b)` with no source track has no markers, or, as the
   writer chooses (8.6), the **clipped program** of the range: a program start at `a`; then,
   for each track `i` in order with `max(a, s_i) < min(b, s_i + d_i)`, a track start at
   `max(a, s_i)` and a track stop at `min(b, s_i + d_i)` when it is the last such track or
   when the next such track starts later. The clipped program is written only when those
   clipped tracks meet item 2's condition; for the play range `[0, P)` it is the program of
   the main text.

Informative: [DSDIFF] says the sound data before the program start and the post-roll are not
part of the area. A disc made from a file of the play range therefore ends its area at
`s_T + d_T`, not at `P`, when `s_T + d_T < P`. No marker list can keep `[s_T + d_T, P)` in the
area without lengthening the last track.

### 8.4.11 Edited master

[DSDIFF] calls a file an **edited master** when it holds one area ready for disc creation. No
field marks a file as one: a file is an edited master when all of the following hold. A writer
that offers edited-master output MUST produce files that meet them all, and refuses an area
that cannot.

An edited master of this specification holds exactly the play range `[0, P)` of the area, with
the markers of 8.4.10's main text. A writer refuses edited-master output for any other range.

1. **Chunks present:** `FVER`; `PROP` with `FS  `, `CHNL`, `CMPR`, `ABSS` and `LSCO`; the
   sound data; `DIIN` with `EMID` and the markers. DST sound data has `FRTE` and the `DSTF`
   chunks.
2. **Area:** a layout of stereo, 3/2 or 5.1, with the channel IDs and `LSCO` value of 8.4.4.
   An area with another channel count or an unspecified layout cannot be an edited master.
3. **Times:** the sample rate is 2 822 400 Hz, the DST frame rate is 75, and the absolute
   start time and every marker lie at a frame boundary (`samples mod 37 632 = 0`).
4. **Markers:** they represent one program (8.4.10). They are in ascending time order, and
   all times are distinct except that the program start may coincide with the first track
   start. Every marker channel is 0.
5. **Tracks:** 1 to 255 tracks, each with a duration `d_i` (after specification 3, 3.9.2) of
   at least 75 frames (1 second); pauses of any length, including 0. The tracks meet 8.4.10
   item 2. A post-roll `[s_T + d_T, P)` of any length, including 0, is allowed (see the note
   to 8.4.10).
6. **Duration:** the program, from program start to the last track stop, is at most 255
   minutes, 59 seconds and 74 frames.
7. **Index markers**, if any, lie inside a track, at most 254 per track. The TOC of
   specification 3 has no index points, so this specification writes none.

Caller chunks (8.4.2 item 5) may follow the local chunks. [DSDIFF] recommends `DIAR`, `DITI`
and `COMT`, and pause 1 and post-roll of 2 seconds. An
area that does not meet items 2 to 6 can still be written as a DSDIFF file, but not as an
edited master.

## 8.5 Not covered

This specification does not define reading DSF or DSDIFF files, the content of ID3v2.3 tags
(8.2.6 item 4) or of caller chunks (8.4.2 item 5), ID3 tags in DSDIFF files ([DSDIFF] defines
none), DSF files with DST data ([DSF] defines only DSD raw), sample rates
other than 2 822 400 Hz, `MANF` chunks, or artwork and other ID3 frames not listed in 8.3.5.

## 8.6 Writer's choices

Every point where the documents allow more than one valid output. A writer that offers
byte-identical output to another program needs to make each choice the same way.

| No. | Point | Options | Where |
|---|---|---|---|
| 1 | DSF bits per sample | 1 (bits reversed) or 8 (unchanged) | 8.2.3 |
| 2 | DSF channel type for an unspecified layout | a type with count `C` from 8.2.4, or refuse with an error code of the writer's own | 8.2.4 |
| 3 | DSF tag | none, an ID3v2.4 tag by 8.3, or a caller-supplied ID3v2.3 tag | 8.2.6 |
| 4 | ID3 footer | footer and no padding (as [ID3] 3.4 requires for a tag after the audio), or no footer | 8.3.1 |
| 5 | ID3 padding, without a footer | none, or any number of 0x00 bytes | 8.3.1 |
| 6 | ID3 frame order | any | 8.3.3 |
| 7 | ID3 text encoding | per frame: 0x00 (when allowed), 0x01 with either byte order, 0x02 or 0x03 | 8.3.4 |
| 8 | ID3 terminator after a text frame's last string, the `TXXX` value and `COMM` text | written or not | 8.3.4 |
| 9 | Control characters in ID3 text | removed or replaced | 8.3.4 |
| 10 | Splitting a TOC string into several ID3 values | no split, or a split of the writer's choice | 8.3.4 |
| 11 | Which TOC items are tagged, and the "Choice" cells of the frame table | as listed | 8.3.5 |
| 12 | `COMM` language and description; `TXXX` descriptions; the separator when both messages share one `COMM` | any | 8.3.5, 8.3.6 |
| 13 | DSDIFF position of `COMT`, `DIIN` and `DSTI` | any after `FVER`, `DSTI` only with DST | 8.4.2 |
| 14 | Order of chunks inside `PROP` and inside `DIIN` | any | 8.4.3, 8.4.8 |
| 15 | DSDIFF channel IDs and `LSCO` for an unspecified layout | `C` distinct IDs from `C000` to `C999`; `LSCO` 65 535 or none | 8.4.4 |
| 16 | `LSCO` for a defined layout | written or not (required in an edited master) | 8.4.3 |
| 17 | `ABSS` frame count `A` | none (`A = 0`), or any `A` with `A + N <= 6 479 999`, such as 0 or `a` | 8.4.8 |
| 18 | Invalid DST frames in DST files | stored bytes, or the 0x55-fill equivalent | 8.4.6 |
| 19 | `DSTC` frame CRCs | all or none | 8.4.6.1 |
| 20 | `DSTI` | written or not | 8.4.6.2 |
| 21 | `DSTI` offset | position of the `DSTF` chunk, or of its data (12 bytes later) | 8.4.6.2 |
| 22 | Whether `D` of `MARK`, `DIAR` and `DITI` counts the text's pad byte | counted or not | 8.4.7 |
| 23 | Marker text and track flags | any; track flags 0 unless the writer has a mapping | 8.4.7 |
| 24 | Whether to write `EMID` (required in an edited master) and its content; whether to write `DIIN`, `DIAR` and `DITI`, and their strings | any text bytes; any reported strings, such as the album or disc artist and title, or the source track's performer and title | 8.4.8 |
| 25 | Comments: whether, time stamps, types, references, text | any valid values | 8.4.8 |
| 26 | Code points outside U+0020 to U+007E in DSDIFF text | replaced, transliterated or removed | 8.4.9 |
| 27 | Markers for tracks that are not in increasing order, overlap, or are empty | none, or approximate markers labelled as such | 8.4.10 |
| 28 | Markers in a file with a source track | none, or a one-track program | 8.4.10 |
| 29 | Source track of a frame-range file | none, or a track the writer names that overlaps the range | 8.1 |
| 30 | Markers of a frame-range file without a source track | none, or the clipped program | 8.4.10 |
| 31 | An empty `DIAR`, `DITI` or comment | written with `k = 0`, or left out | 8.4.8 |
| 32 | What a DSF file holds | a range of positions, or a byte range of a chain | 8.2.7 |
| 33 | Caller chunks in DSDIFF | none, or any number after the last local chunk | 8.4.2 |
| 34 | A `DIIN` that would hold none of its chunks | written with `D = 0`, or left out | 8.4.8 |

## 8.7 Departures from [DSDIFF]

Some existing programs write DSDIFF files that differ from [DSDIFF] in the ways below. These are
not writer's choices: a file with a departure does not conform to [DSDIFF] or to this
specification. They are defined here, byte for byte, so that a writer asked to reproduce such
files does so knowingly and the same way as others.

A writer MUST NOT make a departure unless its caller asks for that departure by its number.
A writer that offers one says, where it documents the option, that the file it writes with it
is not a conforming DSDIFF file. Everything not named in a departure stays as specified.

| No. | Departure | What changes |
|---|---|---|
| D1 | `CMPR` size counts the pad byte | 8.4.3: with plain DSD, `CMPR` is written with `D = 20` instead of 19. The pad byte 0x00 after the name is still written and is now counted, so the chunk occupies the same 32 bytes and no other size changes. With DST, `D` is 16 and even, so nothing changes. |
| D2 | DSDIFF text as ISO 8859-1 | 8.4.9: in `DIAR`, `DITI`, marker text and comment text, a code point from U+0020 to U+007E or from U+00A0 to U+00FF is written as the one byte of the same value, and every other code point is removed. Text lengths `k` count the bytes written. `EMID` and IDs are unchanged. |

1. **Why D1 is not a choice.** [DSDIFF] section 2.3 says the pad byte is not included in a
   chunk's data size, and the `CMPR` structure has no padding of its own. `DIAR`, `DITI` and
   comments do: their text "must be padded" inside the structure, which is why their `D` is a
   choice (8.4.7 item 6) and `CMPR`'s is not. A reader that follows [DSDIFF] still finds the
   compression ID, count and name where they belong.
2. **Why D2 is not a choice.** [DSDIFF] section 2.2 defines a text byte as a byte from 0x20 to
   0x7E, and every text field above is made of text bytes. [DSDIFF] names no other
   character set. A reader that follows [DSDIFF] may show bytes 0xA0 to 0xFF otherwise, or
   reject them.
