# 2. Master TOC and text

This specification defines the master TOC: its fields, the album set, catalogue numbers,
genres, the date, locales, text channels, the character sets and how text is decoded. It is
normative. Conventions are in [00-conventions.md](00-conventions.md); the layout of the image
is in [01-disc-image-layout.md](01-disc-image-layout.md).

## 2.1 Structure

The master TOC is 10 consecutive logical sectors starting at LSN 510:

| Sector | LSN | Signature | Content |
|---|---|---|---|
| 0 | 510 | `SACDMTOC` | disc information and area pointers (2.3) |
| 1 to 8 | 511 to 518 | `SACDText` | disc text, one sector per text channel (2.5) |
| 9 | 519 | `SACD_Man` | manufacturer information (2.7) |

## 2.2 Reading the master TOC

1. A reader MUST read all 10 sectors (LSN 510 to 519) as one read (specification 1, 1.3).
   If the read fails, **reject the image** with `master-toc-truncated`.
2. The first 8 bytes of sector 0 MUST be `SACDMTOC`; otherwise **reject the image** with
   `not-scarletbook`. (After detection this can only fail if a reader was told the format
   instead of detecting it.)
3. If the version major number (2.3, offset 8) is greater than 1, **reject the image** with
   `master-toc-version-unsupported`. Every minor number is accepted, and major number 0 is
   accepted.
4. The remaining fields are read as 2.3 to 2.7 describe. None of them rejects the image. An
   image whose master TOC passes items 1 to 3 is rejected later only if no area is readable
   (`no-audio-areas`, specification 3, 3.1).

## 2.3 Sector 0: disc information

| Offset | Size | Type | Field | Meaning |
|---|---|---|---|---|
| 0 | 8 | chars[8] | signature | `SACDMTOC` |
| 8 | 1 | u8 | version major | 1 on every known disc |
| 9 | 1 | u8 | version minor | 20 on every known disc (version 1.20) |
| 10 | 6 | | reserved | |
| 16 | 2 | u16 | album set size | number of discs in the album set |
| 18 | 2 | u16 | album sequence number | this disc's position in the set, from 1 |
| 20 | 4 | | reserved | |
| 24 | 16 | chars[16] | album catalogue number | fixed text field (2.6.5) |
| 40 | 16 | 4 x genre | album genres | 4 genre entries (2.4) |
| 56 | 8 | | reserved | |
| 64 | 4 | u32 | 2-channel TOC-1 LSN | 0 = no 2-channel area |
| 68 | 4 | u32 | 2-channel TOC-2 LSN | 0 = no backup copy |
| 72 | 4 | u32 | multichannel TOC-1 LSN | 0 = no multichannel area |
| 76 | 4 | u32 | multichannel TOC-2 LSN | 0 = no backup copy |
| 80 | 1 | u8 | disc flags | bit 7: hybrid disc (1 = the disc also has a CD layer). Bits 0 to 6 are not interpreted. |
| 81 | 3 | | reserved | |
| 84 | 2 | u16 | 2-channel area TOC size | in sectors; 0 = no 2-channel area |
| 86 | 2 | u16 | multichannel area TOC size | in sectors; 0 = no multichannel area |
| 88 | 16 | chars[16] | disc catalogue number | fixed text field (2.6.5) |
| 104 | 16 | 4 x genre | disc genres | 4 genre entries (2.4) |
| 120 | 2 | u16 | year | 0 = not set |
| 122 | 1 | u8 | month | 0 = not set |
| 123 | 1 | u8 | day | 0 = not set |
| 124 | 4 | | reserved | |
| 128 | 1 | u8 | text channel count | number of text channels (languages) |
| 129 | 7 | | reserved | |
| 136 | 32 | 8 x locale | locales | 8 locale entries (2.5.1) |
| 168 | 1880 | | not interpreted | |

### 2.3.1 Album set

1. The album set size and sequence number describe multi-disc releases: a box of three discs
   has set size 3, and its discs have sequence numbers 1, 2 and 3.
2. Readers report both values as stored. They are not validated: 0, and a sequence number
   greater than the set size, are reported unchanged.

### 2.3.2 Date

The year, month and day are reported as stored. They are not validated: a month of 13 or a
day of 255 is reported unchanged. A value of 0 means "not set".

### 2.3.3 Area pointers

The pointers at offsets 64 to 87 are used by specification 3, section 3.1. Their values are
not validated here.

## 2.4 Genres

A genre entry is 4 bytes:

| Offset | Size | Type | Field |
|---|---|---|---|
| 0 | 1 | u8 | genre table (category) |
| 1 | 2 | | reserved |
| 3 | 1 | u8 | genre index |

Readers report both numbers as stored, for all 4 entries of a list (including entries of
0, 0). The names below are informative; values outside the tables are reported as numbers and
are not an error.

| Genre table | Name |
|---|---|
| 0 | not used |
| 1 | general |
| 2 | Japanese |

General genre index (table 1):

| Index | Name | Index | Name | Index | Name |
|---|---|---|---|---|---|
| 0 | not used | 10 | Erotic | 20 | Pop Music |
| 1 | not defined | 11 | Folk | 21 | Rap |
| 2 | Adult Contemporary | 12 | Gospel | 22 | Reggae |
| 3 | Alternative Rock | 13 | Hip Hop | 23 | Rock Music |
| 4 | Children's Music | 14 | Jazz | 24 | Rhythm & Blues |
| 5 | Classical | 15 | Latin | 25 | Sound Effects |
| 6 | Contemporary Christian | 16 | Musical | 26 | Sound Track |
| 7 | Country | 17 | New Age | 27 | Spoken Word |
| 8 | Dance | 18 | Opera | 28 | World Music |
| 9 | Easy Listening | 19 | Operetta | 29 | Blues |

The meaning of genre indexes under table 2 is not defined by these specifications.

## 2.5 Text channels and locales

### 2.5.1 Locale entries

A locale entry is 4 bytes:

| Offset | Size | Type | Field | Meaning |
|---|---|---|---|---|
| 0 | 2 | chars[2] | language code | ISO 639-1 code, for example `en` or `ja` |
| 2 | 1 | u8 | character set code | see 2.6.2 |
| 3 | 1 | | reserved | |

1. The **language** is the two bytes mapped one-to-one to the code points U+0000 to U+00FF
   (ISO 8859-1), with every trailing U+0000 and U+0020 removed (from the end only). It may be
   empty. It is not validated.
2. The **character set code** is reported as the stored byte (0 to 255). The character set
   actually used for decoding is the stored byte's low 3 bits (2.6.2).

### 2.5.2 Reported locale list

1. Let `n` be the text channel count (offset 128). The reported locale list is the first
   `min(n, 8)` of the 8 entries, in order. If `n` is 0 the list is empty.
2. A count above 8 is not an error.

### 2.5.3 Which language is used

1. A disc may carry text in up to 8 languages, one per text channel. **Readers use only the
   first text channel.** Text in sectors 2 to 8 of the master TOC is not read.
2. Disc text (2.5.4) is decoded with the character set of **locale entry 0** (offset 138, low
   3 bits). This holds even if the text channel count is 0, and whatever the language code of
   entry 0 says.

### 2.5.4 Disc text sector

Sector 1 of the master TOC (LSN 511) holds the text of the first text channel:

| Offset | Size | Type | Field |
|---|---|---|---|
| 0 | 8 | chars[8] | signature `SACDText` |
| 8 | 8 | | reserved |
| 16 | 32 | 16 x u16 | string offsets, in the order of the table below |
| 48 | 2000 | | string storage |

| Index | Offset of its u16 | Text item |
|---|---|---|
| 0 | 16 | album title |
| 1 | 18 | album artist |
| 2 | 20 | album publisher |
| 3 | 22 | album copyright |
| 4 | 24 | album title, phonetic |
| 5 | 26 | album artist, phonetic |
| 6 | 28 | album publisher, phonetic |
| 7 | 30 | album copyright, phonetic |
| 8 | 32 | disc title |
| 9 | 34 | disc artist |
| 10 | 36 | disc publisher |
| 11 | 38 | disc copyright |
| 12 | 40 | disc title, phonetic |
| 13 | 42 | disc artist, phonetic |
| 14 | 44 | disc publisher, phonetic |
| 15 | 46 | disc copyright, phonetic |

1. If the first 8 bytes of sector 1 are `SACDText`, the reader decodes each of the 16 items:
   - Each offset is relative to the start of sector 1.
   - An offset of 0 means the item is absent.
   - An offset of 2048 or more means the item is absent. This is not reported.
   - Otherwise the item is the text string (2.6.4) starting at that offset, with the
     **containing buffer** being sector 1 alone (2048 bytes).
   - An item whose decoded string is empty is absent.
2. If the first 8 bytes of sector 1 are not `SACDText`, the disc has no text. If the text
   channel count is greater than 0, the reader **warns** `master-text-missing`; if it is 0,
   there is no warning.
3. The text is read when the signature is present, even if the text channel count is 0.
4. Offsets may point anywhere in the sector, including into the offset table itself, and
   several items may share one string. Both are valid.
5. "Phonetic" items carry a reading aid, typically in kana for Japanese titles.

Informative: a player that wants one title and one artist uses the disc title (or, if it is
absent, the album title) and the disc artist (or, if absent, the album artist).

## 2.6 Text decoding

### 2.6.1 Overview

Text in the master TOC and in the area TOCs is stored as byte strings in a character set named
by a locale entry. Converting such a string to code points has four steps: find its bytes
(2.6.4), decode them (2.6.2, 2.6.3), replace control characters, and decide whether it is
empty.

### 2.6.2 Character sets

The character set code's low 3 bits select the decoding. Higher bits are ignored, so a stored
code of 0xFF selects set 7.

| Code (low 3 bits) | Character set | Decoding |
|---|---|---|
| 0 | not assigned | ISO 8859-1 |
| 1 | ISO 646 IRV (ASCII) | ISO 8859-1 (bytes 0x80 to 0xFF still decode) |
| 2 | ISO 8859-1 | ISO 8859-1 |
| 3 | Shift JIS for music (RIS-506) | Shift_JIS-X0213, see below |
| 4 | KS C 5601-1987 (KS X 1001) | EUC-KR |
| 5 | GB 2312-80 | EUC-CN (GB 2312) |
| 6 | Big5 | Big5 |
| 7 | not assigned | ISO 8859-1 |

1. **ISO 8859-1** maps each byte value `b` to the code point U+00`b` (0x00 to 0xFF). It
   never fails.
2. **Shift_JIS-X0213** is the Shift encoding of JIS X 0213:2000 (planes 1 and 2), with these
   single-byte mappings:
   - 0x00 to 0x7F: JIS X 0201 Roman. This equals ASCII except that **0x5C decodes to
     U+00A5 (yen sign)** and **0x7E decodes to U+203E (overline)**.
   - 0xA1 to 0xDF: half-width katakana, U+FF61 to U+FF9F.
   - Lead bytes 0x81 to 0x9F and 0xE0 to 0xFC start a two-byte character; trail bytes are
     0x40 to 0x7E and 0x80 to 0xFC.
   - Some JIS X 0213 characters decode to two code points (a base and a combining mark); a
     reader MUST produce both.
   - A reader MAY also decode the characters added by JIS X 0213:2004. The corpus does not
     test them.
3. **EUC-KR**: bytes 0x00 to 0x7F are ASCII; a two-byte character is a lead byte 0xA1 to 0xFE
   followed by a trail byte 0xA1 to 0xFE, mapped by KS X 1001. Vendor extensions (such as the
   Unified Hangul Code) are not required.
4. **EUC-CN**: bytes 0x00 to 0x7F are ASCII; a two-byte character is a lead byte 0xA1 to 0xF7
   followed by a trail byte 0xA1 to 0xFE, mapped by GB 2312. Extensions (GBK, GB 18030) are not
   required.
5. **Big5**: bytes 0x00 to 0x7F are ASCII; a two-byte character is a lead byte 0xA1 to 0xF9
   followed by a trail byte 0x40 to 0x7E or 0xA1 to 0xFE. Vendor extensions (ETEN, CP950,
   HKSCS) are not required.

### 2.6.3 Invalid byte sequences

1. A byte sequence that is invalid in the selected character set (a lead byte with a bad or
   missing trail byte, an unmapped code, a byte that cannot start a character) MUST be
   replaced by U+FFFD. Decoding MUST continue after it.
2. How many U+FFFD a given invalid sequence produces, and whether a byte that follows an
   invalid lead byte is decoded again on its own, is implementation-defined. The RECOMMENDED
   behaviour is that of the WHATWG Encoding Standard for the corresponding decoder: one U+FFFD
   per invalid sequence, and an ASCII byte that follows an invalid lead byte is decoded again
   as itself.
3. Decoding MUST NOT produce surrogate code points, and MUST NOT fail.
4. Because of item 2, the conformance corpus checks such strings only up to the first U+FFFD
   (see [../corpus/README.md](../corpus/README.md)).

### 2.6.4 Text strings

A **text string** is located by a start position `q` in a containing buffer of `E` bytes. The
containing buffer is stated wherever a text string is used (sector 1 of the master TOC for
disc text; the whole area TOC for area and track text).

1. **Bytes.** The string's bytes are those from `q` up to, but not including, the first byte
   0x00, considering at most the 4096 bytes `[q, min(q + 4096, E))`. If no 0x00 occurs in that
   range, the string is all of it. The string can therefore be cut at 4096 bytes or at the end
   of the buffer, possibly in the middle of a multi-byte character; the cut part decodes per
   2.6.3.
2. **Decoding.** The bytes are decoded with the selected character set (2.6.2, 2.6.3).
3. **Controls.** Every decoded code point below U+0020 is replaced with U+0020 (space). No
   other code point is changed: U+007F, U+0080 to U+009F and all others are kept. Leading
   and trailing spaces are kept.
4. **Emptiness.** If the result is the empty string the text item is absent. Whether an empty
   item is "absent" or "present and empty" matters for reporting; these specifications always
   treat it as absent.

The 4096-byte limit is normative: it bounds the work per string, and it affects the result
when a string is longer.

### 2.6.5 Fixed text fields

Catalogue numbers (2.3) and ISRCs (specification 3) are **fixed text fields** of a stated
length:

1. Take the field's bytes up to, but not including, the first 0x00 (all of them if there is
   none).
2. Map each byte value `b` to the code point U+00`b` (ISO 8859-1), whatever the locale says.
3. Remove from both ends every code point in this set W: U+0009, U+000A, U+000B, U+000C,
   U+000D, U+001C, U+001D, U+001E, U+001F, U+0020, U+0085, U+00A0. Code points in W inside
   the string are kept.
4. Control characters are **not** replaced in fixed text fields (only W is stripped from the
   ends).
5. The result may be empty. A catalogue number is then reported as the empty string; an empty
   ISRC means "no ISRC" (specification 3).

## 2.7 Manufacturer sector

1. Sector 9 of the master TOC (LSN 519) is the manufacturer information sector. If its first
   8 bytes are `SACD_Man`, bytes 8 to 2047 are opaque manufacturer data. Readers MAY expose
   them; they are not interpreted.
2. If the signature is absent the reader **warns** `manufacturer-missing`. Nothing else
   depends on this sector.

## 2.8 Summary of reported values

| Value | Source | Validation |
|---|---|---|
| version (major, minor) | 8, 9 | major > 1 rejects the image |
| album set size, sequence number | 16, 18 | none |
| album and disc catalogue numbers | 24, 88 | fixed text field |
| album and disc genres | 40, 104 | none |
| hybrid flag | 80 bit 7 | none |
| date | 120 to 123 | none |
| locales | 128, 136 | count capped at 8 |
| disc text (16 items) | sector 1 | 2.5.4, 2.6 |
| manufacturer sector present | sector 9 | warning if absent |
