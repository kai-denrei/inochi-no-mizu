---
project: inochi-no-mizu
created: 2026-05-01
status: active
mode: solo
stale_threshold_days: 30
---

# inochi-no-mizu — Index

## Brief
PoC for the rain-kami cave challenge: the player guides droplets to coalesce on an upper surface; merged drops fall to a lower surface below. Mechanically a re-skin of [KikaCentroid](../../KikaCentroid/) — same centroid math, same instant-tap commit, same 1–3 round structure. Experientially the opposite: all explicit scoring (number, timer decay, distance penalty, round counter, achievements) is removed; the audio-visual feedback *is* the score. The PoC validates that the centroid mechanic survives — and gains weight from — the loss of all UI scoring.

Working title in brief: "Kami no Ame". Folder name: `inochi-no-mizu`. Naming canonical-form is unresolved (see [[pm]] Open Questions).

Brief lives at `CLAUDE_inochinomizu.md` in the project root.

## Active Roles
- [[pm]] — owner: Gerald — scope, assumptions, v1 ship gate, naming
- [[arch]] — owner: Gerald — render pipeline (PixiJS vs Canvas2D+SVG), audio graph, single-file deploy shape
- [[dev]] — owner: Gerald — droplet retarget math, metaball threshold, Web Audio voices, round transitions
- [[ux]] — owner: Gerald — 2.5D framing, no-marker tap, evaporation timing, fade-only transitions
- [[qa]] — owner: Gerald — feel-driven validation (no numeric oracle), cold-start playability, perf budget
- [[devops]] — owner: Gerald — GitHub Pages deploy, single-file constraint, no service worker for PoC

## Key Decisions
- Canonical name = `inochi-no-mizu` (2026-05-01) — folder + repo + URL. "Kami no Ame" is the poetic in-brief subtitle, not on-screen [[pm]] [[devops]]
- Render pipeline = Canvas2D + SVG metaball filter, single HTML file, no PixiJS (2026-05-01) — preserves single-file deploy [[arch]]
- Mass-threshold = density-driven (Σ pairwise 1/dist of merged drops) (2026-05-01) — quality wins, not count [[dev]] [[pm]]
- Lower plane = cosmetic only for PoC, no aim target (2026-05-01) — keeps "feedback IS the score" honest [[pm]] [[ux]]
- Round order = tetro-cluster → random-spread → long-shot (2026-05-01) — easiest first, teach by contrast [[pm]] [[ux]]
- Opening line = "the cave listens" (2026-05-01) [[ux]]
- v1 ship gate = first-time observer completes 3 rounds with ≥1 successful merge-and-fall (2026-05-01) [[qa]]

## Open Questions (cross-role)
<!-- All initial Open Questions resolved 2026-05-01. -->

