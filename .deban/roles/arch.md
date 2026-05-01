---
role: arch
owner: Gerald
status: active
last-updated: 2026-05-01
---

# Architecture

## Scope
Owns: render pipeline choice (PixiJS vs Canvas2D+SVG), audio graph topology, single-file deployable shape (inline vs split assets), state management (none beyond per-round), no-persistence stance.

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-05-01 | No persistence layer for PoC | Brief explicitly defers persistence; ephemeral run, no localStorage, no SW | [[devops]] [[pm]] |
| 2026-05-01 | No service worker for PoC | KikaCentroid's SW exists for offline PWA play; the PoC needs only "open URL → play". SW adds complexity (cache busting bit them twice) without earning anything for a PoC reviewer. | [[devops]] |
| 2026-05-01 | Render pipeline: Canvas 2D + SVG feGaussianBlur+feColorMatrix metaball filter, no PixiJS | Single-file constraint — PixiJS adds 150kb. SVG filter wraps the upper-plane canvas via CSS `filter: url(#metaball)`. Threshold via feColorMatrix alpha-amplify-then-clip. KikaCentroid proves Canvas2D scales. | [[dev]] [[devops]] |
| 2026-05-01 | Single HTML file, inline JS + CSS + SVG filters | Brief mandates single-file. Reviewer opens one URL. No build step. SVG filter defs live in the same document as the canvas they apply to. | [[devops]] |
| 2026-05-01 | Audio graph: gain → convolver (procedural cave IR via decaying noise buffer) → destination, with per-voice `OscillatorNode` + `GainNode` envelope | Web Audio is sufficient. IR is generated from a 300ms exponentially-decaying white noise buffer; no asset loading. | [[dev]] |
| 2026-05-01 | 2.5D look via CSS `transform: rotateX(20deg)` on the upper-plane canvas, not via in-canvas perspective math | Cheap, GPU-accelerated, composes cleanly with the SVG filter applied to the same element. Lower plane is a separate sibling element with the same transform. | [[ux]] [[dev]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons

## Open Questions
<!-- All Open Questions resolved 2026-05-01. -->

## Assumptions
- Web Audio API works in all current desktop browsers without polyfill — status: validated-by-baseline — since: 2026-05-01
- A ~200–400ms cave IR can be procedurally generated (decaying noise) instead of loaded as a binary asset, preserving single-file shape — status: pending-build-test — since: 2026-05-01
- CSS rotateX composes correctly with `filter: url(#metaball)` in Chrome and Safari — status: pending-build-test — since: 2026-05-01

## Dependencies
Blocked by: nothing
Feeds into: [[dev]] (render API constrains math integration), [[devops]] (single-file deploy)

## Session Log
2026-05-01 — Pipeline locked: Canvas2D + SVG filter, single HTML file, Web Audio with procedural IR, CSS rotateX for 2.5D.
2026-05-01 — INIT. No-persistence, no-SW stance set. Render pipeline open.
