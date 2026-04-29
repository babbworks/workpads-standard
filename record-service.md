# §4 — RecordService Interface

**Status:** v1 (settled 2026-04-27)
**Applies to:** All Workpads implementations

---

## Purpose

RecordService is the canonical service for workpad record lifecycle management. It handles creating, reading, updating, deleting, archiving, and sharing records. All record operations go through RecordService — no other service or screen reads or writes record data directly.

RecordService is part of the **Exchange Engine**: the set of services responsible for creating, transmitting, and receiving workpad records.

---

## Interface

All methods return Promises. RecordService internally uses two StorageAdapter instances with different prefixes — one for active records, one for archives.

---

### `RecordService.create(fields) -> Promise<record>`

Creates a new record with a generated ID and timestamps, merges `fields`, and persists it to the active store.

```js
RecordService.create({ job: 'Replace faucet', customer: 'Alice' })
  .then(function(record) {
    // record.id is set, record.createdAt is set
  });
```

- `fields` may be empty (`{}`) — the record is created with only system fields set
- The generated record has `id`, `createdAt`, `updatedAt` set by the service
- Returns the complete stored record

---

### `RecordService.get(id) -> Promise<record|null>`

Retrieves a single active record by ID.

```js
RecordService.get('abc123').then(function(record) {
  if (!record) { /* not found */ }
});
```

---

### `RecordService.save(id, record) -> Promise<record>`

Full replacement save — writes the provided record object as-is (after setting `updatedAt`). Used when the complete record state is available (e.g. after wizard completion).

```js
RecordService.save(record.id, record).then(function(saved) {
  // saved is the persisted record
});
```

---

### `RecordService.update(id, fields) -> Promise<record>`

Partial update — merges `fields` into the existing record and saves. Used for auto-save during wizard navigation.

```js
RecordService.update(record.id, { story: 'Replaced elbow joint' }).then(function(updated) {
  // updated is the merged, persisted record
});
```

---

### `RecordService.delete(id) -> Promise<void>`

Permanently removes a record from the active store.

---

### `RecordService.list() -> Promise<Array<record>>`

Returns all active records, sorted newest-first (by `createdAt` descending).

```js
RecordService.list().then(function(records) {
  records.forEach(function(r) { console.log(r.job); });
});
```

---

### `RecordService.archive(id) -> Promise<void>`

Moves a record from the active store to the archive store. The record remains readable via `listArchived` but does not appear in `list`.

```js
RecordService.archive(record.id).then(function() {
  // record is now in archive
});
```

---

### `RecordService.listArchived() -> Promise<Array<record>>`

Returns all archived records.

---

### `RecordService.encodeUrl(record) -> string`

Encodes a record to a URL hash fragment string using the pads-v1 codec. Returns the fragment without the leading `#` and without the `https://workpads.me/p` prefix.

```js
var fragment = RecordService.encodeUrl(record);
// -> '1ag/<base64url>'
var fullUrl = 'https://workpads.me/p#' + fragment;
```

Throws if the record fails validation.

---

### `RecordService.decodeUrl(hashFragment) -> record`

Decodes a hash fragment string back to a record object. Does not persist the record.

```js
var hash = window.location.hash.slice(1);
var record = RecordService.decodeUrl(hash);
```

Throws if the fragment is malformed or uses an unknown algorithm.

---

### `RecordService.storeReceived(record) -> Promise<record>`

Persists a decoded incoming record (from a shared URL) with a `receivedAt` timestamp. The record is stored in the active store alongside locally-created records.

```js
RecordService.storeReceived(incomingRecord).then(function(stored) {
  // stored.receivedAt is set
  App.showView(stored);
});
```

---

## Record Lifecycle

```
create()
  |
  v
[active store]
  |
  +-- get() / list()     read
  |
  +-- save() / update()  modify
  |
  +-- archive()
  |     |
  |     v
  |   [archive store]
  |     |
  |     +-- listArchived()
  |
  +-- delete()           permanent removal

storeReceived() --> [active store] (tagged with receivedAt)
```

---

## ID Strategy

Record IDs are implementation-defined. Requirements:

- Must be unique within the active store
- Must be stable (the same record keeps the same ID across updates)
- Must be a string
- Must not be transmitted in share URLs (IDs are local)

The KaiOS reference implementation uses `Date.now().toString(36) + Math.random().toString(36).slice(2)` — a time-based base36 string with random suffix, producing IDs like `lzqr8k2a4f`. This is not a UUID; it is compact and sufficient for local use.

The legacy CLI uses sequential IDs (`loc_000001`, `loc_000002`). Both are valid under this contract.

---

## Service Boundaries

RecordService is responsible for:
- Record persistence (via StorageAdapter)
- ID generation and timestamp management
- URL encoding and decoding (delegates to WPCodec / @workpads/codec)
- Archive management

RecordService is **not** responsible for:
- Contact storage (BlockRegistry handles `customer_phone` auto-save)
- Identity (ActivityService manages sender profile)
- Personal notes (PersonalService)
- UI rendering (screen layer)
- Codec implementation (WPCodec / @workpads/codec)

---

## Error Handling

- `create`, `save`, `update`, `get`, `list`, `archive`, `listArchived`: reject only on storage failure (rare with localStorage; possible with QuotaExceededError)
- `encodeUrl`: throws synchronously (or rejects) if record is invalid or codec fails
- `decodeUrl`: throws synchronously if the fragment is malformed
- `storeReceived`: may reject if storage fails; does not validate the incoming record beyond what the codec already guaranteed during decode
