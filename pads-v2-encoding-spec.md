# pads-v1 Record Encoding Specification

**Status:** v0.1 draft — 2026-04-27
**Algorithm identifier:** `pads-v1`
**Supersedes:** bitpad-v1 (for new records; bitpad-v1 remains decodable via the existing `alg=bitpad-v1` URL format)
**Decision source:** decisions.md R-PADS-001 through R-PADS-022
**Implements:** svc-basic v2 + financial extension + participants + chains
**Implemented in:** `@workpads/codec` — `src/pads.js` (TASK-ARC-017)

---

## 1. Background and Lineage

pads-v1 is a binary-text hybrid encoding for workpad records, derived from and extending the bitpad-v1 encoding (`bitpad-record-encoding.md`). bitpad-v1 in turn derives from the BitPads protocol (REFS/Bitpads/) — specifically the Role C bit-flag presence header and length-prefixed text block patterns.

**Why a new format beyond bitpad-v1:**
bitpad-v1 handles the 13 svc-basic v2 fields efficiently. pads-v1 adds:
- Financial block (amounts, tax, charges, compound invoices)
- Participants block (sender identity, multiple workers, role types)
- Chain reference (record relationships)
- State Commit and Amendment record types
- Structured mode (binary time/date fields for ~72% URL reduction)
- Condensed URL prefix (18-char saving per URL)
- Draft and recipient-type signals in the meta header

**What does not change:**
- The field presence bit-flag architecture from bitpad-v1
- UTF-8 length-prefixed text block format
- fflate deflate compression + base64url envelope
- The core svc-basic v2 fields (bits 0–11)

---

## 2. URL Envelope

### 2.1 URL format

```
workpads.me/p#<scheme_tag>/<data>[&c=<chain_ref>]
```

Where:
- `<scheme_tag>` = 3 characters (see §2.2)
- `/` = literal separator between tag and data
- `<data>` = base64url encoding of `fflate.deflateSync(frame, {level:9})`
- `[&c=<chain_ref>]` = optional 4-character base64url chain reference (see §8)

**Examples:**
```
workpads.me/p#1ag/eJyLjgUA...          (no chain)
workpads.me/p#1ag/eJyLjgUA...&c=AbCd   (with chain reference)
```

**Previous format (bitpad-v1, still decodable):**
```
workpads.me/p#v=1&alg=bitpad-v1&d=eJy...
```
Prefix saving: `workpads.me/p#1ag/` (17 chars) vs `workpads.me/p#v=1&alg=bitpad-v1&d=` (35 chars) = 18 chars saved per URL.

### 2.2 Scheme tag (3 characters)

| Position | Range | Meaning |
|----------|-------|---------|
| 0 | `1`–`9` | pads version (currently: `1` = pads-v1) |
| 1 | `a`–`z` | Codebook package (currently: `a` = initial workpads codebook) |
| 2 | `a`–`z` | Compression algorithm (currently: `g` = gzip/deflate via fflate) |

Namespace: 9 × 26 × 26 = 6,084 URL scheme slots.

The codebook package (char 1) is the version selector for all codebooks simultaneously: template IDs, role type codes, charge type codes, expense type codes. Changing the codebook package letter (e.g. `b`) selects an entirely different set of codebooks — enabling breaking changes to any codebook without changing the URL scheme version or compression.

### 2.3 Decoding sequence

```
1. Read chars 0, 1, 2 of hash fragment → version, codebook, compression
2. Split on '/' → data portion (and optional &c= chain ref)
3. base64url_decode(data) → compressed bytes
4. fflate.inflateSync(compressed) → frame bytes
5. pads_decode(frame, codebook) → record object
6. If &c= present: parse chain_ref → chain lookup
```

### 2.4 base64url alphabet

Standard: `A-Za-z0-9-_` (no padding). Replace `+`→`-`, `/`→`_`, strip `=` padding.

---

## 3. Binary Frame Structure

```
[ frame ]  =
  [1 byte ] meta1              — always present
  [1 byte ] meta2              — if meta1 bit 7 set
  [1 byte ] setup_byte         — if DOMAIN flags financial (meta2 bits 3-2 ≥ 01)
  [1 byte ] transaction_byte   — if setup_byte present
  [2 bytes] field_flags        — always present (bits 0-15)
  [1 byte ] field_flags3       — if field_flags bit 15 set (EXTENDED_FIELDS)
  [ data blocks ]              — for each set bit in field_flags, in bit-position order
  [ participants block ]       — if meta2 bit 4 set (PARTICIPANTS)
```

---

## 4. Meta Byte 1

Always the first byte of the frame.

| Bit | Name | Meaning when set |
|-----|------|-----------------|
| 7 | META2_PRESENT | Meta Byte 2 follows |
| 6 | TEMPLATE_3 | Template ID bit 3 (MSB) |
| 5 | TEMPLATE_2 | Template ID bit 2 |
| 4 | TEMPLATE_1 | Template ID bit 1 |
| 3 | TEMPLATE_0 | Template ID bit 0 (LSB) |
| 2 | ACK_REQUEST | Sender requests recipient ACK URL |
| 1 | CHAIN | Chain reference present in URL (`&c=XXXX`) |
| 0 | RECIPIENT_TYPE | 0 = customer-facing, 1 = colleague-facing |

**Template ID** = bits 6-3 as a 4-bit value (0–15). See §4.1 for codebook.

### 4.1 Template IDs (codebook package `a`)

| ID (binary) | ID (hex) | Template |
|-------------|----------|---------|
| 0000 | 0x0 | reserved |
| 0001 | 0x1 | svc-basic v2 (general field service) |
| 0010 | 0x2 | svc-billing v1 (structured billing / T&M) |
| 0011–1100 | 0x3–0xC | reserved for future templates |
| 1101 | 0xD | State Commit (snapshot record — see §9) |
| 1110 | 0xE | Amendment (changed-fields record — see §10) |
| 1111 | 0xF | reserved |

---

## 5. Meta Byte 2

Present when meta1 bit 7 (META2_PRESENT) = 1. Encoders must set META2_PRESENT=1 if any bit in Meta Byte 2 would be non-zero.

| Bit | Name | Meaning when set |
|-----|------|-----------------|
| 7 | CONTINUATION | reserved — multi-frame future use |
| 6 | COMPACT_TIME | Binary time fields active (see §7) |
| 5 | EXTENDED_FIELDS | Third field flags byte present |
| 4 | PARTICIPANTS | Participants block present (see §6) |
| 3 | DOMAIN_1 | Domain bit 1 (MSB) |
| 2 | DOMAIN_0 | Domain bit 0 (LSB) |
| 1 | DRAFT | Record is a draft, not a finalized share |
| 0 | RESTRICT_FORWARD | Recipient should not share further |

**DOMAIN** = bits 3-2 as a 2-bit value:
| Code | Domain |
|------|--------|
| 00 | General (no financial block) |
| 01 | Financial |
| 10 | Engineering / measurement |
| 11 | Hybrid (financial + engineering) |

**AMEND signal:** For Amendment records (template 0xE), meta2 bit 4 is repurposed as AMEND when PARTICIPANTS=0. If both are needed, PARTICIPANTS=1 + AMEND is indicated by template ID alone.

---

## 6. Setup Byte

Present when DOMAIN ≥ 01 (financial). Follows Meta Byte 2 (or Meta Byte 1 if META2_PRESENT=0 and domain is derived from template — this case is for future extension; current encoders always use meta2 for domain).

| Bits | Name | Meaning |
|------|------|---------|
| 7–5 | DECIMAL_POS | Decimal places for all value fields (0–5) |
| 4–3 | CURRENCY | Currency code (see §6.1) |
| 2–1 | TAX_CODE | Tax treatment (see §6.2) |
| 0 | COMPOUND_VALUE | 1 = multiple value line items present |

### 6.1 Currency codes (codebook package `a`)

| Code | Currency |
|------|----------|
| 00 | GBP (£) |
| 01 | EUR (€) |
| 10 | USD ($) |
| 11 | Local / unspecified |

### 6.2 Tax codes

| Code | Treatment | Extra bytes |
|------|-----------|-------------|
| 00 | No tax | 0 |
| 01 | Zero-rated | 0 |
| 10 | Standard rate (implicit from currency/jurisdiction) | 0 |
| 11 | Explicit rate | 2 bytes per value: uint8 permille rate + uint16 tax amount |

Standard rate (10) adds 0 bytes — the rate is contextually known from the currency and the sender/recipient's jurisdiction. Encoders using code 10 should document the assumed rate in the Details or Story field if the jurisdiction is ambiguous.

---

## 7. Transaction Byte

Present when DOMAIN ≥ 01 (financial). Follows the Setup Byte.

| Bits | Name | Meaning |
|------|------|---------|
| 7 | DIRECTION | 0 = I (income/in-flowing), 1 = O (outgoing/expense) |
| 6 | TIME | 0 = `<` Past/settled, 1 = `>` Future/pending |
| 5 | EFFECT | 0 = I (asset-increasing), 1 = O (liability/outflow) |
| 4–3 | SUBTYPE | 2 bits: 4 subtypes per primary state |
| 2 | QTY_SPLIT | 0 = lump sum, 1 = qty × rate breakdown |
| 1–0 | ROUNDING | 00 = exact, 10 = down, 11 = up, 01 = error (BitLedger) |

**Primary states** (see `transaction-classification.md` for full table):
- `I < I` (0,0,0): Settled income
- `I > I` (0,1,0): Future income / invoice
- `I < O` (0,0,1): Income → outgoing / refund given
- `I > O` (0,1,1): Future refund / credit note
- `O < O` (1,0,1): Settled expense
- `O > O` (1,1,1): Future expense / bill received
- `O < I` (1,0,0): Recovered expense
- `O > I` (1,1,0): Future reimbursement expected

**Rounding defaults by transaction type (BitLedger rules):**
- Income (DIRECTION=0, EFFECT=0): DOWN (11→10)
- Liability / expense (DIRECTION=1, EFFECT=1): UP (11)
- Tax component: always UP (11) regardless of transaction type

---

## 8. Field Flags

Two bytes (16 bits), always present. Data blocks follow in strict ascending bit-position order for all set bits.

| Bit | Field | Block type | Max bytes (UTF-8) |
|-----|-------|------------|-------------------|
| 0 | `job` | scalar | 120 |
| 1 | `customer` | scalar | 120 |
| 2 | `date` | scalar / uint16 (if COMPACT_TIME) | 10 / 2 |
| 3 | `location` | scalar | 160 |
| 4 | `meeting_time` | scalar / uint16 (if COMPACT_TIME) | 40 / 2 |
| 5 | `start_time` | scalar / uint16 (if COMPACT_TIME) | 40 / 2 |
| 6 | `end_time` | scalar / uint16 (if COMPACT_TIME) | 40 / 2 |
| 7 | `customer_phone` | scalar | 40 |
| 8 | `worker` | scalar (legacy compat; prefer participants block) | 80 |
| 9 | `actions` | array (see §8.2) | variable |
| 10 | `details` | scalar | 500 |
| 11 | `story` | scalar | 2000 |
| 12 | `value_block` | financial block (see §8.3) | variable |
| 13–14 | extended | reserved | — |
| 15 | FLAGS3_PRESENT | 1 = third flags byte follows | — |

### 8.1 Scalar block format (text mode, COMPACT_TIME=0)

For each set bit (except 9, 12, 15) in text mode:
```
[2 bytes] byte_len   — big-endian uint16: UTF-8 byte length
[N bytes] text       — UTF-8; N = byte_len
```
`byte_len = 0` is valid (empty string present). Decoders return `""` not `null`.

### 8.1b Compact time/date format (COMPACT_TIME=1)

When meta2 COMPACT_TIME=1, time and date fields use binary encoding instead of scalar:

| Field | Encoding | Bytes |
|-------|----------|-------|
| `date` (bit 2) | uint16 days since 2025-01-01 | 2 |
| `meeting_time` (bit 4) | uint16 minutes since midnight (0–1439) | 2 |
| `start_time` (bit 5) | uint16 minutes since midnight | 2 |
| `end_time` (bit 6) | uint16 minutes since midnight | 2 |

No length prefix — the block is exactly 2 bytes. Decoders reconstruct ISO strings for display.

### 8.2 Actions array block

When bit 9 is set:
```
[1 byte ] count      — number of actions (0–20, uint8)
for each action:
  [2 bytes] title_len  — uint16
  [N bytes] title      — UTF-8
  [2 bytes] notes_len  — uint16
  [M bytes] notes      — UTF-8
```

### 8.3 Financial block (bit 12)

When field_flags bit 12 is set:
```
[simple, COMPOUND_VALUE=0]
  [3 bytes] customer_amount  — uint24 big-endian
  [if TAX_CODE=11] [1+2 bytes] tax_block
  [if worker_amount bit set in field_flags3] [3 bytes] worker_amount
  [if QTY_SPLIT=1] [3+3 bytes] qty + rate  — both uint24

[compound, COMPOUND_VALUE=1]
  [1 byte ] line_count       — number of line items (1–20)
  for each line:
    [1 byte ] charge_type    — see §8.4
    [3 bytes] amount         — uint24
    [if TAX_CODE=11] [1+2 bytes] tax_block
    [if QTY_SPLIT=1] [3+3 bytes] qty + rate
    [if charge_type=0] [2+N bytes] label  — uint16 + UTF-8
```

**uint24 value interpretation:** Value = bytes as big-endian uint24. Monetary amount = value ÷ 10^DECIMAL_POS. Example: value=12550, DECIMAL_POS=2 → £125.50.

**Encoder DECIMAL_POS selection:**
- Default: DECIMAL_POS=2 (pennies)
- If amount > 16,777,215 pennies (£167,772.15): set DECIMAL_POS=0, round to nearest pound
- If sub-penny precision needed: DECIMAL_POS=3

### 8.4 Charge type codebook (codebook package `a`)

| Code | Charge type |
|------|------------|
| 0 | Custom (label block present) |
| 1 | Urgency / emergency premium |
| 2 | After-hours / out-of-hours |
| 3 | Travel / mileage |
| 4 | Delivery / courier |
| 5 | Equipment hire |
| 6 | Consumables / materials markup |
| 7 | Subcontractor cost |
| 8 | Cancellation / no-show fee |
| 9 | Deposit / retainer |
| 10 | Credit / discount |
| 11 | Warranty / guarantee adjustment |
| 12 | Regulatory / compliance levy |
| 13 | Currency / FX adjustment |
| 14 | Payment handling / processing fee |
| 15 | Multi-charge (nested compound sub-block) |

---

## 9. Participants Block

Present when meta2 PARTICIPANTS=1. Follows all data blocks (including financial block if present).

```
[1 byte ] block_header  — bits 7-5: participant count (1-7), bits 4-0: reserved (0)
for each participant:
  [1 byte ] part_flags
  [2+N bytes] name      — uint16 byte_len + UTF-8 name (required)
  [if HAS_PHONE] [2+M bytes] phone
  [if HAS_EMAIL] [2+P bytes] email
  [if HAS_ROLE_TEXT] [2+Q bytes] role_text  — free text role description
```

**part_flags layout:**
| Bit | Name | Meaning |
|-----|------|---------|
| 7 | IS_SENDER | This participant is the record creator |
| 6–4 | ROLE_TYPE | 3 bits: role codebook entry (see §9.1) |
| 3 | HAS_PHONE | Phone block follows name |
| 2 | HAS_EMAIL | Email block follows phone (or name if no phone) |
| 1 | HAS_ROLE_TEXT | Role text block present |
| 0 | reserved | |

### 9.1 Role type codebook (codebook package `a`)

| Code | Role |
|------|------|
| 000 | Worker (general) |
| 001 | Job owner / supervisor |
| 010 | Subcontractor |
| 011 | Colleague |
| 100 | Site contact |
| 101 | Referred by |
| 110 | Witness / verifier |
| 111 | Extended (role_text carries code/description) |

**IS_SENDER:** Exactly one participant should have IS_SENDER=1. This is the person who generated the URL. If no participant has IS_SENDER=1, decoders should treat the first participant as the sender (graceful degradation).

**Worker field (bit 8) vs participants block:** The bit 8 `worker` text field is preserved for backward compat with bitpad-v1 records. pads-v1 encoders should use the participants block when any of the following are true: multiple participants, sender identity is included, role type is needed, phone/email of participant is included. For simple single-worker attribution with no identity detail, bit 8 `worker` text field remains adequate.

See `participants-block.md` for extended discussion and examples.

---

## 10. Field Flags Byte 3 (Extended)

When field_flags bit 15 = 1, a third flags byte follows the two standard flag bytes. Current assignments:

| Bit | Name | Meaning |
|-----|------|---------|
| 7 | WORKER_AMOUNT | Worker's internal amount present in financial block |
| 6–0 | reserved | |

---

## 11. State Commit Records (Template 0xD)

State Commit records use template ID 1101 (0xD). They record a state at a point in time — no directional flow, no transaction byte.

**State Commit frame structure:**
```
[meta1: template=1101, CHAIN=1 recommended]
[meta2: PARTICIPANTS if identities present, DOMAIN=00 unless financial summary]
[field_flags: typically job (bit 0) + details (bit 10) + others as needed]
[data blocks as normal]
[participants block if meta2 PARTICIPANTS=1]
```

**Subtype** (transaction byte SUBTYPE field — present only if DOMAIN=financial):

| SUBTYPE | Meaning |
|---------|---------|
| 00 | Job completion certificate |
| 01 | Pay summary |
| 10 | Dispute snapshot |
| 11 | Period summary (annual/periodic aggregate) |

State Commit records are chain-anchored (CHAIN=1) to relate them to the work they summarize. The chain reference points to the originating chain anchor.

See `chain-protocol.md` for State Commit usage in chain flows.

---

## 12. Amendment Records (Template 0xE)

Amendment records use template ID 1110 (0xE). They carry only the changed fields — not a full re-encoding. Viewers merge amendments chronologically onto the original record.

**Immutable fields** (cannot appear in an amendment record):
- Chain ID / chain anchor (the original record's chain reference)
- Original record creation timestamp
- Sender identity (IS_SENDER participant from original)
- Template type

**Amendment frame:** Standard pads-v1 frame with template=0xE. Field flags set only for changed fields. Participants block may update non-sender participants (e.g. add a site contact). Chain reference must match the original record's anchor (same anchor, same participant slot, sequence incremented).

---

## 13. Benchmark — pads-v1 vs bitpad-v1

Reference records from bitpad-record-encoding.md §5, with pads-v1 comparison:

| Record | bitpad-v1 URL | pads-v1 text mode | pads-v1 structured |
|--------|--------------|-------------------|-------------------|
| T1 (job only) | 61 chars | ~44 chars | — |
| T2 (5 fields, 2 actions, details) | 216 chars | ~196 chars | — |
| T3 (T2 + 5 more fields) | 346 chars | ~310 chars | ~170 chars |
| Billing record (T3 + financial) | — | ~340 chars | ~97 chars |
| Billing + participants | — | ~370 chars | ~120 chars |

**T1 improvement:** Prefix savings (18 chars) reduce T1 from 61 to ~44 chars.
**Structured mode:** When COMPACT_TIME=1, time fields (3 × "09:15" = 21 chars → 3 × 2 bytes = 6 bytes raw) combined with date (10 chars → 2 bytes) yield ~30 raw bytes saved before compression, translating to significant URL reduction.

---

## 14. Compatibility

### 14.1 bitpad-v1 backward compat

All bitpad-v1 URLs (`workpads.me/p#v=1&alg=bitpad-v1&d=...`) remain decodable by the `@workpads/codec` bitpad-v1 decoder. The absence of the 3-char scheme tag pattern is the detection signal. Decoders should route based on the URL prefix:
- Starts with `#v=1&alg=bitpad-v1`: → bitpad.decode()
- Starts with `#` + 3 chars + `/`: → pads.decode()
- Legacy: starts with `#` without version: → legacy param-per-field decoder

### 14.2 Forward compat (unknown template IDs)

Decoders encountering an unknown template ID should:
1. Attempt to decode using the field_flags (bits 0-11 are stable across templates)
2. Skip unknown blocks (read length prefix, advance offset)
3. Return a partial record with a `_partial: true` flag
4. Display job + date + worker as the minimum viewable record

Never-changing elements (safe to display from any version): template ID, chain reference, job (bit 0), date (bit 2), worker (bit 8) or IS_SENDER participant name.

### 14.3 Codebook package forward compat

Future codebook packages (letters b, c, ...) may redefine template IDs, role codes, and charge codes. Decoders must always consult the codebook package indicated by scheme tag char 1, not hardcode codebook package `a` assignments.

---

## 15. Test Vectors

Canonical test vectors for all template types, block types, and structured/text mode round-trips are defined in `@workpads/codec/test/pads.test.js` (TASK-ARC-017). Required coverage:
- svc-basic v2 with text-mode fields (T1–T5 equivalents)
- svc-basic v2 with COMPACT_TIME=1 (structured time fields)
- Financial block: simple, TAX_CODE=10, TAX_CODE=11, compound (3 line items)
- Participants block: sole trader, employer+worker, 3-participant record
- State Commit: job completion certificate, pay summary
- Amendment: single changed field, multiple changed fields
- Chain reference: encode/decode round-trip for 4-char b64url chain ID
