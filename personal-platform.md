# §7 — Personal Platform

**Status:** v1 draft — 2026-04-26
**Decision source:** `workpads-basicsconform/system/logs/decisions.md` Round 4 (pending quiz responses)
**Original reference:** `workpads-standard/original/README.md` — earliest written standard

---

## Principle

Workpads is a **Personal Platform** — a user-sovereign, secure platform architecture that carries two engines as first-class concerns:

**Exchange Engine** — supports record generation, transmission, and storage. Directly implemented by the PADS model (Process / Actions / Details / Story), RecordService, CodecService, and LinkService. What the tool *does*.

**Learning Engine** — helps users build understanding and knowledge about records and other personal processes. Implemented by PersonalService and the Story field (Screen 4). What the tool *accumulates* on the user's behalf.

Both engines share the same UI shell. The distinction is architectural, not experiential — a worker should never feel like they are switching modes.

---

## The Meta-Activity

Alongside every work activity, a meta-activity always occurs:

- Activity is dispatched or performed
- Processes unfold
- Business devices are used and interacted with
- The worker has personal observations, learnings, hunches, and ideas — often private, often ephemeral, often valuable

For most workers this reality is more of an **ether** than an explicit process. They do not naturally pause to document their internal experience of work. The Personal Platform's Learning Engine is designed to assist this — making capture ambient, low-friction, and available without interrupting the work.

---

## Record Sovereignty

Both engines carry the record sovereignty principle:

- **Exchange records** (workpads) are sovereign to the worker who creates them. They are not by default the employer's property and are not transmitted without explicit worker action.
- **Personal accumulations** (Learning Engine captures) are doubly sovereign — they are explicitly personal. They may reference exchange records but are never embedded in them without explicit worker action.
- **Portability**: a worker's personal accumulations travel with them. The permission architecture for portability is context-dependent (employer/employee, contractor/client, public/private) and must be navigated by the platform, not assumed by the tool.

---

## Architecture Boundary

| Engine | Primary service | Storage namespace | Sharing |
|--------|-----------------|-------------------|---------|
| Exchange Engine | RecordService | `wp_record_*` | Explicit (share link / QR) |
| Learning Engine | PersonalService | `wp_personal_*` | Opt-in (export / link) |

Services communicate in-process (KaiOS v0.1). The gateway orchestrates Exchange Engine operations; PersonalService operates independently with its own interface.

---

## BASICS Correspondence

| Personal Platform Concept | BASICS Rule |
|--------------------------|-------------|
| Profile/context separation (two namespaces) | BASICS-SC-020 |
| Operator-visible storage behavior for both engines | BASICS-SC-021 |
| Quality evidence — operator narrative IS evidence | BASICS-SC-060 |
| Record sovereignty + portability | BASICS-SC-055, BASICS-DEV-POL |
| Exchange Engine offline-first operation | BASICS-SC-041, BASICS-SW-010 |

---

## Reference

The two-engine framing is drawn from the Platform Design Toolkit methodology, which distinguishes between **transaction engines** (what a platform enables) and **learning engines** (how a platform grows participant capability). Workpads uses "Exchange Engine" for the transaction side to emphasize that the fundamental unit is an exchange — a record of something given, received, or performed — not merely a data transaction.

The machine-architecture layer (`workpads-standard/original/machine-architecture/workpad-sequence.md`) establishes that the Exchange Engine is grounded in a binary exchange record format (32+64 bit sequences) designed for radio, bluetooth, and serial transmission. The JSON/PADS representation is a high-level view of this binary reality.

---

---

## PersonalService Interface (confirmed — TASK-ARC-006, 2026-04-27)

PersonalService is the Learning Engine's service module. It is a direct-import JS module (`personal.js`), not a networked service. Storage is handled by a prefix-scoped StorageAdapter instance.

### Method signatures

```javascript
// Capture a personal observation
// Returns the full created capture object (with id + timestamp populated)
capture({ text, tags, source, linkedRecordId?, linkedFieldId? }) → Promise<capture_object>

// List all active captures, sorted by timestamp descending (most recent first)
list() → Promise<Array<capture_object>>

// Retrieve one capture by ID; returns null if not found
get(id) → Promise<capture_object|null>

// Soft-delete: moves capture to archive namespace (wp_archive_)
archive(id) → Promise<void>

// Count active captures
count() → Promise<number>

// Export captures
// format: 'text' → one line per capture: "[Apr 27 09:15] text #tag1"
// format: 'json' → full capture objects array (lossless)
// since: optional epoch-ms timestamp — only captures at or after this time
export({ format: 'text'|'json', since?: number }) → Promise<string>
```

### Capture schema

```javascript
{
  id:             string,    // "wp_<timestamp_ms><random4>"
  text:           string,    // the captured observation (free text)
  tags:           string[],  // free-form; max 5 per capture, max 20 chars each
  source:         string,    // 'quick-note' | 'field-capture' | 'story-draft'
  linkedRecordId: string?,   // set when source = 'field-capture'
  linkedFieldId:  string?,   // set when source = 'field-capture'; field name from svc-basic v2
  timestamp:      number,    // epoch ms
  archived:       boolean    // false = active; true = archived (soft-deleted)
}
```

### Storage

| Key prefix | Adapter instance | Contents |
|-----------|-----------------|---------|
| `wp_personal_` | `personalStore` | Active captures |
| `wp_archive_` | `archiveStore` | Archived (soft-deleted) captures |

No separate index key — `personalStore.list()` (from storage adapter) scans the namespace directly.

### Source values and their meaning

| Source | Trigger context | Linked fields |
|--------|----------------|---------------|
| `'quick-note'` | Long-press CSK, `*` key, Personal Panel menu | None |
| `'field-capture'` | ArrowRight → Personal Panel while in PADS wizard | `linkedRecordId` + `linkedFieldId` |
| `'story-draft'` | Story field save without sharing | `linkedRecordId` only |

### Privacy guarantee

`linkedRecordId` and `linkedFieldId` are stored only in `wp_personal_*` namespace. They never appear in any exchange record, share link, QR code, or codec output. GatewayService and LinkService have no access to PersonalService storage.

### Error handling

- Tag validation exceeded (>5 tags or tag >20 chars): throws `ValidationError`
- Storage failure: throws `StorageError` (from adapter)
- `get()` / `archive()` on non-existent ID: `get()` returns null; `archive()` succeeds silently
