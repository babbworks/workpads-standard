# §5 — Codec & Compact Encoding

**Status:** v1.2 (codebook b — financial block — 2026-05-05)
**Bitpads Registry ID:** `iHX7-3F9KQ2`
**Algorithm:** pads-v1, codebook b
**Compression:** fflate deflateSync (DEFLATE, RFC 1951)
**URL encoding:** base64url (no padding)

---

## Purpose

The codec converts a workpad record to a compact string suitable for use as a URL hash fragment. This allows a record to be shared as a single URL — via SMS, QR code, clipboard, or any link transport — and decoded by any Workpads client without server involvement.

The encoded string is placed after the `#` character in the URL, making it a hash fragment. Hash fragments are not sent to servers by browsers, ensuring the record data remains client-side during transmission.

---

## URL Structure

```
https://workpads.me/p#1bg/<base64url>&c=<chainRef>
```

| Component | Value | Description |
|-----------|-------|-------------|
| Origin | `https://workpads.me` | Canonical base URL for Workpads share links |
| Path | `/p` | Workpad receiver path |
| Scheme tag | `1bg` | 3-character encoding descriptor (see below) |
| `/` | literal | Separator between scheme tag and payload |
| `<base64url>` | variable | The compressed, base64url-encoded binary frame |
| `&c=<chainRef>` | optional | 4-char base64url chain reference (3 random bytes) |

### Scheme Tag

The scheme tag is always 3 ASCII characters:

| Position | Character | Meaning |
|----------|-----------|---------|
| 0 | `1` | pads format version 1 |
| 1 | `b` | codebook b (svc-basic + financial block) |
| 2 | `g` | compression: DEFLATE via fflate |

Detection: a hash fragment matching `/^[0-9][a-z][a-z]\//` is a pads-v1 URL. Extract the payload as `hash.slice(4)`. Read the 3-char tag to determine the codebook (see §Backwards Compatibility).

---

## Encoding Steps

Given a valid record object:

### Step 1 — Assemble the binary frame

Construct a byte array following the pads-v1 frame format (see below).

### Step 2 — Compress

Apply DEFLATE compression using `fflate.deflateSync`:

```js
var compressed = fflate.deflateSync(frameBytes, { level: 9 });
```

### Step 3 — base64url encode

Convert the compressed bytes to a base64url string:

```
base64url = base64(bytes)
              .replace(/\+/g, '-')
              .replace(/\//g, '_')
              .replace(/=+$/, '')
```

Standard base64 uses `+`, `/`, and `=` which are not URL-safe. base64url replaces them with `-`, `_`, and no padding.

### Step 4 — Produce hash fragment

```
1bg/<base64url>
```

This is the value placed after `#` in the share URL. Append `&c=<chainRef>` if a chain reference is present.

---

## Decoding Steps

Given a hash fragment string:

1. Check `/^[0-9][a-z][a-z]\//.test(hash)` — reject if not matched
2. Extract payload: `hash.slice(4)` (skip the 3-char tag and `/`)
3. Reverse base64url: add `=` padding, replace `-`→`+`, `_`→`/`, then base64-decode to bytes
4. Decompress: `fflate.inflateSync(bytes)`
5. Parse the binary frame (see below) to reconstruct the record object

---

## pads-v1 Binary Frame Format

### Frame Header (3 bytes)

```
Byte 0:    Template identifier (uint8)
Bytes 1–2: Presence flags (uint16, big-endian)
```

**Template identifiers:**

| Byte | Template |
|------|----------|
| `0x01` | `svc-basic` v1 |
| `0x02`–`0xFF` | Reserved for future templates |

### Presence Flags — 16-bit Big-Endian Uint16

The presence flags occupy **bytes 1 and 2** as a single 16-bit big-endian unsigned integer.

```js
flags = (frame[1] << 8) | frame[2]
```

**Big-endian: byte 1 is the high byte (bits 8–15), byte 2 is the low byte (bits 0–7).**

> ⚠️ **Critical implementation note:** Do not read the flags as little-endian (`frame[1] | (frame[2] << 8)`). This byte-swap silently produces wrong flag values — fields decode as the wrong type, byte offsets cascade incorrectly, and the record appears corrupt or partially empty. Always read as `(frame[1] << 8) | frame[2]`.

A bit is `1` if the corresponding field has a value and its blob is included in the frame. A bit is `0` if the field is absent and no blob follows.

### Field Slot Assignments (svc-basic, pads-v1 codebook b)

| Bit | Flag mask | Field key | Type |
|-----|-----------|-----------|------|
| 0 | `0x0001` | `job` | scalar |
| 1 | `0x0002` | `customer` | scalar |
| 2 | `0x0004` | `date` | scalar |
| 3 | `0x0008` | `location` | scalar |
| 4 | `0x0010` | `meeting_time` | scalar |
| 5 | `0x0020` | `start_time` | scalar |
| 6 | `0x0040` | `end_time` | scalar |
| 7 | `0x0080` | `customer_phone` | scalar |
| 8 | `0x0100` | `worker` | scalar |
| 9 | `0x0200` | `actions` | actions blob |
| 10 | `0x0400` | `details` | scalar |
| 11 | `0x0800` | `story` | scalar |
| 12 | `0x1000` | financial block | FIN flag — structured binary block (see §Financial Block) |
| 13–15 | — | — | reserved for future use |

Bits 0–8 map to scalar fields appearing before the actions blob. Bits 10–11 map to scalar fields appearing after the actions blob. Bit 9 signals the actions blob. Bit 12 gates the financial block, which follows the high scalars.

**Chain reference:** Appended to the URL as a separate parameter:

```
1bg/<base64url>&c=<chainRef>
```

The chain reference is 4 base64url characters (3 random bytes). It links related records (e.g., a quote and its approval). Strip `&c=<chainRef>` before passing the payload to the frame decoder. Store `_chainRef` on decoded records for chain lookups.

### Frame Body Layout

```
[low scalars: bits 0–8, in ascending bit order]
[actions blob, if bit 9 set]
[high scalars: bits 10–11, in ascending bit order]
[financial block, if bit 12 set]
```

### Scalar Field Blobs

For each scalar field with its bit set:

```
[length_high_byte][length_low_byte][utf8_bytes...]
```

Length is a **2-byte big-endian uint16** giving the byte count of the UTF-8 encoded value. Maximum: 65,535 bytes.

**Example:** `"Replace faucet"` (14 bytes):

```
0x00 0x0E  52 65 70 6C 61 63 65 20 66 61 75 63 65 74
```

### Actions Blob

When bit 9 is set, the actions sequence follows all low scalar blobs:

```
[action_count_byte]
  [title_length_high][title_length_low][title_utf8...]
  [notes_length_high][notes_length_low][notes_utf8...]
  ... (repeated for each action)
```

- `action_count_byte`: number of actions, 0–20 (uint8)
- Each action: title blob then notes blob, both 2-byte big-endian length-prefixed
- `notes` may have length 0; the `0x00 0x00` length prefix is still present

### Financial Block (bit 12)

When bit 12 is set, a structured financial block follows all high scalar blobs. All fields within the block are optional, gated by a single `fin_flags` byte.

```
[uint8 fin_flags]
  if fin_flags & 0x01:  record_type  → uint8 enum (see below)
  if fin_flags & 0x02:  currency     → uint8 enum (see below)
  if fin_flags & 0x04:  vat          → uint8 enum (see below)
  if fin_flags & 0x08:  amount       → [uint16 len][UTF-8 bytes]
  if fin_flags & 0x10:  expenses     → [uint8 count] + expense items
  if fin_flags & 0x20:  payments     → [uint8 count] + payment items
```

**Enum encoding:** A known value is written as a single byte (0-indexed position in the enum array). Unknown/custom values use byte `0xFF` followed by `[uint16 len][UTF-8 bytes]`.

| Field | Enum values (index order) |
|-------|--------------------------|
| `record_type` | `quote`(0), `invoice`(1), `expense`(2), `payment`(3) |
| `currency` | `GBP`(0), `USD`(1), `EUR`(2), `CAD`(3), `AUD`(4), `NZD`(5), `ZAR`(6) |
| `vat` | `0`(0), `5`(1), `7.5`(2), `10`(3), `12.5`(4), `15`(5), `20`(6), `23`(7), `25`(8) |

**Expense item:**

```
[uint8 item_flags]
[uint16][UTF-8 amount]            — always present
[uint16][UTF-8 job]               — if item_flags & 0x01
[uint16][UTF-8 date]              — if item_flags & 0x02
[uint8 billing_enum]              — if item_flags & 0x04
[uint8 actionIdx]                 — if item_flags & 0x08
(is_viewer: item_flags & 0x10)    — flag only, no extra bytes
```

Billing enum: `billable`(0), `non-billable`(1), `absorbed`(2), `cogs`(3).

**Payment item:**

```
[uint8 item_flags]
[uint16][UTF-8 amount]            — always present
[uint16][UTF-8 job]               — if item_flags & 0x01
[uint16][UTF-8 date]              — if item_flags & 0x02
```

**Rationale:** Integrating financial data into the main frame places all record content in a single deflate context, allowing LZ77 to deduplicate repeated strings across the record and its line items. Enum bytes reduce high-frequency strings (`currency`, `vat`, `record_type`) to a single byte each.

---

## Worked Example

Record:

```json
{
  "job": "Fix tap",
  "customer": "Alice",
  "actions": [{ "title": "Arrive", "notes": "" }]
}
```

**Presence flags:**

- bit 0 (`job`): `0x0001`
- bit 1 (`customer`): `0x0002`
- bit 9 (`actions`): `0x0200`

`flags = 0x0203`  → byte 1 = `0x02`, byte 2 = `0x03`

**Frame bytes (before compression):**

```
0x01              template = svc-basic
0x02 0x03         flags big-endian: 0x0203
0x00 0x07         job length = 7
46 69 78 20 74 61 70   "Fix tap"
0x00 0x05         customer length = 5
41 6C 69 63 65    "Alice"
0x01              action count = 1
0x00 0x06         title length = 6
41 72 72 69 76 65 "Arrive"
0x00 0x00         notes length = 0
```

Compressed with `deflateSync` → base64url → prepend `1bg/` → append to `https://workpads.me/p#`.

---

## Byte Order Summary

All multi-byte integers in the pads-v1 frame are **big-endian** (most significant byte first):

| Field | Size | Encoding |
|-------|------|----------|
| Presence flags | 2 bytes | big-endian uint16 |
| Scalar field length | 2 bytes | big-endian uint16 |
| Action title length | 2 bytes | big-endian uint16 |
| Action notes length | 2 bytes | big-endian uint16 |

Correct flag read: `flags = (frame[1] << 8) | frame[2]`
Incorrect (little-endian): `flags = frame[1] | (frame[2] << 8)` — **do not use**

---

## Size Characteristics

| Record content | Approx. hash fragment length |
|----------------|------------------------------|
| Job only | ~60 chars |
| Job + customer + date + location | ~100 chars |
| Full record (all fields, 2 actions) | ~180–220 chars |
| Full record (all fields, 5 actions) | ~260–300 chars |

Well within SMS message limits and QR code capacity.

---

## Implementations

| Implementation | Location | Environment |
|----------------|----------|-------------|
| Browser codec | `workpadsdotme/js/lib/codec.js` — `window.WPCodec` | Browser (fflate UMD) |
| Receiver inline | `workpadsdotme/p/index.html` — inline | Browser, no dependencies |
| Customer inline | `workpadsdotme/p/customer.html` — inline | Browser, no dependencies |
| npm package | `workpads-codec/` | Node + browser (codebook b pending v0.2) |

All inline copies must stay byte-for-byte identical. A URL encoded by any implementation must decode correctly in any other.

## Backwards Compatibility

Codebook-a URLs (`1ag/`) remain valid. Decoders must check the scheme tag character at position 1 and route accordingly:

```js
var tag = hash.slice(0, 3);
var record = (tag === '1ag') ? padsDecodeA(frame) : padsDecodeB(frame);
```

**Codebook-a field slots (bits 12–15):** In codebook a, bits 12–15 were plain scalar fields (`amount`, `currency`, `vat`, `record_type`), encoded as `[uint16][UTF-8]` strings like all other scalars. No financial block. No line items.

New encoders must emit `1bg/`. Existing `1ag/` URLs continue to decode indefinitely.

---

## Codec Identity

This codec is registered in the bitpads registry as:

```
iHX7-3F9KQ2
```

Use this identifier when referencing the codec externally (documentation, interoperability notes, registry queries). The identifier refers to the complete encoding scheme: pads-v1 binary frame + DEFLATE + base64url, covering both codebook a (`1ag/`) and codebook b (`1bg/`).

**Do not use** `bitpad-v1` as an external identifier — that name belongs to the earlier legacy format (`alg=bitpad-v1&d=...`), which is a distinct encoding registered separately.

---

## URL Format History

| Era | Hash format | Detection |
|-----|-------------|-----------|
| Legacy (pre-v1) | `alg=bitpad-v1&v=1&d=<base64url>` | `alg=bitpad-v1` substring |
| pads-v1 codebook a | `1ag/<base64url>` | tag char[1] = `a` |
| pads-v1 codebook b (current) | `1bg/<base64url>` | tag char[1] = `b` |

The `[0-9][a-z][a-z]/` regex detects any pads-v1 URL regardless of codebook. Read char[1] to determine codebook and route to the appropriate decoder. Legacy (`alg=bitpad-v1`) URLs are not cross-compatible and should be rejected by v1 decoders.
