# Kami no Ame — PoC Brief

A proof-of-concept for the rain-kami cave challenge: the player guides droplets to coalesce on an upper surface, and merged drops fall to a lower surface below. Mechanically a re-skin of KikaCentroid; experientially a different game entirely. The point of the PoC is to validate that the centroid mechanic survives — and gains weight from — the loss of all explicit scoring.

## Source mechanic

The local KikaCentroid game is the mechanical seed. **Read its source first.** What to preserve:

- The math: centroid as `x̄ = (1/n)Σxᵢ, ȳ = (1/n)Σyᵢ` of the active droplet set
- The intuition: tap-driven, instant commit, no aim line
- The round structure: a fresh droplet configuration each round; the PoC needs only 1–3 rounds to demonstrate

What to discard:
- All visible scoring (the 0–100, the per-second timer decay, the per-cell distance penalty, the round counter, the achievements)
- The green guess marker
- The grid
- The dot glyphs — droplets replace them

## Visual

A 2.5D side-on perspective with two horizontal planes, viewed at a slight angle (shallow isometric or a single-vanishing-point tilt of ~15–25°):

- **Upper plane** — the kami's surface. Mossy stone, a cave-ceiling fragment, or a curved leaf. Droplets sit on this surface as small soft beads. The player's tap occurs here.
- **Lower plane** — the receiving surface, parallel to the upper plane and visible below through empty space. For PoC: a stone basin with stylized indicators of "where drops land." For the final game: this layer eventually contains the wounded.

The upper plane is rendered with a **metaball field**: each droplet is a soft scalar source, and overlapping fields merge before their visible edges touch. Standard technique — render droplets as blurred radial gradients to an offscreen canvas, then threshold (alpha cutoff or color-matrix step) to harden the merged blob shape. PixiJS has this as a one-line filter; Canvas 2D needs an SVG `feGaussianBlur` + `feColorMatrix` wrapper or a manual two-pass.

## Interaction

On tap (anywhere on the upper plane, **no marker shown**):

1. All droplets retarget their positions toward the tap point, with arrival speed inversely scaled by distance from the true centroid. Drops near a well-aimed tap arrive smoothly. Drops in a misjudged region drift sideways and lose momentum.
2. As droplets converge, the metaball field merges them into a single growing blob. Mass accumulates as the visible area of the merged region.
3. When merged mass crosses a threshold, gravity wins: the merged drop detaches from the upper surface and falls visibly through the gap to the lower plane, landing with a splash on a target zone.

Drops that fail to merge — too scattered, too far from the centroid — never accumulate enough mass to fall. They sit on the upper plane, slowly evaporating (alpha decay over a few seconds), and the round ends with them lost.

## Audio

Procedural via Web Audio API. Two synthesized voices, both built from sine-with-pitch-sweep + amplitude envelope:

- **Merge tic** — fires each time a metaball pair joins. Short (~80ms), high-pitched, low amplitude. Conveys ongoing coalescence.
- **Fall plonk** — fires at the moment of detachment and impact. Lower fundamental for larger merged mass; higher for smaller (real bubble-oscillation scaling). Amplitude scales with mass. Optionally routed through a convolution reverb with a cave impulse response (~200–400ms stone-reflection IR).

Sound is the primary feedback channel for accuracy. A clean centroid produces one big low resonant plonk; a poor read produces a few small high tics and silence.

## What is deliberately absent

- No score number
- No tap indicator
- No accuracy text
- No round-end summary
- No instructions beyond a single line of opening text

The player learns the mechanic by feel. The audio-visual feedback **is** the score.

## Tech stack

**Recommended:** PixiJS for the upper plane (metaball filter is cheap and beautiful, handles 2.5D skewed presentation via sprite transforms). Web Audio API for sound. GSAP or hand-rolled tweening for droplet motion. Single HTML file, deployable to GitHub Pages exactly like KikaCentroid.

**Fallback if avoiding deps:** Canvas 2D with SVG filter wrapper for blur+threshold. ~30% more code, removes 150kb of bundle. Still single-file deployable.

**Avoid:** Three.js, real fluid simulation (SPH), physics engines. Overkill for the target experience and adds weeks to the build.

## PoC scope

**In:**
- One scene, three rounds of varying droplet configurations (tetromino-like cluster, random spread, skewed long-shot — mirroring KikaCentroid's modes)
- Invisible tap → droplet retarget → metaball merge → drop fall → splash
- Procedural audio
- Round transitions with no UI other than a brief fade

**Out (deferred):**
- Narrative reveal layer (wounded, names, historical context)
- Multiple cave scenes
- Difficulty progression
- Persistence
- Mobile-specific tuning (build desktop-first)

## Design principles to preserve

The kami doesn't see its own intention; it feels a pull and watches the drops respond. Every UI choice should reinforce that the player is *part of the system*, not an operator above it.

If a feature would make the player feel like a marksman, it's wrong.
If it would make them feel like weather, it's right.
