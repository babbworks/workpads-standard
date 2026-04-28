# Record Philosophy

**Status:** v0.1 — 2026-04-27
**Decision source:** decisions.md R-PADS-021
**Cross-references:** pads-v2-encoding-spec.md, chain-protocol.md, participants-block.md

---

## The Atomic Unit of Commerce

A workpad record is the atomic unit of a commercial relationship.

Not a row in a database. Not a form submission. Not a message thread. An atom — complete, portable, self-describing, and sovereign.

Business works through individual records: a quote, a job, an invoice, a confirmation, an expense, a handoff. These records are the real substance of commerce. Every folder system, every spreadsheet, every enterprise software suite is an attempt to organize these atoms — but the atoms themselves are what matter. When the software changes, the records remain. When the worker changes phones, the records travel with them. When the customer questions a charge three months later, the record is what resolves it.

Workpads starts from the record itself. Everything else — the app, the CLI, the encoding, the chain protocol — exists to serve the record.

---

## The Five Properties of a Workpad Record

### 1. Complete enough to stand alone

A recipient with no prior context must understand the work the record describes. This is not a goal for every field — it is a constraint on the design. When a worker sends a URL, they should not need to send a cover message explaining what it is. The record is the explanation.

This is why the svc-basic v2 template has a job field, a customer field, a date field, an actions list, a details field, and a story field — because those are the components of a complete account of a piece of work, not just a task title.

### 2. Compact enough to travel as a URL

Any URL-capable channel is a valid transport for a workpad record: SMS, WhatsApp, QR code, NFC tag, email, chat. This is not a technical constraint — it is a design philosophy. If a record cannot be shared by a worker on a feature phone with no data plan and a 160-character SMS limit, the record format has failed its most important user.

Every byte decision in pads-v1 encoding is a decision about what records mean. Compactness is not a performance optimization — it is an accessibility requirement.

### 3. Honest about its state

A record must carry its own state. Is it a draft? A pending invoice? A settled payment? A disputed record frozen at a point in time? The meta byte signals in pads-v1 encoding are not technical flags — they are truth claims about the record's condition.

`DRAFT` = this record is not yet complete; do not treat it as final.
`ACK_REQUEST` = the sender needs confirmation this was received.
`RESTRICT_FORWARD` = the sender's intent is that this stays between the two parties.
`CHAIN` = this record belongs to a sequence; context exists.

When a record lies about its state — when a draft is presented as final, or a past transaction is unmarked as settled — trust breaks down. The encoding exists to make honest records easy and dishonest records structurally difficult.

### 4. Sovereign

The worker controls what is in the record, who sees what version, and whether the record chains to prior records.

Progressive disclosure at URL generation time (the Customer/Colleague toggle) means the worker makes a conscious choice about what information leaves their device. Internal margin amounts, internal notes, worker cost calculations — these are in the record locally but may never appear in a URL. The worker's device is the sole arbiter of what gets encoded.

Record sovereignty also means the worker can revoke nothing once shared — URLs persist. This is an honest constraint, stated plainly: share only what you intend the recipient to hold. The encoding does not prevent screenshots; it prevents accidental oversharing.

### 5. Able to speak to other records via chains

A single record is a snapshot. A chain of records is a relationship.

The invoice that follows the quote that followed the initial enquiry — these are the same commercial relationship expressed across time. The chain protocol in pads-v1 does not create a database of relationships; it enables records to reference one another by a chain anchor, letting any viewer reconstruct the thread from the records they hold.

Records speak to one another. A State Commit record says: "at this moment, the relationship is in this state — both parties agree." An Amendment record says: "the prior record is superseded in these fields." An ACK record says: "I received this; I acknowledge the exchange."

---

## What a Record Is Not

A workpad record is not a contract. It is not legally binding by itself. It is not a replacement for an invoice system, a payroll system, or an accounting ledger. It is not a real-time communication.

It is a document — in the original sense of the word: something that documents. It documents a piece of work, the parties involved, the terms agreed, the time spent, the amounts exchanged. Whether it becomes evidence in a dispute, a record in a ledger, or simply a worker's own memory of what they did — that is the record's life after it leaves the phone.

---

## Implications for Future Design

Every future extension to the pads encoding format, the template set, or the UX should be evaluated against these five properties.

A field that cannot be expressed in a URL is not a workpad field — it may be a backend feature, a local note, or a future media attachment, but it is not part of the record's core structure.

A feature that makes records harder to share (more steps, longer URLs, more required fields) works against the design. A feature that makes records more honest, more complete, or more sovereign works with it.

The record is the product. Everything else is infrastructure.

---

## Lineage

This philosophy is grounded in the earliest workpads standard documents (`original/guiding-principles/`), BitLedger's approach to portable financial records, and the BitPads protocol's commitment to minimal, machine-readable, URL-transmissible data encoding. The pads-v1 encoding format is the technical expression of this philosophy.

See also:
- `pads-v2-encoding-spec.md` — the wire format
- `chain-protocol.md` — how records speak to one another
- `original/guiding-principles/record-sovereignty.md` — sovereignty in depth
- `original/guiding-principles/record-life.md` — the lifecycle of a record
