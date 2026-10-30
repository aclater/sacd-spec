# 6. PS3 SACD Ripper network protocol

This specification defines the network protocol spoken by the "SACD Ripper" homebrew server
running on a PlayStation 3, which reads an SACD in the console's drive and serves its
decrypted logical sectors over TCP. A reader MAY implement a client for it; a reader that does
MUST follow this specification. Conventions are in [00-conventions.md](00-conventions.md).

## 6.1 Overview

1. The server opens the disc, removes the protection of its audio areas, and answers requests
   for ranges of 2048-byte logical sectors. Only plain logical sectors cross the network, so
   the disc as seen through the protocol is an image in the plain format (specification 1,
   1.2) whose `N` is the size the server reports.
2. A server that serves an already decrypted image file behaves identically on the wire; this
   is how the protocol is tested without a console.
3. The protocol has **no authentication and no encryption**. Anyone who can reach the port can
   read the disc, and anyone on the path can read or alter sectors. Clients SHOULD be used on
   trusted networks only and MUST treat every byte received as hostile (6.7).

## 6.2 Transport

1. TCP. The server listens on port **2002** by default. It serves one client at a time.
2. Each request and each response is one **message** (6.3), followed immediately by a single
   byte 0x00. There is no length prefix. Because no field key can be 0 (6.3.2), a 0x00 byte
   **in a key position** ends the message. Other 0x00 bytes occur freely inside field values (a
   sector offset of 0 is sent as the byte 0x00, and sector data contains zeros), so a receiver
   MUST parse the message field by field to find its end; it MUST NOT scan for the first 0x00
   byte. Both directions use this framing.
3. A client sends one request and reads its complete response before sending the next.
4. The connection stays open across requests. After responding to a close request (6.4) the
   server closes the connection.

## 6.3 Message encoding

Messages use a subset of the Protocol Buffers (proto2) binary encoding.

### 6.3.1 Varints

A **varint** encodes an unsigned integer in groups of 7 bits, least significant group first.
A receiver MUST accept non-minimal encodings (extra groups of zero bits) up to the 10-byte
limit, except for the key rule of 6.3.2.
Each byte carries one group in bits 0 to 6; bit 7 is 1 on every byte except the last. A varint
is at most 10 bytes long (64 bits).

A signed 64-bit integer is encoded as the varint of its 64-bit two's-complement pattern, so
every negative value takes 10 bytes. For example, -1 is the 10 bytes FF FF FF FF FF FF FF FF FF
01.

### 6.3.2 Fields

A message is a sequence of fields. Each field starts with a **key**, a varint whose value is
`field number x 8 + wire type`:

| Wire type | Payload |
|---|---|
| 0 | a varint |
| 1 | 8 bytes, little-endian |
| 2 | a varint length `n`, then `n` bytes |
| 5 | 4 bytes, little-endian |

Field numbers are 1 or more, so no key is 0; a key byte of 0x00 is therefore the end-of-message
marker of 6.2. Any other key whose field number is 0 (a key of value 1 to 7, or a value-0 key
written in more than one byte, such as 0x80 0x00) is a protocol error. This protocol's messages use wire types 0 and 2 only. A receiver:

1. MUST treat wire types 3, 4, 6 and 7 as a protocol error, in any field;
2. for a field number defined for this message direction (6.3.3 for requests, 6.3.4 for
   responses), MUST treat a wire type other than the one given there as a protocol error (for
   example, in a response, field 3 sent as a varint, or field 2 sent with wire type 1 or 5);
3. for a field number not defined for this message direction, MUST read and skip its payload
   (wire types 0, 1, 2 or 5); it counts towards the limits of 6.7;
4. when a defined field occurs more than once in one message, MUST use its last occurrence
   (the proto2 rule).

### 6.3.3 Request

| Field number | Wire type | Name | Type | Meaning |
|---|---|---|---|---|
| 1 | 0 | request type | enumeration | see table below; always sent |
| 2 | 0 | sector offset | unsigned 32-bit | first LSN; always sent (0 when unused) |
| 3 | 0 | sector count | unsigned 32-bit | number of sectors; always sent (0 when unused) |

| Request type value | Name |
|---|---|
| 1 | open disc |
| 2 | close disc |
| 3 | read sectors |
| 4 | get disc size |

### 6.3.4 Response

| Field number | Wire type | Name | Type | Meaning |
|---|---|---|---|---|
| 1 | 0 | response type | enumeration | same numbering as the request types: 1 opened, 2 closed, 3 read, 4 size |
| 2 | 0 | result | signed 64-bit | see 6.4; negative values are 10-byte varints |
| 3 | 2 | data | bytes | read responses only: the sectors read |

A field that is absent has its default value: 0 for result, empty for data. A response without
a response type field is a protocol error. A data field in a response other than a read
response is ignored.

## 6.4 Requests and responses

| Request | Response type | Result | Data | Server action |
|---|---|---|---|---|
| open disc | opened (1) | 0 on success; -1 if there is no disc or it cannot be opened | none | opens the drive, reads the disc's TOC (to learn the audio areas, 6.5) and prepares decryption |
| get disc size | size (4) | the disc's total number of sectors; -1 if no disc is open | none | |
| read sectors (offset `o`, count `n`) | read (3) | the number of sectors returned, `r` (0 to `n`), or -1 on a read error | exactly `r x 2048` bytes: sectors `o` to `o + r - 1` | 6.5 |
| close disc | closed (2) | 0 | none | closes the disc, then the connection |

A client opens the disc first. Before a successful open, the server answers get disc size and
read sectors with result -1 and no data, and close disc with result 0.

## 6.5 Read sizes

The server returns **at most 512 sectors per read**, and may return fewer than requested. It
also never returns, in one response, sectors from both sides of an audio area boundary: it
decrypts each audio area separately.

The server keeps a **block size**, an integer that persists between requests on one connection
and starts at 0 when the connection opens. When it opens the disc, the server reads its TOC
(specifications 2 and 3) and takes the areas it yields **in slot order** (the area from the
2-channel slot, then the area from the multichannel slot; a skipped area is left out). Let
`[s1, e1]` be the audio LSN range (first and last audio LSN, specification 3) of the first such
area and `[s2, e2]` that of the second; on a single-area disc `s2 = e2 = 0`. No other relation
between the ranges is assumed: when they overlap or are out of order, the table below still
applies row by row as written. For a read at offset `o`
for `n` sectors, the server first updates the block size by the first row that matches:

| Condition on `o` | New block size |
|---|---|
| `o < s1` | `min(s1 - o, 512)` |
| `s1 <= o <= e1` | `min(e1 + 1 - o, 512)` |
| `e1 < o < s2` | `min(s2 - o, 512)` |
| `s2 <= o <= e2` | `min(e2 + 1 - o, 512)` |
| none of these | unchanged |

It then returns `r = min(n, block size)` sectors (also limited by the end of the disc).

Consequences that clients MUST handle:

1. A read can return fewer sectors than requested. A client that needs `n` sectors MUST
   repeat the request for the remainder until it has them all.
2. On a single-area disc, an offset after the audio area matches no row, so the block size
   is whatever the previous request left. On a new connection that is 0, and every such read
   returns `r = 0`, forever. A client MUST therefore make, as its first read on every
   connection, a read of one sector at **LSN 0**. LSN 0 always matches a row: the first row if
   `s1 > 0`, the second if `s1 = 0`. (A read at LSN 510 does not always match: on a disc whose
   only audio area ends before LSN 510 it matches no row.)
3. A client that follows item 2 never receives `r = 0` from a correct server. A result of 0 is
   therefore a protocol error (6.7 item 5).

Some plain DSD discs have no protection; the server detects this itself. It does not change
the bytes sent.

## 6.6 Session

A typical client session:

1. Connect to port 2002.
2. Send open disc; require result 0.
3. Send get disc size; require a result of 1 or more.
4. Read one sector at LSN 0 (this sets the block size, 6.5 item 2), then the master TOC
   (LSN 510, 10 sectors).
5. Read the area TOCs and audio sectors it needs, looping on short reads.
6. Send close disc; the server closes the connection.

A client that resumes after a dropped connection repeats steps 1 to 4 and MUST verify that
the disc is the same (6.7 item 9) before continuing.

## 6.7 Client requirements for hostile servers

A client MUST treat the server as hostile. In particular:

| No. | Requirement | Value |
|---|---|---|
| 1 | Bound every length-delimited field before allocating for it. A field longer than one full read plus a margin is a protocol error. | 512 x 2048 + 1024 = 1 049 600 bytes |
| 2 | Bound the number of fields in one message, counting every field read, skipped ones included. More is a protocol error. | 16 fields |
| 3 | Reject a varint longer than 10 bytes, or a 10-byte varint whose value exceeds 64 bits. | |
| 4 | Require each response's type to match the request; otherwise it is a protocol error. | |
| 5 | For a read of `n` sectors, require a result `r` with `1 <= r <= n` and a data field of exactly `r x 2048` bytes. Anything else (including more sectors than requested, or 0) is an error. | |
| 6 | Bound the disc size the server reports. A larger size is an error; it would let a server make the client create an enormous file. | at most 2^22 sectors (8 GiB), just above a dual-layer disc |
| 7 | Bound the time per response: a deadline that starts when the request has been sent and covers the whole response, not each receive, so that a server cannot trickle a response a byte at a time. | 60 s RECOMMENDED |
| 8 | Bound reconnection attempts. | 3 in a row RECOMMENDED, with back-off |
| 9 | After a reconnection, check that the disc size and the master TOC sectors (LSN 510 to 519) are unchanged; if not, the disc changed and the transfer in progress (the sequence of reads the client was making) MUST fail with `network-disc-changed`. | |
| 10 | Apply specifications 1 to 4 and 7 to the sectors received exactly as to an image file. In particular, every LSN taken from a TOC MUST be checked against the reported disc size before it is requested. | |

### 6.7.1 Error outcomes

Every failure of a client falls into one of these classes. A reader that implements this
specification reports them with these codes (in addition to those of specification 7):

| Code | Condition | Outcome |
|---|---|---|
| `network-connection-lost` | the connection could not be made, was closed by the server (including in the middle of a message), or a response missed its deadline (item 7) | the client MAY reconnect, within the bound of item 8, and resume; when the bound is exhausted, the transfer fails with this code |
| `network-protocol-error` | any violation of items 1 to 5, a malformed message (6.3), a response of the wrong type, or bytes received when no response is outstanding (after a response's terminating 0x00 and before the next request) | the transfer fails; the client MUST NOT reconnect to retry it |
| `network-disc-error` | open disc returned a result other than 0, or get disc size a result below 1, or a read a result below 0 (this row takes precedence over item 5 for negative read results) | the transfer fails; the client MAY report the result value |
| `network-disc-too-large` | the reported disc size exceeds item 6 | the transfer fails |
| `network-disc-changed` | item 9 | the transfer fails |

A **transfer** is the sequence of requests a client makes to obtain the sectors it needs (for
example the whole disc, or the TOCs and one area). When a transfer fails, any sectors already
received MUST NOT be presented as a complete image.

## 6.8 Server requirements (for test servers)

A server that serves an image file for testing SHOULD:

1. implement 6.5 exactly, including the carried-over block size, so that clients are tested
   against the real behaviour;
2. never read outside the served image, whatever a request says;
3. close connections that stay idle for a bounded time (60 s RECOMMENDED), so one silent client
   cannot hold the single connection forever;
4. listen on the loopback interface unless told otherwise;
5. when the served file is a raw 2054- or 2064-byte image, serve its logical sectors (the 2048
   data bytes of each stored sector, specification 1, 1.3) and report its `N` as the disc
   size;
6. answer an open disc for an image from which specifications 1 to 3 yield no area with
   result -1; answer a repeated open by opening the disc again (the block size of 6.5 is kept);
   answer a read with `n = 0`, or at an offset at or past the disc size, with result 0 and no
   data (after updating the block size as 6.5 says); and close the connection on a request it
   cannot decode or whose type is missing or unknown.

## 6.9 Limits

| Item | Value |
|---|---|
| Port | 2002 |
| Sectors per read response | at most 512 |
| Sector size on the wire | 2048 bytes |
| Request fields | 3 |
| Response fields | 2 or 3 |
| Largest legitimate data field | 512 x 2048 = 1 048 576 bytes |
