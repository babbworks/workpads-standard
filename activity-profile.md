# Activity Profile & Block Registry

**Status:** v0.1 — 2026-04-27
**Decision source:** decisions.md ARC011-1
**Supersedes:** "Business Profile" terminology throughout all prior specs
**Cross-references:** participants-block.md (IS_SENDER), financial-block.md (TAX_CODE defaults), personal-panel-design.md (Work tab)

---

## 1. Activity Profile

### 1.1 Concept

An Activity Profile is a named work context. A worker may have multiple concurrent activities — a registered business, a freelance sideline, employment under another sender. Each Activity carries its own identity, contact details, and financial defaults.

The worker designates one or more Activities as a **business** — this implies a formal name appears on records, and financial defaults (currency, tax) are set by the Activity rather than by global preference.

The active Activity feeds the IS_SENDER participant block in every new record created.

### 1.2 Storage

One entry per activity, one active pointer:

```javascript
// wp_activity_<id>
{
  id:                 string,   // "act_<timestamp><random4>"
  name:               string,   // "Alice's Plumbing" / "Handyman" / "BuildCo"
  type:               string,   // 'business' | 'employed' | 'freelance' | 'personal'
  isBusiness:         boolean,  // formal business designation (name appears on records)
  phone:              string?,
  whatsapp_confirmed: boolean,
  vatRegistered:      boolean,  // true → TAX_CODE default = 10 (standard rate)
  vatNumber:          string?,
  currency:           string    // 'GBP' | 'EUR' | 'USD' | 'local'
}

// wp_activity_active
string  // ID of the currently selected activity
```

### 1.3 Activity types

| Type | Meaning | Default tax | Record identity |
|------|---------|-------------|----------------|
| `business` | Registered business, formal invoicing | VAT if registered | Business name |
| `employed` | Worker under another employer | Per employer | Worker's name |
| `freelance` | Self-employed, informal | None by default | Worker's name |
| `personal` | Non-commercial work | None | Worker's name |

### 1.4 Activity picker in wizard

A compact "Activity:" row appears at the top of Wizard Screen 1, above the field list:

```
Activity: Alice's Plumbing [↕]
──────────────────────────────
Job:      [________________________]
Customer: [________________________]
```

CSK on the Activity row opens an overlay picker. Worker selects the activity for this record before filling fields. Defaults to `wp_activity_active`. Change persists only for this record — does not change the global active activity.

### 1.5 Effect on pads-v1 encoding

Activity Profile is local storage — no wire format change. At encode time:
- IS_SENDER participant: name + phone drawn from the active Activity
- Setup byte CURRENCY: from Activity's `currency` field
- Setup byte TAX_CODE: 10 (standard) if `vatRegistered = true`, else 00 (none) by default

### 1.6 Onboarding

First-launch creates a single Activity from the worker's name input. Type defaults to `freelance`. Worker can edit, add more Activities, or designate as a business from the Settings tab (Management screen).

---

## 2. Block Registry

### 2.1 Concept

The Block Registry is a typed list store of reusable field values. Each block type backs one or more wizard fields with suggestions. Values are saved once and suggested every time the backed field is filled.

Block Registry is **shared across all Activities** in v0.1. Per-activity block scoping is v0.2.

### 2.2 Block types (v0.1)

| Block | Storage key | Backed fields | Entry schema |
|-------|------------|--------------|-------------|
| Customer | `wp_block_customer` | `customer` (bit 1), `customer_phone` (bit 7) | `{ id, fn, tel?, flag }` |
| Location | `wp_block_location` | `location` (bit 3) | `{ id, label, address?, postcode?, intersection?, flag }` |

`flag`: `''` = complete, `'⚠'` = incomplete (e.g. name only, no phone).

### 2.3 Entry schemas

**Customer block entry:**
```javascript
{
  id:    string,   // "blk_<timestamp><random4>"
  fn:    string,   // display name (required)
  tel:   string?,  // phone (optional — cannot be forced)
  flag:  string    // '' | '⚠' (no phone)
}
```

**Location block entry:**
```javascript
{
  id:           string,
  label:        string,   // free text primary label (required)
  address:      string?,  // street address (optional, expanded field)
  postcode:     string?,  // postcode / zip (optional, expanded)
  intersection: string?,  // nearest junction (optional, expanded)
  flag:         string    // '' | '⚠' (label only, no structured address)
}
```

### 2.4 Save-to-block flow

Prompt fires **after** the record is saved or shared — never mid-wizard:

```
Saved: Fix boiler · Alice Smith

Save Alice Smith to your customers?
Phone: +44 7700 900123 (pre-filled if customer_phone was entered)

[CSK: Save]  [LSK: Skip]
```

If only a name was entered (no phone):
```
Save Alice to your customers?
(no phone number — you can add one later)

[CSK: Save]  [LSK: Skip]
```

Saved with `flag: '⚠'` if no phone. The `⚠` appears in the customer block list in the management screen.

### 2.5 Block suggestion overlay

When the worker navigates to a block-backed field, a subtle hint appears below the field:

```
Customer: [________________________]
          ▲ 3 saved customers
```

Pressing ↓ (or CSK when empty) opens the suggestion overlay:

```
┌──────────────────────────────┐
│ Customers                    │
│ ──────────────────────────── │
│ ● Alice Smith  +44 7700...   │
│   Bob Jones    +44 7711...   │
│   Carol A.     ⚠ no phone   │
└──────────────────────────────┘
  LSK: Back    CSK: Select
```

- Up/Down navigates entries
- CSK selects — fills customer field (and customer_phone field if `tel` present)
- LSK dismisses, returns to free text entry
- Worker can ignore the overlay entirely and type freely
- Max 5 entries visible; scroll if more

### 2.6 Expand field pattern

Location (and future block types) uses the expand field pattern:

```
Location: [Client warehouse      ][+]
```

The `[+]` button is a focusable element to the right of the field. Right D-pad moves focus to it. CSK activates expansion:

```
Location: [Client warehouse      ][−]
Address:  [123 High Street       ]
Postcode: [SW1A 1AA              ]
Junction: [High St / Mill Rd     ]
```

`[−]` collapses back. Structured sub-fields are all optional. The label alone is sufficient for block storage and field suggestions.

**This pattern applies consistently across the app wherever a simple field has optional depth:**
- Customer name `[+]` → phone, organisation
- Amount `[+]` → qty × rate breakdown
- Time `[+]` → separate start / end (when shown as a range)

### 2.7 Field-to-block mapping

| svc-basic v2 field | Bit | Block | Auto-fill behaviour |
|--------------------|-----|-------|---------------------|
| `customer` | 1 | Customer | Suggest from `fn` values |
| `customer_phone` | 7 | Customer | Auto-fills if selected customer has `tel` |
| `location` | 3 | Location | Suggest from `label` values |
| `worker` (legacy) | 8 | — | Pre-filled from active Activity name |
| `job` | 0 | — | No block; free text |
| `date` | 2 | — | Default today |

### 2.8 Participation linkage index

At record store time, the codec extracts participant and customer context and writes a thin linkage index entry. This enables the Work tab and Personal Panel to surface related records in O(1) without re-parsing all records.

```javascript
// wp_link_record_<record_id>
{
  customerId:  string?,  // Block Registry customer ID if matched
  locationId:  string?,  // Block Registry location ID if matched
  chainAnchor: string?,  // chain anchor hex if CHAIN bit set
  roles:       string[], // ['sender'] | ['worker'] | ['managed'] etc.
  timestamp:   number
}

// wp_link_customer_<block_id>
[ record_id, record_id, ... ]  // all records for this customer

// wp_link_location_<block_id>
[ record_id, record_id, ... ]  // all records at this location
```

These index entries are written by RecordService at store time (both on create and on receive-and-save). They are cheap to write and make all context lookups fast.

---

## 3. Management Screen — Activity & Block sections

### 3.1 Settings tab — Activities

```
Activities
──────────
● Alice's Plumbing (business, VAT)  [edit]
  Handyman                           [edit]
                              [+ New activity]
```

Active activity shown with ●. Edit: change name, type, phone, VAT status.

### 3.2 Records tab — Block Registry

```
Block Registry
──────────────
Customers   12 saved  [view] [clear]
Locations    8 saved  [view] [clear]
              ⚠ 3 incomplete entries
```

View opens a list of all entries for that block type with edit/delete per entry.

---

## 4. Implications for other specs

| Spec | Update needed |
|------|--------------|
| `personal-platform.md` | "Business Profile" → "Activity Profile" throughout |
| `participants-block.md` | IS_SENDER identity sourced from active Activity |
| `financial-block.md` | TAX_CODE default tied to Activity VAT registration |
| `pads-v2-encoding-spec.md` | Note: Activity Profile feeds encoder at encode time; no wire change |
| `softkeys.yaml` | Add: block-suggestion-overlay, activity-picker entries |
| Work tab | Activity filter column added |
