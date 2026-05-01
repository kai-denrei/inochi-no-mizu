---
role: ux
owner: Gerald
status: active
last-updated: 2026-05-01
---

# User Experience

## Scope
Owns: visual hierarchy of the two planes, no-marker tap feedback, evaporation timing curve, round-transition fade duration, the overall "kami feels a pull, doesn't aim" feeling. Guards the design principle from the brief: *if a feature would make the player feel like a marksman, it's wrong; if it would make them feel like weather, it's right.*

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-05-01 | No tap marker, ever (no green dot, no ripple, no number) | Brief is unambiguous. The visible feedback is the droplet motion itself | [[pm]] [[dev]] |
| 2026-05-01 | Single line of opening text only — no in-game instructions | Brief mandates this. The mechanic must be teachable by feel | [[pm]] |
| 2026-05-01 | Opening line = `"the cave listens"` | Minimal. Doesn't instruct ("tap to play"). Doesn't role-cast ("be the rain"). Sets the listening frame so the player notices audio is the score. | [[pm]] |
| 2026-05-01 | Round-transition fade = 1200ms (600ms out + 600ms in) | A breath, not mechanical. Long enough to feel like a beat between rounds. | [[dev]] |
| 2026-05-01 | Evaporation curve = asymmetric: 1.5s flat, 1.5s linear decay, 0.4s sudden cut | Reads as "still there, still there, fading… gone." Linear-decay-only feels like a CSS animation, not a kami's disappointment. | [[dev]] |
| 2026-05-01 | 2.5D angle = 20° rotateX | Splits the brief's 15-25° range. Tilted enough to read as "two planes," shallow enough that drop motion is still legible in 2D. | [[arch]] |
| 2026-05-01 | Color palette: upper plane mossy dark green (#2a3a2e), drops soft white-blue (#cfe6ff), lower plane stone grey (#3a3633), background near-black (#0a0d10) | Quiet, cave-like, no neon. Drops are the only bright element so the eye tracks them | [[arch]] |
| 2026-05-01 | Lower plane has gentle radial gradient (lighter centre) but no target ring or aim marker | Per [[pm]]: cosmetic only. Lighting cue suggests "things land here" without instructing | [[pm]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons

## Open Questions
<!-- All Open Questions resolved 2026-05-01. -->

## Assumptions
- Players will tolerate not knowing they're playing rounds — status: untested — since: 2026-05-01
- The metaball aesthetic reads as "water" without an explicit blue+wet shader — status: pending-playtest — since: 2026-05-01

## Dependencies
Blocked by: nothing
Feeds into: [[qa]] (ship-gate self-check)

## Session Log
2026-05-01 — Visual + timing knobs locked. Opening line, palette, fade duration, evaporation curve all set.
2026-05-01 — INIT. Marksman-vs-weather principle adopted as the UX guard rail.
