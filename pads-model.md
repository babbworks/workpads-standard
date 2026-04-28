# §1 — PADS Data Model

**Status:** v1 (settled 2026-04-27)
**Applies to:** All Workpads implementations

---

## Overview

The PADS model is the structural foundation of every workpad record. It divides the information required for a field work job into four named sections, each corresponding to a distinct phase of information capture:

| Section | Name | When it is filled |
|---------|------|-------------------|
| **P** | Process | Before the job — what is being done and for whom |
| **A** | Actions | During the job — the steps being performed |
| **D** | Details | During or after — logistics, timing, contacts |
| **S** | Story | After the job — what happened, in plain language |

The sections are sequential in creation flow (wizard steps 0-3) but the underlying record is a flat object. All sections are optional except for the `job` field in the Process section, which serves as the record's primary identifier.

---

## Section Definitions

### P — Process

The Process section answers: *What job is this, who is it for, when, and where?*

| Field | Key | Type | Required | Description |
|-------|-----|------|----------|-------------|
| Job title | `job` | string | **Yes** | Primary record identifier. Short descriptor of the work to be done. E.g. `Replace faucet`, `Site inspection`, `Quarterly maintenance`. |
| Customer | `customer` | string | No | Name of the customer, client, or responsible party. Used to auto-populate Block Registry on save. |
| Date | `date` | string | No | Work date in ISO 8601 format (YYYY-MM-DD). Free text also accepted (implementations should not enforce format). |
| Location | `location` | string | No | Physical address or location descriptor. Free text — no geocoding or structured addressing required. |

### A — Actions

The Actions section answers: *What specific steps are being taken?*

Actions are a list of step items. Each item has a title and optional notes. They function as a lightweight checklist or procedure log.

```json
"actions": [
  { "title": "Arrive",   "notes": "Check scope with customer" },
  { "title": "Install",  "notes": "Replace fixture and seal" },
  { "title": "Test",     "notes": "" }
]
```

| Field | Key | Type | Description |
|-------|-----|------|-------------|
| Actions list | `actions` | array | Array of action objects |
| Action title | `actions[n].title` | string | Short label for the step. Required per action item. |
| Action notes | `actions[n].notes` | string | Optional detail notes for the step. May be empty string. |

There is no maximum on the number of actions. Implementations may display them as a checklist, a numbered list, or a flat log.

### D — Details

The Details section answers: *Who is doing this work, how can they be contacted, and when does it happen?*

| Field | Key | Type | Description |
|-------|-----|------|-------------|
| Worker | `worker` | string | Name of the person performing the work. Pre-fills from the Activity Profile (sender identity) when creating a new record. |
| Start time | `start_time` | string | Work start time. Free text (e.g. `09:00`, `09:00 AM`). |
| End time | `end_time` | string | Work end time. Free text. |
| Meeting time | `meeting_time` | string | Scheduled arrival or meeting time, if different from start. Free text. |
| Customer phone | `customer_phone` | string | Customer contact number. Combined with `customer` to auto-populate Block Registry on save. |

### S — Story

The Story section answers: *What happened? What were the findings or results?*

| Field | Key | Type | Description |
|-------|-----|------|-------------|
| Story | `story` | string | Narrative summary. The primary human-readable account of what occurred. Prose, not structured. |
| Details | `details` | string | Technical notes, material lists, observations, or any structured additional information. Separate from the narrative `story`. |

The distinction between `story` and `details`: `story` is the readable account of the job ("We found the pipe had corroded at the joint..."), while `details` is the technical log ("Replaced 15mm copper elbow, applied PTFE tape, pressure tested to 4 bar").

---

## Record Identity Fields

In addition to the PADS fields, every record has system-managed identity fields:

| Field | Key | Set by | Description |
|-------|-----|--------|-------------|
| Record ID | `id` | RecordService on create | Unique local identifier. Format is implementation-defined (e.g. UUID, sequential ID). Not transmitted in share links. |
| Created at | `createdAt` | RecordService on create | ISO 8601 timestamp of record creation. |
| Updated at | `updatedAt` | RecordService on update | ISO 8601 timestamp of last modification. |
| Received at | `receivedAt` | RecordService.storeReceived | Set when a record arrives via an incoming URL. Marks the record as received (not locally created). |

---

## Invariants

1. **`job` is the only required field.** All other fields are optional. A record with only `job` set is valid and encodable.
2. **All scalar fields are strings.** There are no numeric, boolean, or structured fields outside of `actions`.
3. **Absent fields are omitted, not null.** A field with no value should be `undefined` or absent from the object — not `null` or `""`. Encoding skips absent fields.
4. **Actions may be an empty array.** `actions: []` is valid.
5. **The model is flat.** There is no nesting within sections. All fields are at the top level of the record object.
6. **Sections are a conceptual grouping only.** The record object does not have `process`, `actions`, `details`, `story` sub-objects. All fields are directly on the root object.

---

## Template Versioning

The PADS model is expressed through **templates**. The first-party template is `svc-basic` (field service basic). The template version `v1` defines the field set and encoding slot assignments documented in §2 (Record Schema) and §5 (Codec & Compact Encoding).

Future template versions may add fields. Decoders encountering an unknown template byte should reject gracefully rather than partially decoding.

---

## Design Rationale

The four-section split reflects the natural temporal phases of a service call:
- **Process** is filled in by a dispatcher before the job starts.
- **Actions** are added progressively during the job.
- **Details** may be filled by either party (dispatcher knows the schedule, worker knows arrival time).
- **Story** is filled in by the worker or supervisor after the job ends.

This temporal structure supports progressive disclosure UIs: a 4-step wizard walks the user through the sections in order. Each section can be submitted independently — the record is auto-saved between steps.

The flat object structure (rather than nested sub-objects per section) is a deliberate encoding efficiency choice. Compact binary encoding (bitpad-v1) assigns fixed bit positions to fields, which requires a known, stable, flat field order.
