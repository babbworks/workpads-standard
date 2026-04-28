# §5 — Codec & Compact Encoding

**Status:** v1 (settled 2026-04-27)
**Algorithm:** bitpad-v1
**Compression:** fflate deflateRaw (raw DEFLATE, RFC 1951)
**URL encoding:** base64url (no padding)

---

## Purpose

The codec converts a workpad record to a compact string suitable for use as a URL hash fragment. This allows a record to be shared as a single URL — via SMS, QR code, clipboard, or any link transport — and decoded by any Workpads client without server involvement.

The encoded string is placed after the `#` character in the URL, making it a hash fragment. Hash fragments are not sent to servers by browsers, which ensures the record data remains client-side during transmission.

---

## URL Structure

```
https://workpads.me/p#alg=bitpad-v1&v=1&d=<base64url>
```

| Component | Value | Description |
|-----------|-------|-------------|
| Origin | `https://workpads.me` | Canonical base URL for Workpads share links |
| Path | `/p` | Workpad receiver path |
| `alg` | `bitpad-v1` | Algorithm identifier — used by receiver to detect and route |
| `v` | `1` | Version of the algorithm parameter encoding |
| `d` | `<base64url>` | The encoded, compressed record data |

The `alg=bitpad-v1` substring in the hash is the detection signal. When any Workpads client loads with a hash containing this string, it attempts to decode the workpad.

---

## Encoding Steps

Given a valid record object:

### Step 1 — Assemble the binary frame

Construct a byte array following the bitpad-v1 frame format (see below).

### Step 2 — Compress

Apply raw DEFLATE compression (no zlib header, no gzip header) using the fflate library:

```js
// Browser (fflate UMD)
fflate.deflateRawSync(frameBytes)

// Node.js (@workpads/codec)
fflate.deflateRawSync(frameBytes)
```

Raw DEFLATE produces the most compact output by omitting the 2-byte zlib header and 4-byte checksum present in zlib-wrapped deflate.

### Step 3 — base64url encode

Convert the compressed bytes to a base64url string:

```
base64url = base64(bytes)
              .replace(/\+/g, '-')
              .replace(/\//g, '_')
              .replace(/=+$/, '')
```

Standard base64 uses `+`, `/`, and `=` which are not URL-safe. base64url replaces them with `-`, `_`, and no padding, producing a string safe for use in URLs without percent-encoding.

### Step 4 — Produce hash fragment

```js
'alg=bitpad-v1&v=1&d=' + base64urlString
```

This is the value placed after `#` in the share URL.

---

## Decoding Steps

Given a hash fragment string:

1. Split on `&` to extract `alg`, `v`, and `d` parameters
2. Verify `alg === 'bitpad-v1'` — reject if unknown
3. Reverse base64url: add `=` padding, replace `-`->`+`, `_`->`/`, then decode from base64 to bytes
4. Decompress using `inflateRaw` (raw DEFLATE inflate)
5. Parse the binary frame (see below) to reconstruct the record object

---

## bitpad-v1 Binary Frame Format

### Frame Header (3 bytes)

```
Byte 0:   Template identifier
Byte 1:   Presence flags — bits 0-7  (fields 0-7)
Byte 2:   Presence flags — bits 8-11 (fields 8-11), upper bits reserved 0
```

**Template identifiers:**

| Byte | Template |
|------|----------|
| `0x01` | `svc-basic` v1 |
| `0x02`-`0xFF` | Reserved for future templates |

**Presence flags** encode which fields are present in the frame. One bit per field slot, in the fixed field order below. A bit is `1` if the field has a value and its blob is included; `0` if the field is absent and no blob follows.

### Field Slot Order (svc-basic, bitpad-v1)

| Bit | Field key | Flag byte |
|-----|-----------|-----------|
| 0 | `job` | byte 1, bit 0 (LSB) |
| 1 | `customer` | byte 1, bit 1 |
| 2 | `date` | byte 1, bit 2 |
| 3 | `location` | byte 1, bit 3 |
| 4 | `start_time` | byte 1, bit 4 |
| 5 | `end_time` | byte 1, bit 5 |
| 6 | `meeting_time` | byte 1, bit 6 |
| 7 | `customer_phone` | byte 1, bit 7 (MSB) |
| 8 | `worker` | byte 2, bit 0 (LSB) |
| 9 | `details` | byte 2, bit 1 |
| 10 | `story` | byte 2, bit 2 |
| 11 | `actions` | byte 2, bit 3 |

Bits 4-7 of byte 2 are reserved and must be `0` in v1 frames.

### Scalar Field Blobs

For each field with its presence bit set (in slot order, low to high):

```
[length_high_byte][length_low_byte][utf8_bytes...]
```

Length is a 2-byte big-endian unsigned integer representing the byte count of the UTF-8 encoded field value. Maximum field length: 65,535 bytes.

**Example:** Field value `"Replace faucet"` (14 bytes UTF-8):
```
0x00 0x0E 52 65 70 6C 61 63 65 20 66 61 75 63 65 74
```

### Actions Blob

When bit 11 is set, the actions sequence immediately follows all scalar blobs:

```
[action_count_byte]
  [title_length_high][title_length_low][title_utf8...]
  [notes_length_high][notes_length_low][notes_utf8...]
  ... (repeated for each action)
```

- `action_count_byte`: number of actions (0-255)
- For each action: one title blob then one notes blob, both 2-byte length-prefixed
- `notes` may have length 0 (zero bytes follow the length prefix)

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

**Step 1 — Presence flags:**
- bit 0 (`job`): 1
- bit 1 (`customer`): 1
- bit 11 (`actions`): 1

Flag byte 1: `0b00000011` = `0x03`
Flag byte 2: `0b00001000` = `0x08`

**Frame bytes (before compression):**
```
0x01                 template = svc-basic
0x03                 flag byte 1: job + customer present
0x08                 flag byte 2: actions present
0x00 0x07            job length = 7
46 69 78 20 74 61 70  "Fix tap"
0x00 0x05            customer length = 5
41 6C 69 63 65        "Alice"
0x01                 action count = 1
0x00 0x06            title length = 6
41 72 72 69 76 65    "Arrive"
0x00 0x00            notes length = 0  (empty)
```

**Step 2 — Compress** with deflateRaw → shorter byte sequence.

**Step 3 — base64url** → compact ASCII string.

**Step 4 — Hash fragment:**
```
alg=bitpad-v1&v=1&d=<compressed-base64url>
```

---

## Size Characteristics

Typical encoded sizes for a `svc-basic` record:

| Record content | Approx. hash fragment length |
|----------------|------------------------------|
| Job only | ~60 chars |
| Job + customer + date + location | ~100 chars |
| Full record (all fields, 2 actions) | ~180-220 chars |
| Full record (all fields, 5 actions) | ~260-300 chars |

These lengths are well within SMS message limits and QR code capacity.

---

## Implementations

| Implementation | Location | Environment |
|----------------|----------|-------------|
| Browser codec | `workpadskaios/js/lib/codec.js` — `window.WPCodec` | Browser (fflate UMD) |
| Node.js package | `workpads-codec/src/` — `@workpads/codec` | Node.js >= 14 |

Both implementations produce identical output. A URL encoded by the browser codec can be decoded by the Node.js package and vice versa.

---

## Comparison with v0.x JSON Encoding (workpads-cli legacy)

The original workpads-cli (pre-bitpad-v1) used a different encoding:

| Property | Legacy JSON encoding | bitpad-v1 |
|----------|---------------------|-----------|
| Payload format | JSON object with compact keys | Binary frame |
| Compression | zlib deflateRaw (same algorithm) | fflate deflateRaw |
| Field keys | Short aliases (`j`, `c`, `d`, ...) | No keys — positional presence flags |
| URL path | `/new#` | `/p#` |
| Detection | No `alg=` param | `alg=bitpad-v1` |
| Typical size | ~15-20% larger | baseline |

Legacy URLs (`/new#...`) and bitpad-v1 URLs (`/p#alg=bitpad-v1&...`) are not cross-compatible. Clients should detect by `alg=` param presence and path.
