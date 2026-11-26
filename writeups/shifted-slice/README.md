# Shifted Slice

**Competition:** WolvCTF 2026  
**Category:** Web Exploitation  
**Points:** 134  
**Author:** .ayushmk  
**Handle:** Effectivepawn  
**Status:** Solved

## Overview

Shifted Slice looked like an access-control problem around records stored in an archive. Each record had a position in the archive, and a ticket (also referred to as a slip) authorized reads from a particular byte range.

The part that mattered was what happened when the archive was rearranged.

Rather than trying to forge or tamper with a ticket, I looked at whether I could change the data occupying the range that an already-issued ticket allowed me to read. The ticket stayed valid; the archive layout did not.

That difference was the route to the flag.

## 1. Mapping the archive

I started with the manifest:

```http
GET /api/manifest
```

It listed the records, their sizes, and the layout information. The initial archive contained four records:

| Record | Size |
|:--|--:|
| `m-sealed` | 64 bytes |
| `p-public` | 48 bytes |
| `z-owned` | 64 bytes |
| `zz-scratch` | 128 bytes |
| **Total** | **304 bytes** |

Records were stored in 16-byte-padded frames, so a change to a record or its position could shift everything that followed it.

These were the API operations I focused on:

- `POST /api/tickets`
- `POST /api/records`
- `POST /api/refile`
- `POST /api/compact`
- `POST /api/reset`
- `GET /api/bundle`

The important pieces were record creation, the limited refile operation, compaction, and reading a byte range with a ticket.

## 2. What did the ticket actually authorize?

I requested a ticket for the record I was allowed to access. The ticket covered:

```http
Range: bytes=112-175
```

That is exactly 64 bytes, matching the size of `z-owned` in the initial archive.

At this point, I wanted to distinguish between two possible checks:

- Was the ticket tied to the identity of `z-owned`?
- Or did it only authorize reads from bytes `112–175`, regardless of which record later occupied that space?

I kept the original ticket and session state intact so I could test that distinction.

## 3. Working around the refile restriction

I tested the record-management endpoints to see how much control I had over the layout.

One restriction was clear: `POST /api/refile` accepted the index record `p-public` and a new key. Attempts to rename arbitrary records were rejected.

That ruled out simply renaming `m-sealed` into the desired position. I moved on to the operations that were still available: create a record, change the permitted public key, then compact the archive.

The key was to make compaction do the positioning work.

## 4. Keeping the session and ticket consistent

The two tests needed to start from the same original state.

I issued the ticket first and kept the original session cookie. I did not use `/api/reset` between the tests because reset rotated the session cookie; a fresh session would not be equivalent to the one that issued the ticket.

Instead, I treated the tests as two separate branches derived from the same starting state. Each branch used the same original session identity and the same ticket, but applied a different set of archive changes.

That made the responses comparable.

## 5. First branch: create a known-plaintext reference

For the first branch, I created a record named `q-pad` containing 40 `A` characters.

After compaction, the manifest placed `q-pad` at offset `112`. I then requested the range already authorized by the ticket:

```http
Range: bytes=112-175
```

The server returned:

```http
HTTP/1.1 206 Partial Content
```

The response contained 64 bytes. Since I controlled the content of `q-pad`, this gave me a reference sample for that range.

## 6. Second branch: put the sealed record in the ticket's range

For the second branch, I went back to the same original session state and used the same ticket.

This time I created `a-pad`, again containing 40 `A` characters. I then used the permitted refile operation to change the index record's key:

```text
p-public → b-public
```

After compaction, the manifest showed this layout:

| Record | Offset | Size |
|:--|--:|--:|
| `a-pad` | 0 | 64 bytes |
| `b-public` | 64 | 48 bytes |
| `m-sealed` | 112 | 64 bytes |
| `z-owned` | 176 | 64 bytes |
| `zz-scratch` | 240 | 128 bytes |

The important change was that `m-sealed` now occupied bytes `112–175` — the exact range covered by the ticket issued earlier for `z-owned`.

I sent the same range request with the same ticket. The server again returned:

```http
HTTP/1.1 206 Partial Content
```

The response was 64 bytes long.

The ticket had not changed. The data behind the authorized range had.

## 7. Recovering the sealed data

I now had two 64-byte samples from the same original session and the same ticket range:

- One from the branch where the range contained my controlled `A` data.
- One from the branch where the range contained `m-sealed`.

I compared the samples using the XOR relationship and the known plaintext to derive candidate bytes for the sealed record. I checked possible alignments rather than assuming the flag body started at the beginning of the returned range.

The candidate flag body appeared at offset `17`:

```text
2ad127e3d6d87a29e350f89b89c413a8
```

Using the challenge's `wctf{}` flag format, this produced:

```text
wctf{2ad127e3d6d87a29e350f89b89c413a8}
```

**Flag captured.**

## 8. Why the ticket still worked

The issue was the way authorization was tied to the archive.

The ticket authorized a byte range. Compaction could change which record occupied that range, but the previously issued ticket remained valid. That meant I could move `m-sealed` into a range that the ticket already permitted me to read.

Three things made the chain work:

1. **Predictable layout changes.** Record creation and compaction let me influence record positions.
2. **Ticket validity across compaction.** The ticket continued to authorize the original byte range after the archive layout changed.
3. **A known-plaintext reference.** The first branch gave me controlled data at the same range, which I could compare with the sealed-record response.

The ticket was valid throughout. What changed was the record behind the range it covered.

## Takeaway

This came down to a mismatch between a resource and the location used to identify it.

A byte range describes where data is. It does not identify what that data is.

If a ticket is meant to authorize access to a particular record, the server needs to preserve and validate that relationship when the data is read. Checking only that a request uses the same byte range is not enough when archive operations can move records around.

Compaction and other layout-changing operations need to be part of the access-control model, not treated as unrelated housekeeping.
