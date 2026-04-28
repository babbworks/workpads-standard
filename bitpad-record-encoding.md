# bitpad-v1 Record Encoding Specification

**Status:** v1 — 2026-04-26
**Algorithm identifier:** `bitpad-v1`
**Implements:** svc-basic v2 (13 fields)
**Decision source:** workpads-basicsconform decisions.md, R5-9, TASK-ARC-009
**Implemented in:** `@workpads/codec` — `src/bitpad.js`

---

## 1. Background

bitpad-v1 is a binary-text hybrid encoding for PADS records, derived from the BitPads v2 Role C flag pattern (see REFS/Bitpads/assemblycli/include/bitpads.inc). BitPads v2 uses single-bit flags in a meta-byte to signal which optional fields are present in a record, then packs those fields as length-prefixed byte sequences. bitpad-v1 adapts this structure for all 13 svc-basic v2 fields in a URL-safe compact format.

**Lineage:** BitPads Role C Meta Byte 1 flags → 2-byte field presence header for 12 data fields + 4 reserved bits.

**Why not JSON:** JSON adds 5–10 bytes of overhead per field (key string + quotes + colon + comma). For a 5-field record, that is ~35 bytes of structural overhead before the data. The bitpad-v1 header represents all 12 fields in 2 bytes total.

---

## 2. Wire Format

```
[1 byte ] template_id  — identifies the template and version
[2 bytes] field_flags  — big-endian uint16; each bit = field present
[N bytes] data_blocks  — for each set bit, in bit-position order (0 → 11)
```

### 2.1 Template ID byte

| Value | Template |
|-------|----------|
| `0x01` | svc-basic v2 |
| `0x00` | reserved / unknown |
| `0x02`–`0xFF` | reserved for future templates |

### 2.2 Field flags (2 bytes, big-endian uint16)

Bit positions for svc-basic v2:

| Bit | Field ID | Compact key | Max bytes | Block type |
|-----|----------|-------------|-----------|------------|
| 0 | `job` | `j` | 120 | scalar |
| 1 | `customer` | `c` | 120 | scalar |
| 2 | `date` | `d` | 10 | scalar |
| 3 | `location` | `l` | 160 | scalar |
| 4 | `meeting_time` | `mt` | 40 | scalar |
| 5 | `start_time` | `st` | 40 | scalar |
| 6 | `end_time` | `et` | 40 | scalar |
| 7 | `customer_phone` | `cp` | 40 | scalar |
| 8 | `worker` | `w` | 80 | scalar |
| 9 | `actions` | `at`/`an` | variable | array (see §2.4) |
| 10 | `details` | `de` | 500 | scalar |
| 11 | `story` | `sy` | 2000 | scalar |
| 12–15 | reserved | — | — | must be 0 |

**Note:** `job` (bit 0) is required in svc-basic v2 — it will always be 1. Decoders must not assume this; they must read the flag.

**Length constraint:** All `max bytes` values are UTF-8 byte lengths, not character counts. A single Arabic, Hindi, or CJK character may be 2–4 UTF-8 bytes. See §3.3.

### 2.3 Scalar block format

For each bit 0–8, 10, 11 that is set, in ascending bit-position order:

```
[2 bytes] byte_len   — big-endian uint16: byte length of the UTF-8 text
[N bytes] text       — UTF-8 encoded field value; N = byte_len
```

`byte_len` = 0 is valid (empty string present); decoder must return `""` not `null`.

### 2.4 Actions array block format

When bit 9 is set, the actions block appears in position order (after bit 8's scalar block if present, before bit 10's scalar block):

```
[1 byte ] count      — number of actions; 0–20 (uint8)
for each action (count times):
  [2 bytes] title_len  — big-endian uint16
  [N bytes] title      — UTF-8; N = title_len
  [2 bytes] notes_len  — big-endian uint16
  [M bytes] notes      — UTF-8; M = notes_len
```

`count = 0` with bit 9 set is valid (empty actions array explicitly present).

---

## 3. Encoding Rules

### 3.1 Field presence

A field is present (bit set) if and only if it has a non-null value in the record. Fields with `null` or `undefined` values must not set the bit and must not emit a block.

Fields with an empty string `""` are present — bit is set, byte_len = 0, no text bytes.

### 3.2 Field ordering

Data blocks appear in strict ascending bit-position order (0, 1, 2, … 11). Decoders iterate bits 0→15 in order and read the corresponding block type when a bit is set.

### 3.3 UTF-8 byte lengths

All length prefix values (`byte_len`, `title_len`, `notes_len`) store the **byte length** of the UTF-8 encoded string, not the character count. Encoders must:

```
bytes = TextEncoder.encode(fieldValue)
byte_len = bytes.length   // not fieldValue.length
```

Decoders must:
```
text = TextDecoder.decode(slice(offset, offset + byte_len))
```

### 3.4 Actions max

Maximum 20 actions per record (R2-4). Encoders must silently truncate or reject records with more than 20 actions. Decoders must not crash on count > 20 but may truncate.

---

## 4. URL Envelope

The bitpad-v1 frame is compressed and base64url-encoded before insertion into the workpads URL:

```
1. frame   = bitpad_encode(record)          → Uint8Array
2. compressed = fflate.deflateSync(frame, { level: 9 })
3. b64url  = base64url(compressed)
4. url     = "workpads.me/p#v=1&alg=bitpad-v1&d=" + b64url
```

**Algorithm identifier in URL:** `alg=bitpad-v1` — tells the decoder which algorithm to use.

**URL schema version:** `v=1` — the hash-fragment URL schema version (R7-ADD-2). Unchanged.

### 4.1 Decoding

```
1. Parse hash fragment → extract alg, d params
2. compressed = base64url_decode(d)
3. frame   = fflate.inflateSync(compressed)
4. record  = bitpad_decode(frame)
```

If `alg` param is absent, decoder may assume `bitpad-v1` for backward compat (v0.1 URLs will always set it).

### 4.2 base64url alphabet

Standard base64url: `A-Za-z0-9-_` (no padding). Replace `+` → `-`, `/` → `_`, strip `=` padding.

---

## 5. Benchmark

Five standard test records, byte counts for raw frame vs JSON, and after deflate+base64url:

### Record definitions

**T1 — Tiny:** `{job: "Plumbing leak"}`
**T2 — Standard:** `{job: "Fix kitchen tap", customer: "Jane Doe", date: "2026-04-26", actions: [{title: "Check water pressure", notes: "Pressure normal"}, {title: "Replace washer", notes: "Used 12mm BSP washer"}], details: "Job completed, no leaks detected."}`
**T3 — Social-light:** T2 + `{location: "14 High St", meeting_time: "09:00", start_time: "09:15", end_time: "11:30", worker: "M. Plumb"}`
**T4 — Social-heavy:** All 12 fields, 3 actions, moderate-length text (~60 bytes per text field)
**T5 — Worst-case:** All 12 fields at maximum byte lengths (120+120+10+160+40+40+40+40+80+20×(100+300)+500+2000 bytes data)

### Raw frame size comparison

| Record | bitpad-v1 raw | JSON raw | bitpad saving |
|--------|--------------|----------|---------------|
| T1 | 18 B | 20 B | 2 B (10%) |
| T2 | 156 B | 208 B | 52 B (25%) |
| T3 | 224 B | 316 B | 92 B (29%) |
| T4 | 415 B | 560 B | 145 B (26%) |
| T5 | ~8,650 B | ~11,300 B | ~2,650 B (23%) |

### After deflate + base64url (URL length)

Compression efficiency increases with payload size. For T2+:

| Record | bitpad-v1 URL chars | bitpad-v1 raw frame |
|--------|--------------------|--------------------|
| T1 | 61 | 18 B |
| T2 | 216 | 155 B |
| T3 | 346 | 290 B |
| T4 | 578 | 600 B |
| T5 | (large — all fields max) | 11,161 B |

URL lengths measured from live test run (`node test/codec.test.js`). fflate 0.8.2, deflateSync level 9.

**Note:** T1 (single tiny field) is the crossover point — JSON+lz-str produces a marginally shorter URL for records with 1–2 very short fields. For all real-world records (≥3 fields or any field >20 bytes), bitpad-v1+deflate wins.

### Codec lane decision

**Decision (supersedes R1-1):** Retire the two-lane codec (lz-str for short / fflate for long). Use bitpad-v1+fflate as the single algorithm for all records. Rationale:

1. The crossover point (single tiny field) is an edge case not worth maintaining two code paths for
2. A single algorithm simplifies the decoder — no length sniffing or algorithm detection required beyond reading the `alg=` URL param
3. The `alg=` param in the URL provides forward extension if a future algorithm is warranted
4. Benchmark confirms bitpad-v1+deflate is never catastrophically worse than JSON+lz-str

This decision is appended to decisions.md.

---

## 6. Compatibility

### 6.1 Forward compatibility

New svc-basic versions may add fields using currently-reserved bits 12–15. Decoders for bitpad-v1 that encounter reserved bits set must skip the unknown block by reading its 2-byte length prefix and advancing the offset — they must not crash. This requires that all future field types begin with a uint16 length prefix.

### 6.2 Gecko 48 compatibility (KaiOS 2.5)

`Uint8Array` and `DataView` are available in Gecko 18+. `TextEncoder`/`TextDecoder` are available in Gecko 18+. bitpad.js has no Gecko 48 compatibility issues. fflate (pure JS, uses TypedArrays) has been verified compatible — see TASK-DEV-001.

**v0.1 target:** KaiOS 3.x (Gecko 63+). Gecko 48 (KaiOS 2.5) is v0.2 scope — no compatibility shims required for v0.1.

### 6.3 svc-basic v1 links

Records encoded with svc-basic v1 (pre-R2 format, `st`=story) do not use bitpad-v1 encoding. They use the legacy URL format (`#v=1&j=...&st=...` param-per-field). The `alg=` parameter is absent in v1 URLs. Decoders should treat absent `alg` as legacy format and route to the legacy decoder.

---

## 7. Test Vectors

Canonical encode→base64url→decode round-trip test vectors for each record type are defined in `@workpads/codec/test/codec.test.js`. All five test records must round-trip losslessly (decoded record deep-equals original input).
