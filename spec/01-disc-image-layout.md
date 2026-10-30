# 1. Disc image layout

This specification defines how an SACD disc image stores logical sectors, how a reader
detects the storage format, how LSNs are addressed, and where the tables of contents (TOCs)
are. It is normative. Conventions are in [00-conventions.md](00-conventions.md).

## 1.1 Scope

1. An **image** is a finite sequence of bytes holding the Super Audio CD (Scarletbook) high
   density layer of one disc, from LSN 0 up to some last sector.
2. Images are assumed to hold **decrypted** data. Physical discs protect their audio areas;
   removing that protection is outside the scope of these specifications. An image whose
   audio is still encrypted is read like any other image; its audio sectors are then
   malformed (specification 4) and produce fill.
3. The CD layer of a hybrid disc is not part of an image.

## 1.2 Stored sector formats

An image stores each logical sector inside a **stored sector** of `P` bytes, with the 2048
bytes of user data at offset `D` within it. Three formats exist:

| Format | Stored sector size `P` | Data offset `D` | Bytes before the data | Bytes after the data |
|---|---|---|---|---|
| plain | 2048 | 0 | none | none |
| raw-2054 | 2054 | 6 | 6: sector ID (4) and ID error detection code (2) | none |
| raw-2064 | 2064 | 12 | 12: sector ID (4), ID error detection code (2), copyright management information (6) | 4: error detection code |

1. The bytes before and after the data in a raw format are not interpreted. A reader MUST NOT
   validate them, and their values MUST NOT change any outcome.
2. One image uses one format for all its sectors.

## 1.3 Addressing

For an image of `L` bytes in a format with stored sector size `P` and data offset `D`:

1. The image has `N = floor(L / P)` sectors. A trailing partial stored sector (when `L` is
   not a multiple of `P`) does not count, even if its user-data bytes are all present.
2. Logical sector `k` exists if and only if `k < N`. Its 2048 bytes are image bytes
   `[k x P + D, k x P + D + 2048)`.
3. A **read** of `n` sectors starting at LSN `a` (n >= 1) succeeds if and only if
   `a + n <= N`, computed without overflow. A read that does not succeed returns no data: a
   reader MUST NOT return the sectors that do exist from a read that fails.
4. LSNs read from an image are `u32` values. Any LSN, and any LSN plus a count, can exceed
   `N`; such reads fail as item 3 says. A reader MUST NOT wrap LSN arithmetic.

## 1.4 Format detection

The master TOC begins with the signature `SACDMTOC` at LSN 510 (section 1.5). A reader detects
the format by looking for that signature:

1. For each candidate in this order: plain (`P = 2048`, `D = 0`), raw-2064 (`P = 2064`,
   `D = 12`), raw-2054 (`P = 2054`, `D = 6`):
   - take the 8 image bytes starting at byte position `510 x P + D`;
   - if all 8 bytes exist and equal `SACDMTOC`, the image has this format; stop.
2. If no candidate matches, **reject the image** with error `not-scarletbook`.
3. The order is normative. An image that matches more than one candidate has the first
   matching format.
4. Detection looks at those 8 bytes only. It does not require LSN 510 to exist as a complete
   sector: an image whose length ends inside the master TOC is detected and then rejected by
   specification 2 (`master-toc-truncated`).
5. A reader MAY accept a caller's explicit choice of format instead of detecting it. When it
   detects, it MUST follow this section.

## 1.5 Disc layout

The high density layer is organised as follows. Only the structures marked "read" are used by
these specifications.

| LSN | Content | Read |
|---|---|---|
| 0 to 509 | file system and reserved sectors | no |
| 510 to 519 | master TOC, copy 1 (10 sectors; specification 2) | yes |
| 520 to 529 | master TOC, copy 2 | no |
| 530 to 539 | master TOC, copy 3 | no |
| from 540 | area TOCs and audio areas, at the positions the master TOC gives | yes |

### 1.5.1 Master TOC redundancy

1. The master TOC is stored three times, at LSN 510, 520 and 530. The copies are meant to be
   identical.
2. A reader MUST read the master TOC from LSN 510. Its outcome (accepted, rejected, and every
   value) MUST be exactly what LSN 510 to 519 yield. A reader MAY compare the copies and warn
   when they differ, but MUST NOT substitute a copy for LSN 510.

### 1.5.2 Areas

1. A disc has at most two audio **areas**: the **2-channel area** and the **multichannel
   area**. Either may be absent, but a readable disc has at least one (specification 7).
2. Each area has an area TOC stored twice, **TOC-1** and **TOC-2**, and a contiguous range of
   **audio sectors**. On real discs TOC-1 comes immediately before the audio sectors and TOC-2
   immediately after them, but readers MUST NOT assume this: the master TOC gives the
   position of each copy, and the area TOC gives the audio range.
3. The master TOC fields that locate the area TOCs are in specification 2, section 2.3. The
   way a reader chooses between TOC-1 and TOC-2 is in specification 3, section 3.2.
4. The area TOC fields that give the audio range (first and last audio LSN) are in
   specification 3, section 3.3. Audio sectors are read as specification 4 describes.
5. Structures may overlap or lie in implausible places in a hostile image. These
   specifications define the result of every such case; no structure is required to be at
   any particular LSN except the master TOC.

## 1.6 Limits

| Item | Value |
|---|---|
| Logical sector size | 2048 bytes |
| Stored sector sizes | 2048, 2054, 2064 bytes |
| LSN range | 0 to 2^32 - 1 (`u32`); only LSNs below `N` exist |
| Master TOC position | LSN 510, 10 sectors |
| Largest real disc | 4 173 824 sectors (dual-layer, 8.5 GB). A reader of image files MUST NOT rely on any upper bound on `N`. (A network client applies its own bound to the size a server reports, specification 6, 6.7.) |
