# Workpads Standard

**Status:** v0.1 — complete (2026-04-27)
**Maintained by:** Babb
**BASICS relationship:** Conformant with BASICS v0.1.1 — naming surface flexibility documented in §3

---

This repository holds the formal Workpads Standard: the normative specification for all Workpads implementations (KaiOS, Android, iOS, CLI, browser).

Design decisions are made during research-quiz rounds and logged in `workpads-basicsconform/system/logs/decisions.md`. Once a decision is settled, it is documented here as a formal standard section. This repository is the definitive home for what Workpads is and how it works — not `workpads-basicsconform`, which is the research control plane.

---

## Contents

| Section | File | Status |
|---------|------|--------|
| §1 — PADS Data Model | `pads-model.md` | v1 (2026-04-27) |
| §2 — Record Schema (svc-basic) | `record-schema.md` | v1 (2026-04-27) |
| §3 — Naming Convention | `naming-convention.md` | v1 (2026-04-26) |
| §4 — RecordService Interface | `record-service.md` | v1 (2026-04-27) |
| §5 — Codec & Compact Encoding | `codec.md` | v1 (2026-04-27) |
| §6 — Storage Adapter Contract | `storage-adapter.md` | v1 (2026-04-27) |
| §7 — Two-Build Strategy | `build-strategy.md` | v1 (2026-04-27) |
| §8 — BASICS Conformance Mapping | `basics-conformance.md` | v1 (2026-04-27) |

All 8 sections are complete for v0.1.

---

## Additional Documents

These documents were produced during the design process and are part of the standard but outside the 8 numbered sections:

| File | Description |
|------|-------------|
| `record-philosophy.md` | The five properties of a workpad record — foundational design rationale |
| `personal-platform.md` | The two-engine architecture: Exchange Engine + Learning Engine |
| `panel-access-model.md` | ArrowLeft/ArrowRight panel navigation model |
| `activity-profile.md` | Activity Profile and Block Registry specification |
| `pads-v2-encoding-spec.md` | Extended bitpad-v2 encoding specification (forward design) |
| `chain-protocol.md` | Record chain protocol — how records reference each other |
| `participants-block.md` | Participants block specification |
| `financial-block.md` | Financial block specification |
| `transaction-classification.md` | Transaction classification system |
| `bitpad-record-encoding.md` | Alternative bitpad record encoding specification |

---

## Relationship to Other Repos

| Repo | Role |
|------|------|
| `workpads-standard/` (this repo) | **Live normative standard.** All settled specifications live here. |
| `workpads-basicsconform/` | Research control plane — quiz rounds, task cards, decision log, REFS. Feeds this repo. |
| `workpads/` | Original architecture documents and protocol design notes. Historical reference. |
| `workpadskaios/` | Reference implementation — KaiOS app. Implements the standard. |
| `workpads-codec/` | `@workpads/codec` npm package — implements §5. |
| `workpads-cli/` | CLI reference implementation. Implements §4 (RecordService surface). |

The `original/` subfolder contains a separate git repository with the original Workpads Standard drafts (pre-2026). Those drafts informed but do not supersede the sections in this repo.

---

## Versioning

Section versions use `v1`, `v2`, etc. A section version increments when a breaking change is made to the specification (e.g. a field is removed, an encoding slot is reassigned). Additive changes (new optional fields, new sections) do not increment existing section versions.

The overall standard version (`v0.1`) tracks the implementation generation. It aligns with the app version: Workpads KaiOS v0.1 implements Workpads Standard v0.1.
