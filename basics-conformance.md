# §8 — BASICS Conformance Mapping

**Status:** v1 (settled 2026-04-27)
**Decision sources:** decisions.md R3-1 through R3-10, R7-3, R7-5, R7-9
**BASICS version:** 0.1.1
**Claim file:** `workpads/conformance-tests/BASICS-claim.yaml`
**Evidence location:** `workpads/conformance-tests/`

---

## Conformance Claim

| Property | Value |
|----------|-------|
| Tier | **Core** |
| Profiles | `shared-core`, `software` |
| Claim status | v0.1-draft (pending `compatibility-policy.md`) |
| Claim date | 2026-04-26 |
| Verified repos | `workpads/` (reference arch) + `workpadskaios/` (app) — both must pass dirty-test |

The Core tier claim covers all BASICS-SC-001 through BASICS-SC-061 mandatory rules applicable to a software product. The `software` profile adds SW-001 through SW-041 requirements. Field tier rules (SC-042, SC-043 — sync and conflict) are explicitly deferred to v0.2 as the product is offline-only in v0.1.

---

## Evidence Artifacts

All conformance evidence lives in `workpads/conformance-tests/`:

| Artifact | Rules Covered | Status |
|----------|--------------|--------|
| `command-surface.md` | BASICS-EVID-001, SC-001, SC-002, SC-012 | Present |
| `event-schema.md` | BASICS-EVID-002, SC-050, SW-002 | Present |
| `deviation-registry.md` | BASICS-EVID-004, SC-055 | Present |
| `degraded-mode-matrix.md` | BASICS-EVID-005, SC-040, SC-041 | Present |
| `BASICS-claim.yaml` | Claim declaration | Present (draft) |
| `compatibility-policy.md` | BASICS-EVID-004, SC-051, SW-003 | **Pending** |

`compatibility-policy.md` is deferred until after v0.1 ships. A compatibility policy gains credibility when it can cite a real version history. The codec `v=1` URL parameter is the first worked example — its policy statement is more meaningful once v0.2 (which must handle v0.1 records) exists.

---

## Rule-by-Rule Mapping

### Satisfied Rules

**BASICS-SC-001 — Command surface defined**
The workpads command surface is documented in `command-surface.md`. GUI actions (create, view, edit, share, import, list, delete, archive) and their key bindings are enumerated. The naming deviation (DEV-WP-001) is registered — see §Deviations below.

**BASICS-SC-002 — Command surface versioned**
`command-surface.md` is versioned (v0.1) and will increment on breaking changes to the command vocabulary.

**BASICS-SC-012 — Integration endpoint declared**
The workpad URL schema (`https://workpads.me/p#alg=bitpad-v1&v=1&d=<payload>`) is declared as the integration endpoint in `command-surface.md`. Any tool that can handle a URL can participate in a workpad exchange. This satisfies SC-052 (integration endpoint must be formally declared).

**BASICS-SC-040 — Degraded mode defined**
`degraded-mode-matrix.md` documents workpads behaviour when storage is near-full, when an incoming URL is malformed, when encoding fails, and when the device is offline. Degraded mode is offline-always for v0.1 — the app is designed for offline-first operation, not as a degraded state.

**BASICS-SC-041 — Local create/edit confirmed offline**
Records can be created, edited, saved, and shared entirely without a network connection. The bitpad-v1 codec operates in-memory. localStorage is the only storage backend and requires no network. Confirmed by dirty-test baseline run (2026-04-20).

**BASICS-SC-050 — Event schema defined**
`event-schema.md` documents the events emitted by the workpads application layer: record:created, record:updated, record:archived, record:received, note:captured, profile:updated. Events are not yet emitted on a bus (v0.1 uses direct function calls) but the schema defines the interface for future wiring.

**BASICS-SC-055 — Deviation registration enabled**
Registered deviations exist in `deviation-registry.md`. The naming convention (DEV-WP-001) is the first registered deviation.

**SW-001 — Service ownership documented**
`workpads/architecture/services/` documents all service contracts: GatewayService, RecordService, CodecService, LinkService, TemplateRegistryService, CommentService, StoragePolicy. Ownership boundaries and request/response shapes are defined.

**SW-002 — Event schema present**
`event-schema.md` is present. Codec version flag (`v=1`) is the in-band compat signal at the data layer.

**SW-010 — Offline create/edit**
SC-041 satisfied. Identical evidence.

**SW-011 — Durable mutation handling**
Auto-save triggers on every PADS wizard screen transition (decision R1-8). If the app crashes mid-wizard, the partially-filled record is recoverable from localStorage on next launch. Crash safety is inherent to localStorage's synchronous write model.

**SW-012 — Storage failure surfaced**
`QuotaExceededError` from localStorage is caught and surfaced as a user-visible error, blocking further wizard navigation until space is freed or the record is deleted. Not silently swallowed.

**SW-030 — Secure defaults (SSDF minimal)**
- Offline-only operation: no network calls means no server-side attack surface in v0.1
- No authentication surface: no passwords, tokens, or session state to mishandle
- All data is user-local: no exfiltration path through the app itself
- Dependency surface: fflate (~8 KB, MIT, no network calls) + optional QR library

**SW-031 — No known critical/high CVEs**
No known vulnerabilities in fflate 0.8.2 or the app's own code at v0.1 claim date.

**SW-032 — Safe by default**
Offline-only operation is the safest possible default. No network, no auth, no remote storage.

**SW-040 — SBOM**
`package.json` lists all npm dependencies. A formal SPDX SBOM is generated at release via `npm sbom --sbom-format spdx`.

**SW-041 — Build provenance**
`build-provenance.md` documents the build environment at SLSA Provenance Level 1: OS, Node version, fflate version, build date, and build commands used.

---

### Deferred Rules (explicit, documented)

These rules are deferred to v0.2. The deferral is intentional and documented in `degraded-mode-matrix.md` and `BASICS-claim.yaml` — not an undocumented gap.

| Rule | Reason | Target version |
|------|--------|---------------|
| SC-042 — Sync conflict detection | v0.1 is offline-only; no sync exists | v0.2 |
| SC-043 — Sync conflict resolution | Same as above | v0.2 |
| SC-051 — Machine-readable compat declaration | Requires shipped version history; deferred until v0.2 exists | v0.2 |
| SW-013 — Sync status visible | No sync in v0.1; status N/A | v0.2 |
| SW-020 — Conflict detection on sync | No sync | v0.2 |
| SW-021 — Conflict resolution | No sync | v0.2 |
| SW-003 — Compatibility policy published | Part of SC-051 deferral | v0.2 |

The sync-related rules (SC-042, SC-043, SW-020, SW-021) are Field tier requirements in BASICS (BASICS-TIER-011). Workpads claims Core tier for v0.1. Field tier is the planned target for v0.2 once sync via gateway is implemented.

---

### Not Applicable (N/A)

| Rule | Reason |
|------|--------|
| Any hardware/firmware rules | Workpads is a pure software product |
| Server-side rules | v0.1 has no server component |
| Auth/session rules | No authentication in v0.1 |

---

## Deviations

### DEV-WP-001 — Naming surface convention

| Property | Value |
|----------|-------|
| ID | DEV-WP-001 |
| Applies to | BASICS-SC-001 (command surface / naming) |
| Enabling rule | BASICS-SC-055 |
| Status | Registered in `deviation-registry.md` |

**Description:** Workpads field names, method names, and identifiers correspond to BASICS standard concepts but use concise, role-neutral synonyms or short forms appropriate to the application context.

Examples:
- `worker` (not `technician_name` or `assigned_operator`)
- `customer` (not `recipient_entity`)
- `job` (not `work_order_descriptor`)
- `location` (not `deployment_site`)

This is not a semantic deviation — the BASICS concept is preserved. It is a naming surface decision. A full mapping table connecting every workpads name to its BASICS counterpart exists in `workpads-basicsconform/system/config/basics-naming-commentary.md`.

**Rationale:** BASICS is a general standard spanning hardware, firmware, and software. Its naming favors descriptive precision. Workpads serves field workers (plumbers, nurses, delivery drivers, HVAC technicians) who type on D-pad phones. Names must be short, typeable, memorable, and role-neutral. Imposing BASICS-style technical naming on these users would be a UX failure.

SC-055 explicitly enables this pattern: naming deviations are permitted when the BASICS semantics are preserved and the deviation is formally registered with a mapping document.

---

## In-Band Compatibility Signals

Two compatibility signals are embedded in the wire format (not as separate artifacts):

**Signal 1 — URL schema version (`v=`):**
```
alg=bitpad-v1&v=1&d=<payload>
```
`v=1` identifies the URL schema version. When a v0.2 decoder encounters a `v=1` URL, it knows exactly which field order, flag bit assignments, and compression parameters to use.

**Signal 2 — Field signature (bitpad-v1 frame byte 0):**
The first byte of the decompressed binary frame is the template identifier (`0x01` = svc-basic). Future decoders can detect an unknown template byte and reject gracefully rather than partially decoding.

These two signals together provide forward compatibility: a v0.2 client can read v0.1 records because the template byte and version parameter fully specify the encoding.

---

## Dirty-Test Pass Requirement

Per decision R3-9, **both** the following repos must pass `basics-cli dirty-test` before the Core tier claim is considered verified:

1. `workpads/` — reference architecture conformance (validates design)
2. `workpadskaios/` — app implementation conformance (validates code)

Divergence between design and implementation is a quality risk. Requiring both repos to pass catches it.

The dirty-test baseline on `workpads/` was run 2026-04-20 and established the initial conformance state. A clean run against `workpadskaios/` is TASK-CONF-003 (deferred until the app repo has a stable evidence directory structure).

---

## Conformance Claim Finalisation

The v0.1-draft claim in `BASICS-claim.yaml` becomes a final claim when:

1. `compatibility-policy.md` is written (post-v0.1 ship)
2. `workpadskaios/` passes dirty-test
3. `BASICS-claim.yaml` `claimVersion` is updated from `v0.1-draft` to `v0.1`

Expected timeline: Q3 2026, at the v0.2 kickoff.

---

## Relationship to workpads-standard

This section documents the conformance mapping — which BASICS rules the workpads design satisfies, how, and where the evidence lives. It is not a substitute for the BASICS evidence artifacts themselves.

The canonical claim is `workpads/conformance-tests/BASICS-claim.yaml`. This document is the human-readable explanation of how the Workpads design maps to BASICS, for implementors building new Workpads clients who need to understand the conformance requirements they are inheriting.

Any new Workpads implementation (Android app, web app, second CLI) must satisfy the same rules mapped here. The evidence artifacts it produces must be stored in its own `conformance-tests/` directory and pass dirty-test before it can claim Core tier independently.
