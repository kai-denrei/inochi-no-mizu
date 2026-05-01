---
role: devops
owner: Gerald
status: active
last-updated: 2026-05-01
---

# DevOps

## Scope
Owns: GitHub Pages deploy, repo setup, branch hygiene (no direct push to main per kainode rules), single-file constraint compliance, PoC deploy URL.

## Decisions
| Date | Decision | Rationale | Linked roles |
|---|---|---|---|
| 2026-05-01 | Deploy via GitHub Pages from `main`, mirroring KikaCentroid pattern | Single-file static; no build step; reviewer just opens a URL | [[arch]] |
| 2026-05-01 | No service worker, no PWA manifest for PoC | Per [[arch]]; defer until persistence/offline becomes a v1+ requirement | [[arch]] |
| 2026-05-01 | Repo name = `inochi-no-mizu` (matches folder + canonical PM choice) | Per [[pm]] canonicalization decision | [[pm]] |
| 2026-05-01 | Repo visibility = public from day 1 (KikaCentroid pattern) | Pages requires public for free accounts; reviewer just gets a URL | [[pm]] |
| 2026-05-01 | Pages source = `feat/poc-v1` branch (set after PR), not `main` | User rule: never push to main, Gerald merges. Branch deploy lets Gerald see the PoC live before merging. | — |

## Dead Ends
<!-- APPEND ONLY. Never delete. -->
| Date | What was tried | Why it failed / was rejected |
|---|---|---|

## Lessons

## Open Questions
<!-- All Open Questions resolved 2026-05-01. -->

## Assumptions
- The kainode GitHub identity (`kai-denrei`) is the right account for this repo — status: validated-by-rule — since: 2026-05-01
- Branch protection on `main` will be enforced even for solo work (per kainode rules) — status: validated-by-rule — since: 2026-05-01

## Dependencies
Blocked by: nothing
Feeds into: v1 reviewer URL

## Session Log
2026-05-01 — Repo decisions locked: name=inochi-no-mizu, public, deploy from feat/poc-v1 branch.
2026-05-01 — INIT. GitHub Pages deploy stance set; repo creation deferred to PM agent.
