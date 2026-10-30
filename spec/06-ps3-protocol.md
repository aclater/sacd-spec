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

1. TCP. The server listens on port **2002** by default. It accepts one client at a time
   (listen backlog 1).
2. Each request and each response is one **message** (6.3), followed immediately by a single
   byte 0x00. There is no length prefix: the 0x00 byte, which cannot start a field, ends the
   message. Both directions use this framing.
3. A client sends one request and reads its complete response before sending the next.
4. The connection stays open across requests. After responding to a close request (6.4) the
   server closes the connection.

## 6.3 Message encoding

Messages use a subset of the Protocol Buffers (proto2) binary encoding.

### 6.3.1 Varints

A **varint** encodes an unsigned integer in groups of 7 bits, least significant group first.
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
marker of 6.2. This protocol's messages use wire types 0 and 2 only. A receiver MUST be able
to skip fields of wire types 1 and 5, MUST treat any other wire type as a protocol error, and
MUST ignore field numbers it does not know.

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

A field that is absent has its default value: 0 for result, empty for data.

## 6.4 Requests and responses

| Request | Response type | Result | Data | Server action |
|---|---|---|---|---|
| open disc | opened (1) | 0 on success; -1 if there is no disc or it cannot be opened | none | opens the drive, reads the disc's TOC (to learn the audio areas, 6.5) and prepares decryption |
| get disc size | size (4) | the disc's total number of sectors; -1 if no disc is open | none | |
| read sectors (offset `o`, count `n`) | read (3) | the number of sectors returned, `r` (0 to `n`), or -1 on a read error | exactly `r x 2048` bytes: sectors `o` to `o + r - 1` | 6.5 |
| close disc | closed (2) | 0 | none | closes the disc, then the connection |

A client opens the disc first. Requests other than open before a successful open fail with
result -1 or are answered with no data.

## 6.5 Read sizes

The server returns **at most 512 sectors per read**, and may return fewer than requested. It
also never returns, in one response, sectors from both sides of an audio area boundary: it
decrypts each audio area separately.

The server keeps a **block size**, an integer that persists between requests on one connection
and starts at 0 when the connection opens. Let `[s1, e1]` be the audio LSN range (first and last
audio LSN, specification 3) of the first area the server found when it opened the disc, and
`[s2, e2]` that of the second; on a single-area disc `s2 = e2 = 0`. For a read at offset `o`
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
   returns `r = 0`, forever. A client MUST therefore make a read that matches a row (for
   example one sector at LSN 0, or the master TOC at LSN 510) as its first read on every
   connection, before reading anything after the audio area.
3. A result of 0 sectors for a read that should have data is not progress. A client MUST NOT
   loop on it without bound (6.7).

When an encrypted range is first read, the server checks whether the disc is protected at all
(some plain DSD discs are not) by examining bytes 16 to 23 of the first block. This affects
only the server's own decryption, not the bytes sent.

## 6.6 Session

A typical client session:

1. Connect to port 2002.
2. Send open disc; require result 0.
3. Send get disc size; require a result of 1 or more.
4. Read the master TOC (LSN 510, 10 sectors). This also sets the block size (6.5 item 2).
5. Read the area TOCs and audio sectors it needs, looping on short reads.
6. Send close disc; the server closes the connection.

A client that resumes after a dropped connection repeats steps 1 to 4 and SHOULD verify that
the disc is the same (6.7 item 6) before continuing.

## 6.7 Client requirements for hostile servers

A client MUST treat the server as hostile. In particular:

| No. | Requirement | Value |
|---|---|---|
| 1 | Bound every length-delimited field before allocating for it. A field longer than one full read plus a margin is a protocol error. | 512 x 2048 + 1024 = 1 049 600 bytes |
| 2 | Bound the number of fields in one message. More is a protocol error. | 16 fields |
| 3 | Reject a varint longer than 10 bytes, or a 10-byte varint whose value exceeds 64 bits. | |
| 4 | Require each response's type to match the request; otherwise it is a protocol error. | |
| 5 | For a read of `n` sectors, require a result `r` with `1 <= r <= n` and a data field of exactly `r x 2048` bytes. Anything else (including more sectors than requested, or 0) is an error. | |
| 6 | Bound the disc size the server reports. A larger size is an error; it would let a server make the client create an enormous file. | at most 2^22 sectors (8 GiB), just above a dual-layer disc |
| 7 | Bound the time per response: a deadline covering the whole response, not each receive, so that a server cannot trickle a response a byte at a time. | 60 s RECOMMENDED |
| 8 | Bound reconnection attempts. | 3 in a row RECOMMENDED, with back-off |
| 9 | After a reconnection, check that the disc size and the master TOC sectors are unchanged; if not, the disc changed and the copy MUST fail. | |
| 10 | Apply specifications 1 to 4 and 7 to the sectors received exactly as to an image file. In particular, every LSN taken from a TOC MUST be checked against the reported disc size before it is requested. | |

## 6.8 Server requirements (for test servers)

A server that serves an image file for testing SHOULD:

1. implement 6.5 exactly, including the carried-over block size, so that clients are tested
   against the real behaviour;
2. never read outside the served image, whatever a request says;
3. close connections that stay idle for a bounded time (60 s RECOMMENDED), so one silent client
   cannot hold the single connection forever;
4. listen on the loopback interface unless told otherwise.

## 6.9 Limits

| Item | Value |
|---|---|
| Port | 2002 |
| Sectors per read response | at most 512 |
| Sector size on the wire | 2048 bytes |
| Request fields | 3 |
| Response fields | 2 or 3 |
| Largest legitimate data field | 512 x 2048 = 1 048 576 bytes |
