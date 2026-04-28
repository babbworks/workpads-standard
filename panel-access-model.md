# Panel Access Model

**Status:** v0.1 — 2026-04-27
**Decision source:** decisions.md ARC016-1
**Supersedes:** R4-2 (ArrowLeft = Personal Panel), R9-4 (same) — both superseded by R9-ADD-5
**Cross-references:** personal-panel-design.md, activity-profile.md, personal-platform.md, softkeys.yaml

---

## 1. Two Universal Panels

The workpads app has two app-wide overlay panels, each accessible via a D-pad trigger from any supported screen:

| Panel | Trigger | Slides from | Role |
|-------|---------|-------------|------|
| Workpads Panel | ArrowLeft | Left | Exchange Engine context layer — record preview, customer history, share events |
| Personal Panel | ArrowRight | Right | Learning Engine layer — personal captures, Quick Note, field-level capture |

Both panels are 80% screen width (192px on a 240px display). The remaining 20% (48px) of the underlying screen stays visible as a visual underlay — confirming the worker's position without closing the panel.

---

## 2. Screen Access Matrix

| Screen | Workpads Panel (ArrowLeft) | Personal Panel (ArrowRight) |
|--------|---------------------------|----------------------------|
| Main list | Full panel — live record preview | Full panel — Learning Engine |
| Record view | Full panel — this record's context | Full panel — captures for this record |
| Wizard screens 1–4 | 80% overlay from left | 80% overlay from right |
| Share screen | Full panel — share context | Full panel — send-event capture |
| Actions sub-screen | Suppressed | Suppressed |
| Dialogs / modals | Suppressed | Suppressed |
| Management screen | Suppressed | Suppressed |

On the management screen, both panels are suppressed. D-pad Left/Right navigates the `Records | Personal | Settings` tabs directly — no panel trigger conflict.

---

## 3. Trigger and Dismiss Behaviour

```
ArrowLeft (not in panel)   → open Workpads Panel (if guard passes)
ArrowLeft (Workpads Panel open) → close Workpads Panel
ArrowRight (Workpads Panel open) → close Workpads Panel
  — worker presses ArrowRight again from main screen to open Personal Panel

ArrowRight (not in panel)  → open Personal Panel (if guard passes)
ArrowRight (Personal Panel open) → close Personal Panel
ArrowLeft (Personal Panel open) → close Personal Panel
  — worker presses ArrowLeft again from main screen to open Workpads Panel

BACK (either panel open)   → close panel, return to prior position
```

**No simultaneous panels.** Opposite arrow closes the current panel — it does not switch to the other panel in one gesture. The worker closes first, then opens the other. This is intentional: the two panels serve different purposes; switching in a single gesture risks accidental context loss.

### 3.1 Trigger guard

Both triggers check the same guard before opening:

```javascript
function shouldOpenPanel(e) {
  var el = document.activeElement;
  if (el && (el.tagName === 'INPUT' || el.tagName === 'TEXTAREA')) return false;
  if (el && el.dataset.navAxis === 'horizontal') return false;
  if (currentScreen.panelsSuppressed) return false;
  return true;
}

document.addEventListener('keydown', function(e) {
  if (e.key === 'ArrowLeft') {
    if (personalPanelOpen) { closePersonalPanel(); return; }
    if (workpadsPanelOpen) { closeWorkpadsPanel(); return; }
    if (shouldOpenPanel(e)) { openWorkpadsPanel(); }
  }
  if (e.key === 'ArrowRight') {
    if (workpadsPanelOpen) { closeWorkpadsPanel(); return; }
    if (personalPanelOpen) { closePersonalPanel(); return; }
    if (shouldOpenPanel(e)) { openPersonalPanel(); }
  }
});
```

`data-nav-axis="horizontal"` must be applied to all tab bar items and any horizontal carousel or picker elements.

---

## 4. Workpads Panel — Content Model

### 4.1 Panel header (permanent)

```
┌──────────────────────────────┐
│ ⬛ Workpads               ← │
├──────────────────────────────┤
│  [context content]           │
```

The `←` glyph is a visual affordance only — not a softkey label. The header row is always present. CSK on the header row has no action in v0.1 (future: navigate to Records management tab).

### 4.2 Main list — list header focused (no record focused)

Aggregate view. Shown when focus is on the list header row or the density selector, not on a specific record.

```
┌──────────────────────────────┐
│ ⬛ Workpads               ← │
├──────────────────────────────┤
│ This week                    │
│ 4 sent · 1 received          │
│ 1 pending response           │
├──────────────────────────────┤
│ Apr 27 · Fix boiler    24    │
│ Apr 26 · Annual svc          │
│ Apr 24 · Flat roof     14    │
└──────────────────────────────┘
```

- Summary row: records sent + received this week, pending response count
- Last 3 records with date and abbreviated job title
- Numbers after job title = count of personal captures linked to that record (Learning Engine bridge)
- No amount shown in aggregate view — amounts are record-level detail

### 4.3 Main list — record focused (live tracking)

As the worker navigates Up/Down through the record list, the Workpads Panel tracks the focused record and updates its content. This makes the panel a live preview pane.

```
┌──────────────────────────────┐
│ ⬛ Workpads               ← │
├──────────────────────────────┤
│ Fix boiler                   │  ← focused record title
│ Apr 27 · Alice Smith         │  ← date · customer
│ 24 sent · £95 pending        │  ← role + amount + status
├──────────────────────────────┤
│ Shared: Apr 27 14:03         │  ← most recent share event
│ No response recorded         │
├──────────────────────────────┤
│ 2 captures linked        24  │  ← Learning Engine bridge
└──────────────────────────────┘
```

- Role indicator: 24 = sent (↑), 14 = received (↓) — same compact notation as Work tab
- Amount + status: `£95 pending` / `£95 settled` / (blank if no financial fields)
- Share history: most recent event + response status. Up to 3 events shown if scroll available.
- Captures linked: count of personal captures with `linkedRecordId` matching this record. Tapping into Personal Panel from here will filter to these captures.
- Update latency: panel content updates on `focus` event of each list item — no debounce needed (D-pad navigation is sequential).

### 4.4 Record view

Full context for the open record. More share history visible than preview mode.

```
┌──────────────────────────────┐
│ ⬛ Workpads               ← │
├──────────────────────────────┤
│ 24 Sent · Alice Smith        │
│ Apr 27 · £95 pending         │
├──────────────────────────────┤
│ Shares (2)                   │
│ Apr 27 14:03 — no response   │
│ Apr 28 09:11 — resent        │
│ Apr 28 11:30 — viewed (est)  │
├──────────────────────────────┤
│ No chain                     │
├──────────────────────────────┤
│ 2 captures linked            │
└──────────────────────────────┘
```

- Up to 3 share events shown (last 3 if more exist)
- Chain status: "No chain" / "Chain: 3 records" with anchor ID if chained
- Captures linked: same bridge signal as list preview

### 4.5 Wizard (screens 1–4)

80% overlay from left. 20% of wizard remains visible at the right edge as underlay. The field label or content currently focused in the wizard is readable in the 20% strip — no label needed inside the panel.

**Screen 1 (Process) — before customer field populated:**
```
┌──────────────────────────────┐
│ ⬛ Workpads               ← │
├──────────────────────────────┤
│ New record                   │
│ svc-basic · 3 required fields│
│ Job · Customer · Date        │
└──────────────────────────────┘
```

Field guidance: required fields listed. Template name suppressed — appears in the app About screen, not here. Field names only.

**Screen 1 — after customer field populated:**
```
┌──────────────────────────────┐
│ ⬛ Workpads               ← │
├──────────────────────────────┤
│ Alice Smith                  │
│ Last 2 jobs:                 │
│ Apr 20 · Fix boiler    £95   │
│ Apr 12 · Annual svc    £80   │
├──────────────────────────────┤
│ 5 captures · this customer   │  ← Learning Engine bridge
└──────────────────────────────┘
```

Customer context replaces the field guidance as soon as the customer field is non-empty. Past jobs for this customer sourced from `wp_link_customer_<block_id>` if the customer matches a block entry; otherwise from text-match against `customer` fields in stored records.

**Screen 2–4:**
```
┌──────────────────────────────┐
│ ⬛ Workpads               ← │
├──────────────────────────────┤
│ Alice Smith                  │
│ Fix boiler · Apr 27          │  ← in-progress record summary
├──────────────────────────────┤
│ 3 required fields complete   │
│ Actions: 2 added             │
└──────────────────────────────┘
```

Record-in-progress summary: customer + job (once filled), completion signal for required fields, action count.

### 4.6 Share screen

```
┌──────────────────────────────┐
│ ⬛ Workpads               ← │
├──────────────────────────────┤
│ Sending as:                  │
│ Alice's Plumbing             │  ← active Activity name
├──────────────────────────────┤
│ Fix boiler · £95             │  ← payload summary
│ 1 action · Apr 27            │
├──────────────────────────────┤
│ Prior shares: none           │
└──────────────────────────────┘
```

- Activity confirmation: "Sending as: [Activity name]" — reminds worker which identity is on the record
- Payload summary: job title, amount (if present), action count, date
- Prior share events for this record (if any resends)

---

## 5. Personal Panel — Workpads Panel Bridge

When the Workpads Panel shows "N captures linked" for a focused record, the worker can:
1. Close the Workpads Panel (ArrowRight or BACK)
2. Open the Personal Panel (ArrowRight again)

The Personal Panel detects that the last Workpads Panel context was a specific record and pre-filters its view to captures linked to that record ID. This is a soft bridge — not automatic, but the context carries across the dismiss-and-open sequence.

The bridge signal is stored in a transient variable `lastWorkpadsContext` (not persisted):

```javascript
var lastWorkpadsContext = null;  // { recordId, customerId }

function openWorkpadsPanel() {
  lastWorkpadsContext = getCurrentRecordContext();
  // ... render panel
}

function openPersonalPanel() {
  if (lastWorkpadsContext && lastWorkpadsContext.recordId) {
    // pre-filter Personal Panel to captures for this record
    renderPersonalPanel({ filterRecordId: lastWorkpadsContext.recordId });
  } else {
    renderPersonalPanel({});
  }
}
```

If the worker opens the Personal Panel without having opened the Workpads Panel first, it renders unfiltered (standard behaviour).

---

## 6. Wizard Underlay Behaviour

When either panel opens over the wizard, the 20% underlay strip remains visible:

- **Workpads Panel (left):** underlay is the right 20% of the wizard. The currently focused field's label or input content is partially visible. No additional context label needed inside the panel.
- **Personal Panel (right):** underlay is the left 20% of the wizard. The field label of the focused field appears in the strip. The field name is readable at normal orientation — the narrow strip is sufficient for a single field label word ("Customer", "Job", "Date").

```
[Workpads Panel 80%][20% wizard right edge]   ← Workpads Panel open
[20% wizard left edge][Personal Panel 80%]    ← Personal Panel open
```

The underlay is rendered at full opacity — it is not dimmed. The panel itself carries a left or right border to visually separate it from the underlay. The worker knows they are in a panel state and where to return.

---

## 7. Workpads Panel Softkey Map

While the Workpads Panel is open:

```yaml
workpads-panel:
  lsk:
    label: Close
    action: nav.close-panel
  csk:
    label: Open
    action: record.open          # opens focused record (from list context)
  rsk: null
```

CSK "Open" is contextual — only meaningful when a specific record is previewed. On wizard/share contexts, CSK is null or contextual to the panel content.

---

## 8. Navigation Summary

```
Main list (record A focused)
  ArrowLeft                → Workpads Panel opens, shows record A preview
    Up/Down (list nav)     → panel updates live to focused record
    ArrowRight             → close Workpads Panel
      ArrowRight again     → Personal Panel opens
    BACK                   → close Workpads Panel
    CSK                    → open focused record

  ArrowRight               → Personal Panel opens
    ArrowLeft              → close Personal Panel
      ArrowLeft again      → Workpads Panel opens
    BACK                   → close Personal Panel

Wizard (Screen 1, Customer field focused)
  ArrowLeft                → Workpads Panel opens (field guidance / customer context)
    20% right underlay shows Customer field
    ArrowRight or BACK     → close panel, return to Customer field
  ArrowRight               → Personal Panel opens (field-level capture)
    20% left underlay shows Customer field label
    ArrowLeft or BACK      → close panel, return to Customer field

Management screen
  ArrowLeft                → tab navigation (no panel)
  ArrowRight               → tab navigation (no panel)
```
