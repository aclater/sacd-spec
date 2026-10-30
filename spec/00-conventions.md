# 0. Conventions used by every specification

This document defines the terms, notation and outcome vocabulary that specifications 1 to 7
share. It is normative.

## 0.1 Requirement keywords

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED,
MAY and OPTIONAL are to be interpreted as described in RFC 2119 and RFC 8174 when, and only
when, they appear in capitals.

A **reader** is an implementation that reads disc images as these specifications describe.
A reader that satisfies every MUST and MUST NOT in specifications 1 to 5 and 7 is
**conforming**. Specification 6 (the PS3 network protocol) is an optional extension: a reader
that implements it MUST follow it.

## 0.2 Numbers and bytes

1. A **byte** is 8 bits. Bit 0 is the least significant bit (LSB) and bit 7 the most
   significant bit (MSB).
2. Unless a specification says otherwise, every multi-byte integer stored in a disc image is
   unsigned and **big-endian** (most significant byte first).
3. A field written as "bits a to b" of a byte or word (a < b) is the unsigned integer formed by
   those bits, with bit b as its most significant bit and bit a as its least significant bit.
   For example, "bits 5 to 7" of the byte 0xE4 is 7, and "bits 2 to 4" of it is 1.
4. Field types in syntax tables:

   | Type | Size | Meaning |
   |---|---|---|
   | `u8` | 1 byte | unsigned integer, 0 to 255 |
   | `u16` | 2 bytes | unsigned big-endian integer, 0 to 65 535 |
   | `u32` | 4 bytes | unsigned big-endian integer, 0 to 4 294 967 295 |
   | `chars[n]` | n bytes | n bytes compared or decoded as stated for the field |
   | `bytes[n]` | n bytes | n bytes with no defined interpretation |
   | `tc` | 3 bytes | a timecode (0.5) |

5. Offsets are byte offsets from the start of the structure the table describes, written in
   decimal. Ranges `[a, b)` include `a` and exclude `b`; ranges `[a, b]` include both ends.
6. Bytes that a syntax table does not list, or lists as reserved, MUST be ignored by readers.
   Their values never change an outcome.
7. Integer arithmetic on values read from an image MUST NOT overflow. A reader MUST use at
   least 64-bit arithmetic for byte positions (a 32-bit sector number multiplied by a sector
   size exceeds 32 bits).
8. `floor(x)` is the largest integer not greater than x. `min` and `max` have their usual
   meaning.

## 0.3 Sectors and LSNs

1. A **logical sector** is 2048 bytes of user data.
2. A **logical sector number (LSN)** numbers logical sectors from 0 at the start of the image.
3. How logical sectors are stored in an image file (2048, 2054 or 2064 bytes per stored
   sector) is defined in specification 1. Every other specification addresses data only by
   LSN and by byte offset within logical sectors.

## 0.4 Signatures

A **signature** is an 8-byte field compared byte for byte with an ASCII string, for example
`SACDMTOC`. The comparison is exact and case-sensitive. A signature written in this document as
`SACD_IGL` contains the byte 0x5F (underscore).

## 0.5 Timecodes and frames

1. Audio is divided into **frames** of exactly 1/75 second. At the only defined sample rate,
   64 x 44 100 Hz = 2 822 400 Hz ("DSD64"), a frame holds 37 632 one-bit samples per
   channel, that is **4704 bytes per channel**.
2. A **timecode** is 3 bytes: minutes `m` (byte 0), seconds `s` (byte 1) and frames `f`
   (byte 2), each a `u8`. Its **frame number** is

       frame number = (m x 60 + s) x 75 + f

   computed exactly. Readers MUST NOT reject or correct a timecode whose `s` is 60 or more
   or whose `f` is 75 or more: the formula applies to every byte value. The largest frame
   number a timecode can express is (255 x 60 + 255) x 75 + 255 = 1 166 880.
3. When a frame number `n` has to be shown as a timecode, it is written as minutes
   `floor(n / 4500)`, seconds `floor(n / 75) mod 60` and frames `n mod 75`.
4. Frame numbers within an area start at 0, the first frame of the area's audio.

## 0.6 DSD byte layout

1. A **DSD byte** holds 8 consecutive one-bit samples of one channel. The earliest sample is
   in bit 7 (the MSB) and the latest in bit 0.
2. A frame of `C` channels in **interleaved layout** is `C x 4704` bytes. Byte `k x C + c`
   holds DSD byte `k` (0 to 4703) of channel `c` (0 to C-1).
3. Two byte values are used as fill:

   | Value | Name | Used for |
   |---|---|---|
   | 0x69 | missing-frame fill | a frame the disc does not supply (specification 4) |
   | 0x55 | invalid-frame fill | a DST frame that cannot be decoded (specifications 4 and 5) |

   Both are idle patterns that represent silence. The difference lets a test tell the two
   cases apart, and conformance depends on it.

## 0.7 Text

1. **Code point** means a Unicode scalar value. Strings produced by a reader are sequences of
   Unicode scalar values; they never contain surrogates.
2. U+FFFD is the replacement character.
3. How bytes in an image become text is defined in specification 2, section 2.6. Every
   specification that reads text uses that procedure.

## 0.8 Outcomes

Hostile input is expected. Every value read from an image is checked before use, and every
check has one of the outcomes below. Specification 7 lists every check with its outcome and
its diagnostic code.

| Outcome | Meaning |
|---|---|
| **reject the image** | The reader reports an error with the stated **error code** and produces no TOC and no audio. Whether it also reports warnings it emitted before the rejection is not specified, and the corpus does not compare them. |
| **skip the area** | The area is left out of the TOC and produces no audio. The reader emits a warning with the stated code and continues with the other area. Warnings about the area as a whole that were emitted before it was skipped (`area-toc1-unreadable`, `area-toc2-unreadable`, `area-toc-copies-differ`) remain reported; warnings about its tracks (codes beginning `track-`) are never reported for a skipped area (specification 3, 3.1 item 5). |
| **skip the track** | The track remains in the TOC, but the reader produces no audio for it and reports the stated code for it. |
| **refuse the track** | As "skip the track", but the condition is an error rather than an empty track. Specification 7 says which codes are which. |
| **replace the frame** | One audio frame is replaced by a fill pattern (0.6). Neighbouring frames are unaffected. |
| **warn** | The reader emits a warning with the stated code and continues; the value is used as stated. |
| **clamp** | The value is reduced to the stated bound, with a warning. |

A reader MUST NOT crash, hang, read outside the image, or allocate memory in proportion to
any value read from an image without first bounding that value as these specifications state.

Warnings and errors are identified by the **diagnostic codes** in specification 7. A reader
MAY emit additional warnings. A reader MUST emit every warning these specifications require,
with its code; how it presents them (log, list, callback) is not specified.

## 0.9 Symbols

Single-letter symbols (`P`, `C`, `N`, `T`, `H`, `K`, `Q`, `L` and others) are local to the
specification, and often to the section, that defines them. For example `C` is a channel count
in specification 4 and the arithmetic decoder's code value in specification 5. When one
specification uses a quantity from another, it names it in words.

## 0.10 Document map

| No. | Document | Content |
|---|---|---|
| 0 | [00-conventions.md](00-conventions.md) | this document |
| 1 | [01-disc-image-layout.md](01-disc-image-layout.md) | sector formats, detection, LSN addressing, where the TOCs are |
| 2 | [02-master-toc-and-text.md](02-master-toc-and-text.md) | the master TOC, disc text and character sets |
| 3 | [03-area-toc.md](03-area-toc.md) | the two area TOCs, tracks, track text, TOC-1 and TOC-2 |
| 4 | [04-audio-multiplexing.md](04-audio-multiplexing.md) | audio sectors, packets, frame reassembly, tracks |
| 5 | [05-dst.md](05-dst.md) | the DST frame format and decoding process |
| 6 | [06-ps3-protocol.md](06-ps3-protocol.md) | the PS3 SACD Ripper network protocol |
| 7 | [07-validation-and-limits.md](07-validation-and-limits.md) | every limit, every check, every outcome, diagnostic codes |

The conformance corpus and its manifest are described in [../corpus/README.md](../corpus/README.md).
