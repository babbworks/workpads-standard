# §6 — Storage Adapter Contract

**Status:** v1 (settled 2026-04-27)
**Applies to:** All Workpads implementations using local storage

---

## Purpose

The Storage Adapter is the single abstraction layer between Workpads services and the underlying storage mechanism. All reads and writes to persistent state go through the Storage Adapter. This keeps services independent of the storage backend (localStorage, IndexedDB, filesystem, in-memory) and provides a consistent async interface.

---

## Contract

### Instantiation

```js
var store = StorageAdapter(prefix);
```

`prefix` is a string prepended to all keys managed by this instance. Each service uses a distinct prefix to namespace its data:

| Service | Prefix |
|---------|--------|
| RecordService (active) | `wp_rec_` |
| RecordService (archive) | `wp_arc_` |
| ActivityService | `wp_act_` |
| PersonalService | `wp_per_` |
| BlockRegistry | `wp_block_` |

Multiple `StorageAdapter` instances with different prefixes are independent — listing or clearing one does not affect another.

### Methods

All methods return Promises, even when the underlying implementation is synchronous. This ensures services are written asynchronously throughout and can switch to an async backend (IndexedDB, fetch) without service changes.

---

#### `store.get(key) -> Promise<value|null>`

Returns the parsed value for the given key, or `null` if not found.

`key` is the unprefixed key — the adapter prepends the prefix internally.

```js
store.get('abc123').then(function(record) {
  if (record) { /* found */ }
});
```

---

#### `store.set(key, value) -> Promise<void>`

Serialises `value` to JSON and writes it under `prefix + key`.

```js
store.set('abc123', { job: 'Replace faucet', ... }).then(function() {
  // written
});
```

---

#### `store.remove(key) -> Promise<void>`

Removes the entry for `prefix + key`. No-op if not found.

```js
store.remove('abc123').then(function() {
  // removed
});
```

---

#### `store.list() -> Promise<Array<{ key, data }>>`

Returns all entries managed by this adapter (all keys with the matching prefix), as an array of `{ key, data }` objects.

- `key` is the unprefixed key (prefix stripped)
- `data` is the parsed value

Order is not guaranteed. Callers should sort as needed.

```js
store.list().then(function(entries) {
  entries.forEach(function(e) {
    console.log(e.key, e.data.job);
  });
});
```

---

#### `store.count() -> Promise<number>`

Returns the number of entries managed by this adapter.

```js
store.count().then(function(n) {
  console.log(n + ' records stored');
});
```

---

## Reference Implementation (localStorage)

The KaiOS implementation uses `localStorage` as the backing store. The adapter wraps synchronous `localStorage` calls in `Promise.resolve()` to satisfy the async contract.

```js
function StorageAdapter(prefix) {
  function fullKey(k) { return prefix + k; }

  return {
    get: function(key) {
      var raw = localStorage.getItem(fullKey(key));
      return Promise.resolve(raw ? JSON.parse(raw) : null);
    },
    set: function(key, value) {
      localStorage.setItem(fullKey(key), JSON.stringify(value));
      return Promise.resolve();
    },
    remove: function(key) {
      localStorage.removeItem(fullKey(key));
      return Promise.resolve();
    },
    list: function() {
      var results = [];
      for (var i = 0; i < localStorage.length; i++) {
        var k = localStorage.key(i);
        if (k && k.indexOf(prefix) === 0) {
          var unprefixed = k.slice(prefix.length);
          var raw = localStorage.getItem(k);
          if (raw) results.push({ key: unprefixed, data: JSON.parse(raw) });
        }
      }
      return Promise.resolve(results);
    },
    count: function() {
      var n = 0;
      for (var i = 0; i < localStorage.length; i++) {
        var k = localStorage.key(i);
        if (k && k.indexOf(prefix) === 0) n++;
      }
      return Promise.resolve(n);
    },
  };
}
```

---

## Implementation Notes

### Key format

Keys passed to `get`, `set`, `remove` are implementation-defined identifiers — UUIDs, sequential IDs, or normalised strings. The adapter is not responsible for key generation. Services manage their own key strategy.

### Serialisation

The adapter JSON-serialises all values on write and JSON-parses on read. Values must be JSON-serialisable (no functions, no circular references, no `undefined` top-level).

### Error handling

The reference implementation (localStorage) does not throw on read operations. On write, `localStorage.setItem` may throw `QuotaExceededError` if storage is full. Services should propagate this error rather than swallowing it.

### Atomicity

The adapter provides no transaction support. Multi-step operations (e.g. update record then update index) are not atomic. Services should design for this — the most recent write wins.

---

## Future Backends

When migrating to IndexedDB (for larger storage or KaiOS 2.x compatibility), the adapter is the only file that changes. All services remain unchanged because they program against the Promise-returning contract, not against `localStorage` directly.

A server-backed adapter (wrapping `fetch` to a gateway endpoint) is also possible with the same interface — the service layer does not need to know whether storage is local or remote.
