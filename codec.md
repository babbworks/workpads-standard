# §5 — Codec & Compact Encoding

**Status:** v1.1 (settled 2026-04-27, financial fields added 2026-04-28)
**Bitpads Registry ID:** `iHX7-3F9KQ2`
**Algorithm:** pads-v1
**Compression:** fflate deflateSync (DEFLATE, RFC 1951)
**URL encoding:** base64url (no padding)

---

## Purpose

The codec converts a workpad record to a compact string suitable for use as a URL hash fragment. This allows a record to be shared as a single URL — via SMS, QR code, clipboard, or any link transport — and decoded by any Workpads client without server involvement.

The encoded string is placed after the `#` character in the URL, making it a hash fragment. Hash fragments are not sent to servers by browsers, ensuring the record data remains client-side during transmission.

---

## URL Structure

```
https://workpads.me/p#1ag/<base64url>
```

| Component | Value | Description |
|-----------|-------|-------------|
| Origin | `https://workpads.me` | Canonical base URL for Workpads share links |
| Path | `/p` | Workpad receiver path |
| Scheme tag | `1ag` | 3-character encoding descriptor (see below) |
| `/` | literal | Separator between scheme tag and payload |
| `<base64url>` | variable | The compressed, base64url-encoded binary frame |

### Scheme Tag

The scheme tag is always 3 ASCII characters:

| Position | Character | Meaning |
|----------|-----------|---------|
| 0 | `1` | pads format version 1 |
| 1 | `a` | codebook A (svc-basic field set) |
| 2 | `g` | compression: DEFLATE via fflate |

Detection: a hash fragment matching `/^[0-9][a-z][a-z]\//` is a pads-v1 URL. Extract the payload as `hash.slice(4)`.

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
1ag/<base64url>
```

This is the value placed after `#` in the share URL.

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

### Field Slot Assignments (svc-basic, pads-v1)

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
| 12 | `0x1000` | `amount` | scalar (decimal string) |
| 13 | `0x2000` | `currency` | scalar (ISO 4217 code: `GBP`, `EUR`, `USD`) |
| 14 | `0x4000` | `vat` | scalar (`none`, `standard`, `zero`, `reduced`) |
| 15 | `0x8000` | `record_type` | scalar (`job`, `quote`, `invoice`, `ack`) |

Bits 0–8 map to scalar fields appearing before the actions blob. Bits 10–15 map to scalar fields appearing after the actions blob. Bit 9 signals the actions blob between the two scalar groups.

**Chain reference:** Financial and relational context is carried by an optional chain reference appended to the URL:

```
1ag/<base64url>&c=<chainRef>
```

The chain reference is 4 base64url characters (3 random bytes). It links related records (e.g., a quote and its approval). Strip `&c=<chainRef>` before passing the payload to the frame decoder. Store `chainRef` on decoded records for chain lookups.

### Frame Body Layout

Fields appear in bit-order (lowest bit first), split into two groups around the actions blob:

```
[low scalars: bits 0–8, in ascending bit order]
[actions blob, if bit 9 set]
[high scalars: bits 10–15, in ascending bit order]
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

Compressed with `deflateSync` → base64url → prepend `1ag/` → append to `https://workpads.me/p#`.

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
| Standalone decoder | `workpadsdotme/p/customer.html` — inline | Browser, no dependencies |

Both produce identical output. A URL encoded by any implementation can be decoded by any other.

---

## Codec Identity

This codec is registered in the bitpads registry as:

```
iHX7-3F9KQ2
```

Use this identifier when referencing the codec externally (documentation, interoperability notes, registry queries). The identifier refers to the complete encoding scheme: pads-v1 binary frame + DEFLATE + base64url + `1ag/` scheme tag.

**Do not use** `bitpad-v1` as an external identifier — that name belongs to the earlier legacy format (`alg=bitpad-v1&d=...`). The current format with scheme tag `1ag/` is a distinct encoding registered separately.

---

## URL Format History

| Era | Hash format | Detection |
|-----|-------------|-----------|
| Legacy (pre-v1) | `alg=bitpad-v1&v=1&d=<base64url>` | `alg=bitpad-v1` substring |
| Current (pads-v1) | `1ag/<base64url>` | `/^[0-9][a-z][a-z]\//` regex |

Legacy and current formats are not cross-compatible. Decoders should detect by regex and handle or reject accordingly.
