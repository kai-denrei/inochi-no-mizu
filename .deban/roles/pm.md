---
role: pm
owner: Gerald
status: active
last-updated: 2026-05-01
---

# Project Management

## Scope
Owns: brief interpretation, scope boundaries (what's in/out for PoC), v1 ship gate, naming canonicalization, prioritization when [[ux]]/[[dev]]/[[arch]] disagree. Does not own implementation choices once scope is set.

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-05-01 | PoC scope frozen to brief's "In" list | Brief is explicit and tight; defer narrative layer, multi-scene, difficulty curve, persistence, mobile tuning | [[dev]] [[ux]] [[arch]] |
| 2026-05-01 | Solo mode, single owner across all 6 roles | Per user instruction; no team to coordinate | — |
| 2026-05-01 | KikaCentroid is the source-of-truth for centroid math + round generation | Brief explicitly says "Read its source first." Mode generators (tetro / random / long-shot) are reusable | [[dev]] [[arch]] |
| 2026-05-01 | Canonical name = `inochi-no-mizu` | Folder match; "water of life" maps onto deferred wounded/healing narrative; "Kami no Ame" is the poetic in-brief subtitle, not displayed on screen | [[devops]] [[ux]] |
| 2026-05-01 | Round-end signal = splash on success OR silence-into-fade on evaporation | Brief allows only fade for transitions. Splash sound itself is a strong end-marker; failure rounds end when all drops evaporate, then fade. No text needed | [[ux]] [[dev]] |
| 2026-05-01 | Lower plane is cosmetic only for PoC; aim is implicit in merge quality | Brief is ambiguous; choosing the version that keeps "feedback IS the score" honest. A hidden target zone re-introduces the marksman frame | [[ux]] [[dev]] |
| 2026-05-01 | Mass threshold is density-driven (sum of pairwise 1/dist of merged drops) | Brief favors "feel" over "puzzle"; quality-driven means a clean centroid-tap rewards itself; bad merges never cross the threshold and evaporate naturally | [[dev]] |
| 2026-05-01 | Cold-start: round 1 = tetro-cluster (easiest), round 2 = random spread, round 3 = long-shot | Teaches by contrast with success front-loaded; first-time observer hears the big plonk in round 1, not a silence-failure | [[ux]] [[qa]] |
| 2026-05-01 | v1 ship gate adopted as proposed by [[qa]] | A first-time observer can complete 3 rounds with at least one successful merge-and-fall. Falsifiable, no numeric score required | [[qa]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons

## Open Questions
<!-- All initial Open Questions resolved 2026-05-01. New ones go here as they arise. -->

## Assumptions
- Single-file HTML deployment is preserved (matches KikaCentroid pattern, GitHub Pages compatible) — status: validated-by-build — since: 2026-05-01
- Canvas2D + SVG filter is sufficient (PixiJS dropped) — status: validated-by-spike — since: 2026-05-01
- Desktop-first, no mobile tuning for PoC — status: validated-by-brief — since: 2026-05-01
- 3 rounds is enough to demonstrate the mechanic for a reviewer — status: untested-pending-playtest — since: 2026-05-01

## Dependencies
Blocked by: nothing
Feeds into: v1 ship-gate self-check

## Session Log
2026-05-01 — Resolved all 5 brief-level Open Questions. PoC build started. Canonical name = inochi-no-mizu. Lower plane cosmetic only. Density-driven mass threshold. Round order = tetro → random → long-shot.
2026-05-01 — INIT. Brief ingested, 5 untested assumptions surfaced. Source mechanic at ../KikaCentroid confirmed. PM agent dispatched.
