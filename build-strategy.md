# §7 — Two-Build Strategy

**Status:** v1 (settled 2026-04-27)
**Decision sources:** decisions.md R1-3, R1-4, R1-9, R1-10
**Applies to:** workpadskaios (KaiOS app builds)

---

## Overview

Workpads KaiOS targets two distinct platform generations with meaningfully different JavaScript engines, Web API support, and manifest formats. Rather than managing these differences at runtime via feature detection guards, the architecture uses **two separate builds** — one per platform generation.

This is not a compile-time branching hack. It reflects a fundamental difference in the platform contracts:

| Property | KaiOS 2.x | KaiOS 3.x |
|----------|-----------|-----------|
| Gecko version | ~48 | ~84 |
| ES module support | No | Yes |
| Promise support | Partial (DOMRequest-based APIs) | Full |
| Clipboard API | `document.execCommand('copy')` | `navigator.clipboard.writeText()` |
| App origin | `app://` | `https://<name>.localhost/` |
| Service Worker | No | Yes |
| Manifest format | `manifest.webapp` (Mozilla) | `manifest.webmanifest` (W3C) |
| Packaging | ZIP with manifest.webapp | ZIP with manifest.webmanifest |

Managing these differences with a single codebase of `if (gecko48) ... else ...` guards makes the code harder to read, harder to test, and harder to reason about. The split is real and architectural.

---

## Current State (v0.1)

The v0.1 KaiOS app targets **KaiOS 3.x only**. It is:

- Vanilla JavaScript, written in ES5-compatible syntax throughout — no transpile step required
- A plain HTML/CSS/JS directory, loaded directly by the browser or packaged as a KaiOS app
- No build tool, no Babel, no bundler
- Uses `manifest.webmanifest` (KaiOS 3.x format)
- Includes `browser-dev.js` for desktop browser development (D-pad emulator) — this file is excluded from device builds

The "two-build strategy" in v0.1 manifests as the distinction between the **browser dev build** and the **device build**:

| Build | Contents | Audience |
|-------|----------|---------|
| Device build | `index.html`, `css/`, `js/` (minus browser-dev.js), `manifest.webmanifest` → ZIP | KaiOS 3.x device |
| Browser dev build | Same + `js/lib/browser-dev.js` loaded | Desktop browser for development |

The `browser-dev.js` file provides:
- On-screen D-pad button overlay
- Keyboard arrow key mapping for desktop navigation testing
- Visual soft key labels that mirror the KaiOS soft key bar

It is loaded via the last `<script>` tag in `index.html` and has no effect on app behaviour — it only adds navigation surface. Remove this tag when packaging for device.

---

## Target State (v0.2+)

From v0.2, the two-build strategy expands to cover both KaiOS versions with a proper build pipeline:

### Build Targets

**Build: KaiOS 3.x** (`npm run build:3.x`)
- Source: ES6+ (modern JS)
- Transpile: Babel targeting Gecko 84 (Firefox 84 compat)
- Manifest: `manifest.webmanifest`
- Clipboard: `navigator.clipboard.writeText()` (with execCommand fallback)
- Storage: localStorage (v0.2 may introduce IndexedDB)
- Output: `dist/kaios3/` → ZIP

**Build: KaiOS 2.x** (`npm run build:2.x`)
- Source: Same shared core
- Transpile: Babel targeting Gecko 48 (Firefox 48 compat, ES5 output)
- Manifest: `manifest.webapp`
- Clipboard: `document.execCommand('copy')` only
- Storage: localStorage
- Output: `dist/kaios2/` → ZIP

### Shared Core

All business logic lives in `src/core/` and is shared between builds:
- `RecordService` — CRUD, URL encode/decode
- `ActivityService` — identity
- `BlockRegistry` — contacts
- `PersonalService` — quick notes
- `WPCodec` — pads-v1 encoder/decoder

### Platform Adapters

Platform-specific differences are isolated into adapters in `src/adapters/<platform>/`:
- `clipboard.js` — clipboard write (navigator.clipboard vs execCommand)
- `storage.js` — storage backend (localStorage, future IndexedDB)
- `navigation.js` — key event differences between Gecko versions
- `manifest.json` (or .webapp) — built-appropriate manifest

The build script swaps in the correct adapter set per target.

### Source Language

Write all source in **ES6+**. Babel handles transpilation. This enables:
- `const`/`let`, arrow functions, template literals, spread, destructuring
- `async`/`await` (Babel transforms to generator-based async for Gecko 48)
- ES modules (bundled for Gecko 48 which lacks module support)

Vanilla JS only — no framework. No Preact, no Vue, no React. The KaiOS screen constraint and the form-focused interaction model do not justify framework overhead.

---

## workpads-dev CLI (v0.2 toolchain)

A lightweight build orchestration CLI (`workpads-dev`) is planned for v0.2 to manage the two-build lifecycle:

```bash
workpads-dev build --target kaios3
workpads-dev build --target kaios2
workpads-dev package --target kaios3 --out dist/workpads-3.x.zip
workpads-dev set-keys --profile kaios3
```

`workpads-dev` is a developer tool, not a product CLI. It lives in `workpadskaios/tools/workpads-dev/` and is not shipped to users.

The soft key mapping is stored in `softkeys.yaml` — a control file that `workpads-dev` reads and applies per build target. Different targets may have different soft key labels or assignments (e.g. if KaiOS 2.x has different key event names for the soft keys).

---

## fflate Compatibility Note

The codec uses **fflate** (~8 KB) for deflate compression. fflate is the chosen library because it is significantly smaller than pako (~50 KB), which matters on a 256 MB RAM budget typical of KaiOS 2.x devices.

KaiOS 2.x / Gecko 48 compatibility with fflate must be verified in device testing before the KaiOS 2.x build ships. If fflate fails on Gecko 48, the fallback is pako. This is captured in TASK-DEV-001 (fflate compat verification).

The browser codec (`js/lib/codec.js`) uses the fflate UMD bundle which is compatible with non-module environments — making it suitable for Gecko 48's lack of ES module support.

---

## Deep Link Behaviour

Both build targets must handle the incoming workpad URL case: a user taps a `workpads.me/p#...` link in the KaiOS SMS app. The expected behaviour is that the link opens in the workpads app (via `mozActivity` handle or system URL handler). The exact mechanism differs by KaiOS version and must be tested per target:

- KaiOS 3.x: Standard URL intent handling; `manifest.webmanifest` `protocol_handlers` or share target
- KaiOS 2.x: `mozActivity` registration in `manifest.webapp`

This is in scope for TASK-DEV-002 (deep link and clipboard compatibility matrix).

---

## Branch Strategy

Each build target release gets a dedicated git branch:

| Branch | Build |
|--------|-------|
| `v0.1` | KaiOS 3.x, no build tool, vanilla JS |
| `v0.2-kaios3` | KaiOS 3.x, Babel pipeline |
| `v0.2-kaios2` | KaiOS 2.x, Babel + Gecko 48 target |
| `main` | Latest development |

Releases are ZIP archives built from the appropriate branch, not from `main` directly.
