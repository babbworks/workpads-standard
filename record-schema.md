# §2 — Record Schema (svc-basic v1)

**Status:** v1 (settled 2026-04-27)
**Template:** `svc-basic`
**Template version:** `v1`
**Codec:** bitpad-v1

---

## Overview

This section defines the complete field schema for the `svc-basic` template, version 1. This is the first-party Workpads template for field service work records. It is the only template in v0.1.

---

## Full Field Listing

| # | Field | Key | Type | Section | Required | Codec slot |
|---|-------|-----|------|---------|----------|------------|
| 1 | Job title | `job` | string | Process | **Yes** | bit 0 |
| 2 | Customer | `customer` | string | Process | No | bit 1 |
| 3 | Date | `date` | string | Process | No | bit 2 |
| 4 | Location | `location` | string | Process | No | bit 3 |
| 5 | Start time | `start_time` | string | Details | No | bit 4 |
| 6 | End time | `end_time` | string | Details | No | bit 5 |
| 7 | Meeting time | `meeting_time` | string | Details | No | bit 6 |
| 8 | Customer phone | `customer_phone` | string | Details | No | bit 7 |
| 9 | Worker | `worker` | string | Details | No | bit 8 |
| 10 | Details | `details` | string | Story | No | bit 9 |
| 11 | Story | `story` | string | Story | No | bit 10 |
| 12 | Actions | `actions` | array | Actions | No | bit 11 |

Codec slot numbers correspond directly to bit positions in the 2-byte presence flags of the bitpad-v1 frame (see §5).

---

## Runtime Schema (JSON)

The compiled runtime schema for `svc-basic v1` is:

```json
{
  "id": "svc-basic",
  "version": "v1",
  "label": "Field Service — Basic",
  "codec": "bitpad-v1",
  "variants": ["plain"],
  "fields": [
    { "key": "job",            "label": "Job",            "type": "string", "required": true,  "section": "process" },
    { "key": "customer",       "label": "Customer",       "type": "string", "required": false, "section": "process" },
    { "key": "date",           "label": "Date",           "type": "string", "required": false, "section": "process", "inputType": "date" },
    { "key": "location",       "label": "Location",       "type": "string", "required": false, "section": "process" },
    { "key": "start_time",     "label": "Start time",     "type": "string", "required": false, "section": "details" },
    { "key": "end_time",       "label": "End time",       "type": "string", "required": false, "section": "details" },
    { "key": "meeting_time",   "label": "Meeting time",   "type": "string", "required": false, "section": "details" },
    { "key": "customer_phone", "label": "Customer phone", "type": "string", "required": false, "section": "details", "inputType": "tel" },
    { "key": "worker",         "label": "Worker",         "type": "string", "required": false, "section": "details" },
    { "key": "details",        "label": "Details",        "type": "string", "required": false, "section": "story",   "multiline": true },
    { "key": "story",          "label": "Story",          "type": "string", "required": false, "section": "story",   "multiline": true },
    { "key": "actions",        "label": "Actions",        "type": "array",  "required": false, "section": "actions" }
  ]
}
```

---

## Record Object (JavaScript)

At runtime, a record is a plain JavaScript object:

```js
{
  // System fields
  id:          'string',    // implementation-defined; not encoded in share URL
  createdAt:   'string',    // ISO 8601
  updatedAt:   'string',    // ISO 8601
  receivedAt:  'string',    // ISO 8601 — only present on received records

  // PADS fields (all optional except job)
  job:            'string',
  customer:       'string',
  date:           'string',
  location:       'string',
  start_time:     'string',
  end_time:       'string',
  meeting_time:   'string',
  customer_phone: 'string',
  worker:         'string',
  details:        'string',
  story:          'string',
  actions: [
    { title: 'string', notes: 'string' },
  ],
}
```

Fields that have no value are omitted (not set to `null` or `""`). System fields are always present on stored records. PADS fields may be absent.

---

## Validation Rules

A record is valid for encoding if:

1. `job` is a non-empty string
2. All present scalar fields are strings
3. `actions`, if present, is an array where each element has `title` as a non-empty string
4. `notes` within an action may be an empty string

A record failing validation must not be encoded. `WPCodec.validate(record)` and `@workpads/codec validate(record)` implement these checks and return `{ valid: boolean, errors: string[] }`.

---

## Variants

`svc-basic` supports one variant: `plain`.

Variants are a future extension point for sub-types of a template (e.g. a `detailed` variant requiring more fields, or an `inspection` variant with a different field set). In v0.1, the variant field is stored on the record but has no effect on field availability or encoding.

---

## Template Source Files

| File | Format | Purpose |
|------|--------|---------|
| `templates/svc-basic.kv` | Key-value | Human-readable canonical definition |
| `templates/svc-basic.yaml` | YAML | Compile source |
| `templates/runtime/svc-basic.v1.json` | JSON | Runtime schema loaded by implementations |

The `.kv` format is the canonical source of truth. The `.yaml` file is a compilation source. The `.json` file is produced by `template:compile` and is what implementations actually load.

---

## Adding Future Templates

New templates must:

1. Define a unique template identifier byte (the first byte of the bitpad-v1 frame)
2. Define a fixed, stable field order for the presence flags
3. Register a compiled runtime schema in the template registry
4. Be documented in this standard before use

The `svc-basic` template occupies template byte `0x01`. Future templates start at `0x02`.
