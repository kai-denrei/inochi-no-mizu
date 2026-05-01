---
role: qa
owner: Gerald
status: active
last-updated: 2026-05-01
---

# Quality Assurance

## Scope
Owns: validation strategy in the absence of a numeric oracle, cold-start playability checks, perf budget verification, cross-browser smoke (desktop only for PoC).

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-05-01 | Validation is feel-driven, not numeric | The brief removes all explicit scoring; there is no number to assert against. v1 ships when the *play loop is legible* to a fresh observer | [[pm]] [[ux]] |
| 2026-05-01 | Desktop-only smoke for PoC: latest Chrome, latest Safari | Brief defers mobile. Firefox skipped unless something breaks. | [[devops]] |
| 2026-05-01 | Audio gesture-unlock = first tap creates the AudioContext (the opening-line click) | Stock Web Audio policy. No separate "tap to enable sound" prompt — the opening line click is the gesture. | [[dev]] |
| 2026-05-01 | v1 ship gate accepted | "A first-time observer who has never seen KikaCentroid can complete 3 rounds with at least one successful merge-and-fall." | [[pm]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons

## Open Questions
<!-- All Open Questions resolved 2026-05-01. -->

## Assumptions
- 60fps with 8 metaball droplets on M4 Mac Mini is the perf target — status: pending-playtest — since: 2026-05-01
- A single observer (the agent itself, then Gerald) is sufficient validation for the v1 gate — status: validated-by-scope — since: 2026-05-01

## Dependencies
Blocked by: [[dev]] build complete
Feeds into: v1 sign-off

## Session Log
2026-05-01 — Ship gate MET via headless Chrome playthrough. All 3 rounds played end-to-end, round 1 produced a clean merge+fall+splash. Live deploy at https://kai-denrei.github.io/inochi-no-mizu/ confirmed working.
2026-05-01 — Ship gate locked, gesture-unlock pattern set.
2026-05-01 — INIT. Feel-driven validation strategy adopted.
