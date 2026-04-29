# Codec Sync Protocol

**Status:** v1 (2026-04-28)  
**Source:** Adapted from workpadsdotme/system/SYNC.md  
**Applies to:** All Workpads implementations that inline or bundle the pads-v1 codec

---

## The Problem

The pads-v1 codec (`codec.md` §5) is implemented in multiple places across the Workpads ecosystem:

| Location | Codec copy | Purpose |
|----------|------------|---------|
| `workpads-codec` npm package | Canonical source | Used by CLI and as reference |
| `workpadskaios/js/lib/codec.js` | Inline UMD bundle | KaiOS runtime (no npm) |
| `workpadsdotme/js/lib/codec.js` | Inline browser bundle | Web app main codec |
| `workpadsdotme/p/index.html` | Inline decode-only | Standalone receiver (no app required) |
| `workpadsdotme/p/customer.html` | Inline encode+decode | Customer ACK flow |

When the spec changes — a field is added, a slot is reassigned, the actions blob format is updated — every copy must be updated together. A missed copy produces a decoder that silently mis-parses records from newer encoders, or an encoder generating URLs no older receiver can decode.

The canonical specification is always `codec.md`. Code is truth; spec is spec. When they conflict, investigate before deciding which to change.

---

## What Must Stay in Sync

These are the codec elements most likely to drift:

### 1. Scheme tag regex

The scheme tag identifies the codec generation. All copies must parse the same tag.

```js
// Canonical (v1.1)
var SCHEME = /^(?:https?:\/\/workpads\.me\/p[/?]?)?#?1ag\//;
```

If the scheme tag changes (new codebook char), update this regex in every copy simultaneously. A mismatch here causes complete decode failure — no partial degradation.

### 2. SCALAR_FIELDS array

The field slot table maps bit positions to field names. Order, indices, and names must be identical in every copy.

```js
// Canonical (v1.1) — 15 fields, bits 0–15 (bit 9 reserved for actions)
var SCALAR_FIELDS = [
  'job',           // bit 0
  'customer',      // bit 1
  'date',          // bit 2
  'location',      // bit 3
  'meeting_time',  // bit 4
  'start_time',    // bit 5
  'end_time',      // bit 6
  'customer_phone',// bit 7
  'worker',        // bit 8
  // bit 9 = actions blob (not a scalar)
  'details',       // bit 10
  'story',         // bit 11
  'amount',        // bit 12
  'currency',      // bit 13
  'vat',           // bit 14
  'record_type'    // bit 15
];
```

Adding a field: assign the next available bit, append to the array, bump the codebook char in the scheme tag.  
Removing a field: never reuse its bit position. Mark it `null` or `_reserved_NN` in the array. Bump the codebook char.

### 3. Actions blob format

The actions blob structure (at bit 9) must be identical in all copies:

```
[count: uint8]
  for each action:
    [title_len: uint16 big-endian]
    [title: UTF-8 bytes]
    [notes_len: uint16 big-endian]
    [notes: UTF-8 bytes]
```

Max actions: 20. A count byte > 20 is a decode error.

### 4. Template byte

Template byte `0x01` = svc-basic. This byte is the first byte of every pads-v1 frame. All copies must emit `0x01` for svc-basic records and reject unknown template bytes (or at minimum flag them as unrecognised).

### 5. Byte order

All multi-byte integers are **big-endian**. `uint16` lengths, presence flags — big-endian throughout. Do not change this; it is fixed for the lifetime of the `1ag` codebook.

---

## Change Procedure

When `codec.md` is updated:

1. **Identify which sync elements changed** (scheme tag, field list, actions format, template byte, byte order).
2. **Update `workpads-codec` first** — this is the canonical implementation. Tests must pass.
3. **Update inline copies in order:**
   - `workpadsdotme/js/lib/codec.js`
   - `workpadsdotme/p/index.html` (decode-only — only update decode path)
   - `workpadsdotme/p/customer.html` (encode + decode — update both paths)
   - `workpadskaios/js/lib/codec.js`
4. **If SCALAR_FIELDS changed:** bump the codebook char in the scheme tag. Update the SCHEME regex in all copies at the same time. Old decoders will reject new URLs (by design — the scheme tag is the compatibility signal).
5. **If only the actions format or byte limit changed without a field set change:** the codebook char does not need to bump, but add a version comment noting the change and the date.
6. **Run tests:** `workpads-codec` round-trip tests, KaiOS `test/flow.test.js`, workpadsdotme manual encode/decode smoke test.
7. **Update `codec.md`** section version (e.g., v1.1 → v1.2) if the change is breaking; add a note if additive.

---

## Checklist for Codec PRs

Before merging any change that touches codec logic:

- [ ] `codec.md` section version updated if breaking
- [ ] `workpads-codec` tests pass
- [ ] `workpadsdotme/js/lib/codec.js` updated
- [ ] `workpadsdotme/p/index.html` decode path updated (if decode changed)
- [ ] `workpadsdotme/p/customer.html` updated (if encode or decode changed)
- [ ] `workpadskaios/js/lib/codec.js` updated
- [ ] Scheme tag bumped if SCALAR_FIELDS changed
- [ ] SCHEME regex updated in all copies if scheme tag changed
- [ ] `implementation-notes.md` updated if a deviation is introduced or resolved

---

## Decode-Only Copies

`workpadsdotme/p/index.html` is a standalone receiver. It only needs the decode path. When updating it:

- Do **not** import encode logic — keep the file self-contained and minimal
- Update SCHEME regex, SCALAR_FIELDS, and the actions blob parser
- Do **not** add fields it doesn't yet render to the UI — add a TODO comment if a new field lands in the codec before the receiver UI handles it

---

## Detecting Drift

If you suspect copies have drifted:

```bash
# Compare SCALAR_FIELDS across copies
grep -h 'SCALAR_FIELDS\|scalar_fields' \
  workpads-codec/src/codec.js \
  workpadskaios/js/lib/codec.js \
  workpadsdotme/js/lib/codec.js \
  workpadsdotme/p/index.html \
  | sort | uniq -c | sort -rn
```

If the field list appears more than once with different content, there is drift. The `workpads-codec` version is authoritative.

---

## See Also

- `codec.md` — §5 normative specification
- `implementation-notes.md` — DEV-WP-URL-001 (scheme tag mismatch, KaiOS v0.1 vs web)
- `build-strategy.md` — §7 two-build strategy and codec bundling approach
