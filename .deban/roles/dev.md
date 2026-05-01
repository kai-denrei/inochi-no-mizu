---
role: dev
owner: Gerald
status: active
last-updated: 2026-05-01
---

# Engineering

## Scope
Owns: droplet state model, retarget math (per-drop velocity inversely scaled by distance from true centroid), metaball merge detection + threshold, Web Audio voices (merge tic, fall plonk), round transition logic, evaporation timing.

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-05-01 | Centroid math reused verbatim from KikaCentroid: x̄ = (1/n)Σxᵢ, ȳ = (1/n)Σyᵢ over the active droplet set | Brief mandates preservation; no need to redesign | [[arch]] [[pm]] |
| 2026-05-01 | Round generators (tetro-cluster, random-spread, long-shot-skewed) reuse KikaCentroid mode logic, parameter-tuned for droplet counts ≥4 | Three rounds == three modes is symmetric and matches the brief's "1–3 rounds" | [[pm]] |
| 2026-05-01 | Retarget speed = `base * clamp(1 - distFromTrueCentroid / maxDist, 0.15, 1)` | Drops near the true centroid arrive fast (well-aimed feel); drops far from it crawl (poor read = visible struggle). Floor at 0.15 so even worst drops drift, not freeze. | [[ux]] |
| 2026-05-01 | Metaball merge detection: drops within radius `2*r` of each other are flagged "merged" and assigned the same group id; merged groups render as a single soft blob (already handled by SVG filter alpha-threshold) | Cheap O(n²) pair check at n≤8. Group id used for fall detachment and audio. | [[arch]] |
| 2026-05-01 | Mass-threshold: per-group score = Σ pairwise (1/dist) over members; threshold = `members * 0.8` | Density-driven per [[pm]]. Tight cluster of 4 crosses easily; spread of 4 never crosses. Group must have ≥3 members minimum to be eligible. | [[pm]] |
| 2026-05-01 | Merge tic on each new pair-join: 80ms sine, ~1200Hz with rapid pitch sweep down 200Hz, peak gain 0.04 | Brief: "high-pitched, low amplitude, ~80ms" | [[arch]] |
| 2026-05-01 | Fall plonk: fundamental = `220 * (4/groupSize)^0.4`, 600ms decay, peak gain `0.15 + 0.05*log(size)` | Larger merge → lower frequency, louder. The 0.4 exponent is feel-tuned (a square-root felt too dramatic). | [[arch]] |
| 2026-05-01 | Cave IR: 300ms exponentially-decaying white noise buffer, generated once at audio init | Procedural keeps single-file. Sounds caverny enough; not trying to be a real space. | [[arch]] |
| 2026-05-01 | Evaporation: alpha = 1 for first 1.5s post-tap, then linear decay over 1.5s, then sudden cut over 0.4s. Drops below alpha 0.1 are removed from the active set. | Asymmetric curve per [[ux]] decision (slow then sudden). | [[ux]] |
| 2026-05-01 | Round transitions: 1200ms cross-fade between rounds. End-of-round trigger = splash event OR all drops evaporated. | Per [[pm]] decision. Fade is the only between-round signal. | [[ux]] |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons

## Open Questions
<!-- All Open Questions resolved 2026-05-01. -->

## Assumptions
- Droplet count per round 4–7 — status: chosen — since: 2026-05-01 (round 1 tetro = 4, round 2 random = 6, round 3 long-shot = 5+1 outlier)
- 60fps achievable on a 2020-era laptop with SVG-filter metaball at 8 droplets — status: pending-playtest — since: 2026-05-01
- Web Audio API's procedural IR (decaying noise) is sufficient for the cave reverb — status: pending-playtest — since: 2026-05-01

## Dependencies
Blocked by: nothing
Feeds into: [[qa]] (perf budget, ship gate)

## Session Log
2026-05-01 — Math + audio + timing knobs locked. Build started.
2026-05-01 — INIT. Centroid math + round generators bound to KikaCentroid as source-of-truth.
