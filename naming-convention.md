# §3 — Naming Convention

**Status:** v1 — established 2026-04-26
**Decision source:** `workpads-basicsconform/system/logs/decisions.md` — R2-META, R2-META-RESOLVED
**Detail:** `workpads-basicsconform/system/config/basics-naming-commentary.md`

---

## Principle

Workpads field names, method names, and identifiers **correspond to BASICS standard concepts** but use **concise, role-neutral synonyms or short forms** appropriate to the application context.

This is not a deviation from BASICS semantics. It is a naming surface decision. A published mapping table (see link above) connects every workpads name to its BASICS counterpart.

**Rationale:** BASICS is a general standard spanning hardware, firmware, software, and operational extensions. Its naming tends toward descriptive precision. Workpads serves field workers on constrained devices — names must be short, typeable, memorable, and free of role assumptions. A plumber, nurse, delivery driver, and HVAC technician all use workpads; none should see a field labelled "technician_name".

---

## Rules

1. **Check BASICS first** — document the BASICS concept this field corresponds to.
2. **Prefer common English over technical jargon** — `worker` not `technician`, `customer` not `recipient_entity`.
3. **Prefer single words** — `location` not `job_site_address`.
4. **Compact keys: 1–3 chars, no collision** — check `basics-naming-commentary.md` before assigning.
5. **Role-neutral** — the name must apply equally to an HVAC tech, nurse, driver, and inspector.

---

## Current Field Registry (svc-basic v2)

| Workpads Field | Compact Key | BASICS Concept |
|----------------|-------------|----------------|
| `job` | `j` | command / work-order descriptor |
| `customer` | `c` | operator-designated recipient |
| `date` | `d` | record timestamp / event date |
| `location` | `l` | deployment site / operator context |
| `meeting_time` | `mt` | scheduled activation time |
| `start_time` | `st` | work commencement timestamp |
| `end_time` | `et` | work completion timestamp |
| `customer_phone` | `cp` | operator contact reference |
| `worker` | `w` | assigned operator / executing agent |
| `actions[].title` | `at` | command sequence step |
| `actions[].notes` | `an` | command sequence step notes |
| `details` | `de` | field observation record |
| `story` | `sy` | narrative evidence record |

---

## BASICS Amendment Proposal

This naming approach is a candidate for a formal provision in BASICS-SC-001 permitting conformant products to use synonymous short-form names provided a published mapping table exists. See `basics-naming-commentary.md` §BASICS Amendment / Deviation Status.
