# 5. DST frame format and decoding process

This specification defines the complete frame syntax and decoding process of DST (Direct
Stream Transfer), the lossless coding of 1-bit audio used on Super Audio CD and defined in
MPEG-4 Audio (ISO/IEC 14496-3). It is self-contained: a decoder can be written from it alone.
It is normative except where marked informative. Conventions are in
[00-conventions.md](00-conventions.md).

Sections 5.1 to 5.4 define the syntax, 5.5 the decoding process, 5.6 the validity rules, 5.7
the constants and limits. Annex 5.A (informative) describes how to produce valid frames.

## 5.1 Parameters and notation

### 5.1.1 Inputs and output

A DST decoder decodes one frame at a time. Frames are independent: no state carries over from
one frame to the next. The decoder takes:

| Input | Meaning | Source on SACD |
|---|---|---|
| frame bytes | `FB` bytes | the reassembled frame (specification 4, 4.4) |
| `FB` | frame length in bytes | the reassembled frame's size |
| `Nch` | number of channels, 1 to 6 | the area's channel count (specification 3) |

and produces either an **output frame** of `FL x Nch` bytes in interleaved layout
(specification 0, 0.6), or the result **invalid**. On SACD an invalid frame is replaced by
`FL x Nch` bytes of 0x55 (specification 4, 4.6).

### 5.1.2 Frame size constants

DST is parametric in the sampling rate, but SACD defines only 64 x 44 100 Hz. These
specifications fix it:

| Symbol | Meaning | Value |
|---|---|---|
| `NB` | bits (samples) per channel per frame | 37 632 |
| `FL` | bytes per channel per frame | 4704 |

Informative: for a rate of `k x 44 100` Hz, `NB = 588 x k` and `FL = NB / 8`; the field widths
of 5.3 follow from `FL`. Nothing else depends on the rate.

### 5.1.3 Notation

1. `bitlen(x)` for an integer `x >= 0` is the smallest `y >= 0` with `2^y > x`: bitlen(0) =
   0, bitlen(1) = 1, bitlen(2) = bitlen(3) = 2, bitlen(4) to bitlen(7) = 3, bitlen(4576) = 13.
2. `uimsbf(n)` is an n-bit unsigned integer, most significant bit first. A field of width 0
   occupies no bits and has the value 0.
3. `tcimsbf(n)` is an n-bit two's-complement integer, most significant bit first, with range
   -2^(n-1) to 2^(n-1) - 1.
4. `bslbf(1)` is one bit.
5. `floor` is mathematical floor; all divisions are exact, followed by the stated rounding.
   `floor((x + 4) / 8)` equals an arithmetic right shift of `x + 4` by 3 for every integer x.
6. `|x|` is the absolute value, computed in ordinary integers.

### 5.1.4 Bit order

The frame is a sequence of bits: bit 0 is the MSB of byte 0, bit 7 its LSB, bit 8 the MSB of
byte 1, and so on, up to bit `8 x FB - 1`. Fields are read in that order, MSB first, and are
not byte-aligned. Fields follow each other with no padding unless stated.

## 5.2 Frame syntax: top level

| Element | Bits | Condition | Meaning |
|---|---|---|---|
| `DST_coded` | 1 | always | 1 = DST-coded frame; 0 = plain frame |
| *if `DST_coded` = 0:* | | | |
| `plain_X_bit` | 1 | | ignored, any value |
| `stuffing` | 6 | | MUST be 0 |
| `DSD_data` | the rest | | raw DSD in interleaved layout |
| *if `DST_coded` = 1:* | | | |
| `segmentation` | var | | 5.3.1 |
| `mapping` | var | | 5.3.3 |
| `half_probability` | `Nch` | | 5.3.4 |
| `filter_coefficient_sets` | var | | 5.3.5 |
| `probability_tables` | var | | 5.3.6 |
| `arithmetic_code` | the rest | | 5.3.7 |

The bits from `DST_coded` up to the end of the probability tables are the **header**; let `H`
be their number. The rest of the frame, `fs = 8 x FB - H` bits, is the arithmetic code.

### 5.2.1 Frame length

1. A frame of 0 or 1 byte (`FB <= 1`) is invalid.
2. Plain frame (`DST_coded` = 0): the header is exactly 1 byte. The output is the first
   `min(FB - 1, FL x Nch)` bytes after it, followed by zero bytes up to `FL x Nch` bytes when
   the frame is shorter. Extra bytes beyond `1 + FL x Nch` are ignored.
3. Coded frame: if the header needs more bits than the frame has (`H > 8 x FB`), the frame is
   invalid. A header that ends exactly at the end of the frame (`fs = 0`) is valid.

## 5.3 Coded frame syntax

### 5.3.1 segmentation

There are two **segmentations** per frame, one for **filters** and one for **probability
tables** (ptables). Each divides each channel's `NB` bits into consecutive segments.

| Element | Bits | Condition | Meaning |
|---|---|---|---|
| `same_segmentation` | 1 | always | 1 = the ptable segmentation equals the filter segmentation and is not sent |
| `table_segmentation(FILTER)` | var | always | 5.3.2 |
| `table_segmentation(PTABLE)` | var | if `same_segmentation` = 0 | 5.3.2 |

If `same_segmentation` = 1, the ptable segmentation is a copy of the filter segmentation: the
same number of segments per channel at the same bit positions.

### 5.3.2 table_segmentation(kind)

Parameters:

| kind | `Lmin` (minimum segment length, bits) | `Lmin / 8` (bytes) | Most explicit lengths, shared (`same_segm_all_ch` = 1) | Most explicit lengths, per channel (`same_segm_all_ch` = 0) |
|---|---|---|---|---|
| FILTER | 1024 | 128 | 4 | 3 |
| PTABLE | 32 | 4 | 7 | 7 |

State for one segmentation:

- `R`, the segment **resolution** in bytes. It is sent at most once per segmentation, before
  the first explicit length of that segmentation (whichever channel it belongs to), and is
  then shared by all its channels. If no explicit length is sent, `R` is not sent and is not
  needed.
- For the channel being read: `rem`, the bytes still available, initially `FL - Lmin/8`
  (4576 for filters, 4700 for ptables); and `n`, the number of explicit lengths read so far,
  initially 0.

Syntax:

| Element | Bits | Condition / loop | Meaning |
|---|---|---|---|
| `same_segm_all_ch` | 1 | always | 1 = one list is sent and used for every channel |
| *for each channel to read (channel 0 only if `same_segm_all_ch` = 1, else channels 0 to `Nch - 1` in order); reset `rem` and `n`:* | | | |
| *repeat:* | | | |
| `end_of_channel_segm` | 1 | | 1 = this channel's list ends |
| *if `end_of_channel_segm` = 0:* | | | |
| `segment_resolution` | uimsbf(bitlen(FL - Lmin/8)) | only if `R` has not yet been read for this segmentation | `R` |
| `segment_length` | uimsbf(bitlen(floor(rem / R))) | | `L`, the segment length in units of `R` bytes |
| *until `end_of_channel_segm` = 1* | | | |

At DSD64 the resolution field is 13 bits for both kinds.

Semantics:

1. Each explicit length `L` makes a segment of `8 x R x L` bits. After it, `rem` decreases by
   `R x L` and `n` increases by 1. The width of the next length field is computed from the new
   `rem`.
2. A channel with `n` explicit lengths has `n + 1` segments. The last one is implicit: it runs
   from the end of the explicit segments to bit `NB`.
3. With `same_segm_all_ch` = 1, channel 0's segments are used for every channel.
4. Segment positions: for a channel with explicit lengths `L_0 .. L_(n-1)`, let `B_0 = 0` and
   `B_(j+1) = B_j + 8 x R x L_j`. Segment `j < n` covers bits `[B_j, B_(j+1))`; segment `n`
   covers `[B_n, NB)`. Every boundary is a multiple of 8.

Validity (each makes the frame invalid):

1. `R = 0`, or `R > FL - Lmin/8`.
2. An explicit segment with `R x L < Lmin / 8` bytes (this includes `L = 0`), or with
   `R x L > rem` (it would leave less than `Lmin` bits for the rest of the channel).
3. An `end_of_channel_segm` = 0 read when the channel already has the most explicit lengths
   allowed for this kind and mode (table above).

Consequences: every segment, the implicit one included, is at least `Lmin` bits long. A
channel has at most 5 filter segments in shared mode and 4 in per-channel mode, and at most 8
ptable segments in either mode.

### 5.3.3 mapping

The mapping assigns a **table number** to each segment: a filter number to each filter
segment, and a ptable number to each ptable segment.

| Element | Bits | Condition | Meaning |
|---|---|---|---|
| `same_mapping` | 1 | always | 1 = the ptable mapping equals the filter mapping and is not sent |
| `table_mapping(FILTER)` | var | always | uses the filter segmentation |
| `table_mapping(PTABLE)` | var | if `same_mapping` = 0 | uses the ptable segmentation |

**table_mapping.** `T` is the number of distinct tables introduced so far. It starts at 1:
segment 0 of channel 0 always has table 0, and its number is not sent. `nseg[c]` is the number
of segments of channel c in this segmentation.

| Element | Bits | Condition / loop | Meaning |
|---|---|---|---|
| `same_map_all_ch` | 1 | always | 1 = one list of table numbers is sent and used for every channel |
| *if `same_map_all_ch` = 1: for j = 1 to `nseg[0]` - 1:* | | | |
| `table_index` | uimsbf(bitlen(T)) | | table number of segment j, for every channel |
| *if `same_map_all_ch` = 0: for c = 0 to `Nch` - 1, for j = 0 to `nseg[c]` - 1, except (c = 0, j = 0):* | | | |
| `table_index` | uimsbf(bitlen(T)) | | table number of segment j of channel c |

Semantics and validity:

1. The width of each `table_index` is computed from the current `T` before it is read:
   width 1 for T = 1, 2 for T = 2 or 3, 3 for T = 4 to 7, 4 for T = 8 to 15.
2. A value `v < T` reuses table v. A value `v = T` introduces a new table, and T increases
   by 1. A value `v > T` makes the frame **invalid**.
3. Tables are therefore numbered in order of first use, scanning channel 0's segments, then
   channel 1's, and so on.
4. With `same_map_all_ch` = 1: after channel 0's numbers are read, every channel must have the
   same number of segments as channel 0 in this segmentation, otherwise the frame is
   **invalid**. Channel 0's numbers are then used for every channel. If `nseg[0] = 1`, the
   mapping is the flag bit alone and every channel uses table 0.
5. When the mapping is complete, the final T is the number of tables of this kind: `NF`
   filters or `NP` ptables. If `T > 2 x Nch` the frame is **invalid**. This check is made once,
   after the whole mapping.

Copied mapping (`same_mapping` = 1):

1. For every channel, the ptable segmentation must have the same number of segments as the
   filter segmentation, otherwise the frame is **invalid**. Their lengths may differ.
2. The ptable number of segment j of channel c is the filter number of segment j of channel c,
   and `NP = NF`.

### 5.3.4 half_probability

| Element | Bits | Loop | Meaning |
|---|---|---|---|
| `half_prob[c]` | 1 | c = 0 to `Nch` - 1 | 1 = the first bits of channel c are coded with probability 1/2 (5.5.5) |

### 5.3.5 filter_coefficient_sets

One set per filter number f = 0 to `NF` - 1, in order:

| Element | Bits | Condition | Meaning |
|---|---|---|---|
| `coded_pred_order` | uimsbf(7) | | prediction order `N_f = coded_pred_order + 1`, 1 to 128 |
| `coef_coded` | 1 | | 0 = coefficients sent verbatim; 1 = entropy coded |
| *if `coef_coded` = 0:* | | | |
| `coef[i]` | tcimsbf(9) | i = 0 to `N_f` - 1 | coefficient `a_i`, -256 to 255 |
| *if `coef_coded` = 1:* | | | |
| `coding_method` | uimsbf(2) | | method k, 0 to 2 |
| `coef[i]` | tcimsbf(9) | i = 0 to k | the first `K = k + 1` coefficients, verbatim |
| `rice_param` | uimsbf(3) | | Rice parameter m, 0 to 7 |
| `coef_residual[i]` | Rice code (5.3.8) | i = K to `N_f` - 1 | residual `r_i` |

Reconstruction of coded coefficients, for i = K to `N_f` - 1 in increasing order:

    x_i = sum over t = 0 .. K-1 of  b[k][t] x a_(i-1-t)
    a_i = r_i - floor((x_i + 4) / 8)

| Method k | K | b[k][0] | b[k][1] | b[k][2] |
|---|---|---|---|---|
| 0 | 1 | -8 | | |
| 1 | 2 | -16 | 8 | |
| 2 | 3 | -9 | -5 | 6 |

Validity (each makes the frame invalid):

1. `coding_method` = 3.
2. `K >= N_f` (the method's order is not less than the prediction order). So a filter of order
   1 must be sent verbatim. This is checked before the K verbatim coefficients are read.
3. A reconstructed `a_i` outside -256 to 255.

The Rice parameter is not range-checked. Coefficients with index `i >= N_f` are 0.

### 5.3.6 probability_tables

One table per ptable number p = 0 to `NP` - 1, in order:

| Element | Bits | Condition | Meaning |
|---|---|---|---|
| `coded_ptable_len` | uimsbf(6) | | table length `M_p = coded_ptable_len + 1`, 1 to 64 |
| *if `M_p` = 1:* | | | **nothing more is read**; the single entry is `P_p[0] = 128` |
| *if `M_p` > 1:* | | | |
| `ptable_coded` | 1 | | 0 = entries verbatim; 1 = entropy coded |
| *if `ptable_coded` = 0:* | | | |
| `entry[i]` | uimsbf(7) | i = 0 to `M_p` - 1 | `P_p[i] = entry + 1`, 1 to 128 |
| *if `ptable_coded` = 1:* | | | |
| `coding_method` | uimsbf(2) | | method k, 0 to 2 |
| `entry[i]` | uimsbf(7) | i = 0 to k | `P_p[i] = entry + 1` for the first `K = k + 1` entries |
| `rice_param` | uimsbf(3) | | m |
| `entry_residual[i]` | Rice code | i = K to `M_p` - 1 | residual `r_i` |

Reconstruction, for i = K to `M_p` - 1, on the stored entries (the +1 already applied):

    x_i = sum over t = 0 .. K-1 of  c[k][t] x P_p[i-1-t]
    P_p[i] = r_i - floor((x_i + 4) / 8)

| Method k | K | c[k][0] | c[k][1] | c[k][2] |
|---|---|---|---|---|
| 0 | 1 | -8 | | |
| 1 | 2 | -16 | 8 | |
| 2 | 3 | -24 | 24 | -8 |

Validity (each makes the frame invalid): `coding_method` = 3; `K >= M_p`; a reconstructed
entry outside 1 to 128.

Note that a one-entry table has no `ptable_coded` bit, whereas a filter always has
`coef_coded`.

### 5.3.7 arithmetic_code

All `fs` bits after the header are the arithmetic code `cb[0] .. cb[fs-1]`, in frame order.

1. If `fs > 0`, `cb[0]` MUST be 0; otherwise the frame is **invalid**.
2. `cb[1]` onwards is the codeword decoded by 5.5.7.
3. For decoding, every code bit at a position `>= fs` reads as 0. A frame whose codeword was
   cut short therefore still decodes (to whatever those zero bits give); this is not an error.

### 5.3.8 Rice code

A Rice-coded signed integer with parameter m is read as:

1. **Unary part:** count `q`, the number of 0 bits before the next 1 bit; the 1 is consumed.
2. **Low part:** `lsb` = uimsbf(m) (nothing if m = 0).
3. **Magnitude:** `u = q x 2^m + lsb`.
4. **Sign:** if `u != 0`, one more bit: 1 means the value is `-u`, 0 means `+u`. If `u = 0`
   no sign bit is read and the value is 0.

If the frame ends before the terminating 1 of the unary part, or inside the low part or the
sign bit, the frame is **invalid**. (Rice codes appear only in the header, so this is a case
of 5.2.1 item 3.)

## 5.4 Derived per-bit assignments

For channel c and bit index n (0 to `NB` - 1):

1. `F(c, n)` is the filter number mapped to the filter segment of channel c that contains n.
2. `P(c, n)` is the ptable number mapped to the ptable segment of channel c that contains n.
3. `NH[c]`, the half-probability length of channel c, is `N_(F(c, 0))` (the prediction order
   of the filter of channel c's **first** filter segment) if `half_prob[c]` = 1, and 0
   otherwise. It does not depend on the filter in force at bit n.

The two segmentations are independent unless `same_segmentation` = 1; even then the ptable
numbers may differ from the filter numbers when `same_mapping` = 0.

## 5.5 Decoding process (coded frames)

### 5.5.1 Order of symbols

The arithmetic code is one stream shared by all channels. Symbols are decoded in this order:

1. one symbol, the coded X bit (5.5.8);
2. then for n = 0, 1, ..., `NB` - 1, and within each n for c = 0, 1, ..., `Nch` - 1: one
   symbol for bit n of channel c.

### 5.5.2 Prediction history

Each channel c has a history of its 128 most recently decoded bits, `h_c[0] .. h_c[127]`,
`h_c[0]` being the most recent.

1. **Initial state**, at the start of every frame: `h_c[i] = 1` for odd i and `h_c[i] = 0` for
   even i.
2. **Update**, after the output bit `y` for (c, n) is known: every `h_c[i]` moves to
   `h_c[i + 1]`, `h_c[127]` is dropped, and `h_c[0] = y`.
3. The history continues across segment boundaries: a change of filter or ptable does not
   reset it.

### 5.5.3 Prediction

With f = F(c, n), N = `N_f` and `a_i` its coefficients:

    S = sum over i = 0 .. N-1 of  a_i x (2 x h_c[i] - 1)

(a history bit of 1 contributes `+a_i`, a history bit of 0 contributes `-a_i`). S is then
reduced to a 16-bit two's-complement value:

    Spred = ((S + 32768) mod 65536) - 32768

S lies in -32768 to +32768, so the reduction changes only S = +32768, which becomes -32768.

The **sign indicator** is `s = 1` if `Spred < 0`, else `s = 0`.

### 5.5.4 Probability

1. If `n < NH[c]`, the probability is `p = 128`.
2. Otherwise, with q = P(c, n) and its length `M_q`:

       j = min( floor(|Spred| / 8), M_q - 1 )
       p = P_q[j]

`p` is an integer 1 to 128 on a scale of 256: the probability that the coded symbol is 0.

### 5.5.5 Half-probability region

The region `n < NH[c]` lies inside channel c's first filter segment (segments are at least
1024 bits and orders at most 128). It may span several ptable segments and overrides all of
them. Prediction and history updates still happen for these bits; only the probability is
fixed.

### 5.5.6 Output bit

The decoded symbol is `e` (0 or 1). The output bit is `y = s XOR e`. It updates the history
(5.5.2) and is written to the output (5.5.9).

### 5.5.7 Arithmetic decoder

State: `A`, the interval width (12 bits); `C`, the code value (12 bits); `ptr`, the index of
the next code bit to read. A code bit at index `>= fs` reads as 0.

**Initialisation:** `A = 4095`; `C` = the 12 bits `cb[1] .. cb[12]` as an MSB-first integer;
`ptr = 13`.

**Decoding one symbol with probability p (1 to 128):**

1. `Q = floor(A / 256)`, then if bit 7 of A is 1, set bit 0 of Q to 1. Equivalently
   `Q = floor(A / 256) OR (floor(A / 128) AND 1)`.
2. `w0 = Q x p`, the width of symbol 0's sub-interval.
3. `w1 = A - w0`, the width of symbol 1's sub-interval.
4. If `C >= w1`: the symbol is `e = 0`, then `C = C - w1` and `A = w0`.
   Otherwise: the symbol is `e = 1`, then `A = w1` (C unchanged).
5. **Renormalise:** while `A < 2048`: `A = 2 x A`, `C = 2 x C + cb[ptr]`, `ptr = ptr + 1`.

Before every symbol `A >= 2048`, so Q is 8 to 15, `w0` is 8 to 1920 and `w1` is at least 128:
both sub-intervals are non-empty, and A and C stay within 12 bits.

**End of frame:** after the last symbol, the number of unread code bits is `fs - ptr`. If it
is greater than 7 (`ptr < fs - 7`) the frame is **invalid**. Unread bits may have any value.

### 5.5.8 Coded X bit

Immediately after initialisation the decoder decodes one symbol with probability

    p_X = bitrev7(a_0 mod 128) + 1

where `a_0` is coefficient 0 of filter 0 taken as its 9-bit two's-complement pattern (so
`a_0 mod 128` is its low 7 bits) and `bitrev7` reverses the order of 7 bits (1 becomes 64,
2 becomes 32, 127 stays 127). `p_X` is 1 to 128. The decoded value is discarded; it has no
meaning on SACD. It still moves the decoder state, so it MUST be decoded.

### 5.5.9 Output packing

The output is `FL x Nch` bytes. Bit n of channel c goes to byte `floor(n / 8) x Nch + c`, at bit
position `7 - (n mod 8)` (bit 7 is the MSB). This is the interleaved layout of
specification 0, 0.6. A plain frame's payload uses the same layout and is copied unchanged.

## 5.6 Validity rules

A frame is **invalid** exactly when one of these holds. A decoder MUST detect each of them and
MUST NOT report any other frame as invalid.

| No. | Code | Condition | Section |
|---|---|---|---|
| 1 | frame-too-short | `FB <= 1` | 5.2.1 |
| 2 | stuffing | plain frame with non-zero stuffing bits | 5.2 |
| 3 | header-truncated | the header (Rice codes included) needs more bits than the frame has | 5.2.1, 5.3.8 |
| 4 | resolution | `R = 0` or `R > FL - Lmin/8` | 5.3.2 |
| 5 | segment-length | an explicit segment shorter than `Lmin` bits, or longer than `rem` | 5.3.2 |
| 6 | too-many-segments | an `end_of_channel_segm` = 0 beyond the allowed number of explicit lengths | 5.3.2 |
| 7 | table-number | a `table_index` greater than T | 5.3.3 |
| 8 | too-many-tables | final `NF` or `NP` greater than `2 x Nch` | 5.3.3 |
| 9 | channel-mapping | `same_map_all_ch` = 1 with unequal segment counts | 5.3.3 |
| 10 | segment-count-mismatch | `same_mapping` = 1 with different filter and ptable segment counts for a channel | 5.3.3 |
| 11 | coefficient-coding | coded filter with method 3 or `K >= N_f` | 5.3.5 |
| 12 | coefficient-range | reconstructed coefficient outside -256 to 255 | 5.3.5 |
| 13 | ptable-coding | coded ptable with method 3 or `K >= M_p` | 5.3.6 |
| 14 | ptable-range | reconstructed entry outside 1 to 128 | 5.3.6 |
| 15 | arithmetic-code | `fs > 0` and `cb[0] = 1` | 5.3.7 |
| 16 | unread-code-bits | more than 7 code bits left unread at the end | 5.5.7 |

The codes in this table are names for the conditions; a decoder MAY report them, but the
corpus checks only that a frame is invalid. When several conditions hold, which one is
reported does not matter.

Not checked (MUST NOT make a frame invalid): the value of `plain_X_bit` and of the coded X
bit; the Rice parameter's value; the values of unread code bits; extra bytes after a plain
frame's payload; a coded frame's code being longer than `NB x Nch` bits.

## 5.7 Constants and limits

| Item | Value |
|---|---|
| Channels `Nch` | 1 to 6 |
| `NB`, bits per channel per frame | 37 632 |
| `FL`, bytes per channel per frame | 4704 |
| Plain frame length | `1 + FL x Nch` bytes (shorter: zero-filled; longer: extra ignored) |
| Most filters, `NF` | `2 x Nch` |
| Most ptables, `NP` | `2 x Nch` |
| Filter segments per channel | at most 5 (shared) or 4 (per channel); each at least 1024 bits |
| Ptable segments per channel | at most 8; each at least 32 bits |
| Segment resolution `R` | 1 to 4576 bytes (filters), 1 to 4700 bytes (ptables); field 13 bits |
| Prediction order | 1 to 128 |
| Filter coefficient | -256 to 255 |
| Ptable length | 1 to 64 |
| Ptable entry | 1 to 128 |
| Coding methods | 0 to 2 (3 invalid) |
| Rice parameter | 0 to 7 (not checked) |
| Arithmetic registers | 12 bits; renormalise while `A < 2048` |
| Unread code bits allowed | at most 7 |
| Frame duration | 1/75 s |

Informative: the frame length on SACD is bounded by the reassembly limit of 65 536 bytes
(specification 4); a decoder MUST nevertheless not depend on that bound for memory safety.

## 5.8 Informative notes on interoperability

1. Other decoders in circulation differ on frames that no correct encoder produces. A stream
   that follows annex 5.A decodes identically in all of them. In particular, encoders SHOULD
   keep the arithmetic code no longer than `NB x Nch` bits (falling back to a plain frame),
   SHOULD NOT use a ninth ptable segment, and SHOULD NOT append bytes after the codeword.
2. Two to three extra zero bytes after a valid coded frame illustrate rule 16: because the
   decoder's 12-bit window reads 11 bits past the last codeword bit, one appended zero byte is
   always accepted, two are accepted only when the original padding was at most 2 bits, and
   three are always rejected. Removing trailing zero bytes of the codeword is always accepted.
3. Real SACD streams use few of the optional features; the corpus exercises all of them.

## Annex 5.A (informative): producing valid frames

This annex states what an encoder must produce for a frame to be valid under 5.6, and gives an
arithmetic encoder that inverts 5.5.7 exactly. It is useful for generating test data.

### 5.A.1 Frame-level choices

1. **Plain frame:** the byte 0x00 followed by the `FL x Nch` DSD bytes in interleaved layout.
   An encoder uses it whenever the coded frame would be larger.
2. **Simplest coded frame:** `same_segmentation` = 1, `same_segm_all_ch` = 1,
   `end_of_channel_segm` = 1 (one segment per channel, no resolution field), then the mapping.

### 5.A.2 Segmentation

1. Choose `R` with `1 <= R <= FL - Lmin/8`; every explicit segment is a multiple of `8 x R`
   bits. Send `R` once per transmitted segmentation, just before its first explicit length;
   with `same_segmentation` = 1 the ptables reuse the filter segmentation.
2. Each explicit length must satisfy `8 x R x L >= Lmin`, and the remainder of the channel
   must stay at least `Lmin` bits. The last segment is implicit.
3. Write each length in `bitlen(floor(rem / R))` bits.
4. Keep within the explicit-length limits of 5.3.2.

### 5.A.3 Mapping

Number tables in order of first use (channel 0's segments, then channel 1's, ...), write each
index in `bitlen(T)` bits, use at most `2 x Nch` tables of each kind, and send every table
number from 0 to T-1.

### 5.A.4 Filters and ptables

1. Coded filters require `k + 1 < N`; the residual is `r_i = a_i + floor((x_i + 4) / 8)` with
   `x_i` computed from the true coefficients.
2. Coded ptables require `k + 1 < M`; the residual is `r_i = P[i] + floor((x_i + 4) / 8)`.
3. A one-entry ptable sends only its 6-bit length (value 0) and always has probability 128.
4. Rice parameters up to 6 (filters) and 4 (ptables) are customary.
5. Rice encoding of r with parameter m: `u = |r|`; write `floor(u / 2^m)` zero bits and a
   1; write `u mod 2^m` in m bits; if `r != 0` write 1 for negative or 0 for positive.
6. Ptable entries approximating 256 times the probability that the output bit equals the sign
   indicator give the best compression; any value 1 to 128 is decodable.

### 5.A.5 Symbols

Run the same history, prediction and probability selection as the decoder, in the order of
5.5.1. Encode 0 for the coded X bit (any value decodes), then `e = y XOR s` for each output
bit y.

### 5.A.6 Arithmetic encoder

State: `Lo`, a non-negative integer of unbounded precision; `A`, 12 bits; `K`, the number of
renormalisation shifts so far. Start with `Lo = 0`, `A = 4095`, `K = 0`.

To encode symbol e with probability p: compute Q, `w0` and `w1` from A as in 5.5.7. If e = 1
set `A = w1`. If e = 0 set `Lo = Lo + w1` and `A = w0`. Then while `A < 2048`:
`A = 2 x A`, `Lo = 2 x Lo`, `K = K + 1`.

The decoder's register satisfies `C = floor(V x 2^K) - Lo` with
`V = sum over i >= 1 of cb[i] x 2^(12 - i)`; intervals `[Lo, Lo + A) / 2^K` are nested
starting from `[0, 4095)`, so a final V in the last interval decodes every symbol. Adding `w1`
can carry into bits already produced; propagate the carry towards `cb[1]` (it never reaches
`cb[0]`, because `Lo < 2^(12 + K)`).

Termination: after the last symbol, with interval `[Lo, Lo + A)` and `A >= 2048`, choose
`Mv = 2^11 x ceil(Lo / 2^11)`, which lies in the interval. The codeword is `cb[0] = 0` followed
by the `K + 1` bits of `floor(Mv / 2^11)`, MSB first. Its length is `K + 2` bits.

Padding: with H header bits and a codeword of n bits, choose `FB = ceil((H + n) / 8)` and fill
the remaining `8 x FB - H - n` bits with 0. The decoder ends with `ptr = 13 + K` and the frame
has `fs <= K + 9` code bits, so rule 16 is satisfied. If H already fills the frame, still emit
at least `cb[0]`.
