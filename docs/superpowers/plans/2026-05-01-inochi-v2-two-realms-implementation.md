# inochi-no-mizu v2 — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Visual rewrite of the inochi-no-mizu PoC from Canvas2D + SVG metaball + CSS rotateX trick to Three.js orthographic 3D with two visually distinct realms (DMT-vocabulary upper, photo-real lower) and a void where falling drops morph between them. v1 mechanic is preserved verbatim.

**Architecture:** Single HTML file. Three.js loaded via `<script type="importmap">` from `esm.sh`. Inline GLSL shaders as template-literal strings inside `<script type="module">`. v1 game logic ported into the same module. Five phases, each ending in a reviewable commit playable at `http://localhost:8765`.

**Tech Stack:** Three.js (esm.sh CDN), WebGL2, GLSL ES 3.0 shaders, Web Audio API (reused from v1), GitHub Pages.

---

## File Structure

- **`index.html`** — single deployable file, fully rewritten over phases. New top-level sections:
  - `<head>`: meta, `<script type="importmap">` for Three.js, inline `<style>` for canvas mount + opening line
  - `<body>`: `<canvas id="stage">`, `<div id="intro">the cave listens</div>`, `<div id="fade">` for round transitions
  - `<script type="module">`: imports, constants, scene setup, shaders, mechanic, audio, game loop
- **`docs/superpowers/plans/2026-05-01-inochi-v2-two-realms-implementation.md`** — this file
- **`.deban/`** — updated in phase 5 with v2 decisions

## Branching strategy

Branch `feat/v2-two-realms` cut from current `feat/poc-v1` HEAD. Tag v1's tip as `v1-poc` before any destructive change to `index.html`. Spec calls for branching off `main` after v1 PR merges, but the v1 merge is the user's call — we don't block.

## Verification approach

No test framework (per `.deban/roles/arch.md` — solo PoC, manual playtesting only). Each task ends with a manual browser check at `http://localhost:8765`. The dev server is already running. **Verify by opening, clicking the opening line, and observing the listed behaviors.** Each phase ends in a commit.

## World coordinate system

Orthographic camera frustum: top=50, bottom=-50, left=-aspect*50, right=aspect*50. Height = 100 world units. Z range -100 to +100.

Region mapping (Y-axis):
- Upper realm: y ∈ [5, 50] — 45 units (~55% viewport at 80-unit visible height)
- Void: y ∈ [-7, 5] — 12 units (~15%)
- Lower realm: y ∈ [-30, -7] — 23 units (~30%)

Drop positions stored in world coordinates as `(x, y)` in this Y-vertical layout. v1's pixel-space mechanic functions are coordinate-agnostic — the centroid/retarget/merge math works in any consistent unit.

### Architecture note: simplified isometric

The spec ideal is a fully horizontal upper plane that the camera tilts to "look down onto." This plan implements a simpler approximation: the upper plane is a vertical region in screen space, drops live in its XY plane, and the iso feel comes from orthographic projection alone (no perspective foreshortening). The lower basin in Phase 4 is a true horizontal mesh, since it visually requires the look-down view to read as a bowl.

The simplification keeps the centroid/merge/retarget math identical to v1 (2D) and avoids a raycaster click-handler with plane intersection. If after Phase 5 the look feels too 2D, an architectural follow-up — make the upper plane horizontal, re-route clicks through `Raycaster`, animate falling drops in Y from upper-plane Y to basin Y — is straightforward and doesn't reach into the shader code or atmosphere meshes. Calling it out so the executing agent doesn't get surprised mid-stream.

---

## Phase 1 — Three.js scaffold + v1 mechanic ported, drops as plain spheres

End state: game is fully playable in Three.js. Three rounds, click to retarget, drops merge, fall, splash. Visuals are minimal placeholder: gradient background, spheres for drops, dark basin. No shaders yet. **Architecture proven.**

### Task 1.1 — Branch, tag v1, scaffold

**Files:**
- Create: branch `feat/v2-two-realms`
- Create: tag `v1-poc` on current HEAD
- Modify: `index.html` (full rewrite — overwrite v1)

- [ ] **Step 1: Verify clean working tree, current branch is `feat/poc-v1`**

```bash
cd /Users/minikai/Documents/Dev/inochi-no-mizu
git status
git branch --show-current
```

Expected: `nothing to commit, working tree clean`, branch `feat/poc-v1`.

- [ ] **Step 2: Tag v1 and create the v2 branch**

```bash
git tag v1-poc
git checkout -b feat/v2-two-realms
```

- [ ] **Step 3: Overwrite `index.html` with the v2 scaffold**

Write this content to `index.html`:

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<title>inochi no mizu</title>
<script type="importmap">
{
  "imports": {
    "three": "https://esm.sh/three@0.160.0"
  }
}
</script>
<style>
  :root { --bg: #06030f; --text: #8a9aa3; }
  * { margin: 0; padding: 0; box-sizing: border-box; }
  html, body {
    width: 100%; height: 100%;
    background: var(--bg);
    color: var(--text);
    font-family: -apple-system, BlinkMacSystemFont, "Helvetica Neue", system-ui, sans-serif;
    overflow: hidden;
    user-select: none;
    -webkit-tap-highlight-color: transparent;
  }
  #stage { position: fixed; inset: 0; display: block; cursor: crosshair; }
  #intro {
    position: fixed; left: 50%; top: 50%;
    transform: translate(-50%, -50%);
    font-size: 14px; letter-spacing: 0.18em;
    color: rgba(180, 200, 220, 0.7);
    pointer-events: none;
    transition: opacity 800ms ease;
    z-index: 2;
  }
  #intro.gone { opacity: 0; }
  #fade {
    position: fixed; inset: 0;
    background: var(--bg);
    opacity: 0;
    pointer-events: none;
    transition: opacity 600ms ease;
    z-index: 3;
  }
  #fade.in { opacity: 1; }
</style>
</head>
<body>
<canvas id="stage"></canvas>
<div id="intro">the cave listens</div>
<div id="fade"></div>
<script type="module">
import * as THREE from "three";

// ── Constants (mechanic-preserving — copied verbatim from v1) ─────────────
const DROP_RADIUS = 1.8;          // world units (was 14 px in v1)
const FALL_DURATION_MS = 900;
const SPLASH_LIFE_MS = 1400;
const FADE_BETWEEN_MS = 1200;
const EVAP_FLAT_MS = 1500;
const EVAP_DECAY_MS = 1500;
const EVAP_CUT_MS = 400;
const EVAP_TOTAL_MS = EVAP_FLAT_MS + EVAP_DECAY_MS + EVAP_CUT_MS;
const RETARGET_MAX_SPEED = 0.30;  // world units / ms (scaled from v1's 2.4 px/ms)
const RETARGET_MIN_SPEED_RATIO = 0.15;
const ARRIVAL_RADIUS = 0.5;
const MERGE_DIST_FACTOR = 1.8;
const MIN_GROUP_FOR_FALL = 3;
const DENSITY_PER_MEMBER = 0.8;

// ── World layout ──────────────────────────────────────────────────────────
const UPPER_TOP = 50;
const UPPER_BOTTOM = 5;
const UPPER_HALF_W = 35;          // upper region horizontal extent (left/right)
const VOID_TOP = 5;
const VOID_BOTTOM = -7;
const LOWER_TOP = -7;
const LOWER_BOTTOM = -30;
const CAMERA_TILT = -25 * Math.PI / 180;

// ── Scene, renderer, camera ───────────────────────────────────────────────
const canvas = document.getElementById("stage");
const renderer = new THREE.WebGLRenderer({ canvas, antialias: true, alpha: false });
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
renderer.setClearColor(0x06030f, 1);

const scene = new THREE.Scene();

const camera = new THREE.OrthographicCamera();
camera.position.set(0, 0, 80);
camera.rotation.x = CAMERA_TILT;

function resize() {
  const w = window.innerWidth, h = window.innerHeight;
  renderer.setSize(w, h, false);
  const aspect = w / h;
  const halfH = 50;
  camera.left = -aspect * halfH;
  camera.right = aspect * halfH;
  camera.top = halfH;
  camera.bottom = -halfH;
  camera.near = 0.1;
  camera.far = 200;
  camera.updateProjectionMatrix();
}
resize();
window.addEventListener("resize", resize);

// ── Placeholder regions (will become real content in later phases) ────────
const upperBg = new THREE.Mesh(
  new THREE.PlaneGeometry(200, UPPER_TOP - UPPER_BOTTOM),
  new THREE.MeshBasicMaterial({ color: 0x1a0d2e })
);
upperBg.position.set(0, (UPPER_TOP + UPPER_BOTTOM) / 2, -10);
scene.add(upperBg);

const lowerBg = new THREE.Mesh(
  new THREE.PlaneGeometry(200, LOWER_TOP - LOWER_BOTTOM),
  new THREE.MeshBasicMaterial({ color: 0x1a1614 })
);
lowerBg.position.set(0, (LOWER_TOP + LOWER_BOTTOM) / 2, -10);
scene.add(lowerBg);

// ── Game state placeholder (filled in next tasks) ─────────────────────────
const S = {
  phase: "intro",      // "intro" | "playing" | "transition"
  drops: [],           // active drops on upper plane
  falling: [],         // drops in flight to lower
  splashes: [],        // active splash effects
  roundIdx: 0,
  roundTotal: 3,
  roundStartTime: 0,
  retargetTo: null,
  audioReady: false,
};

// ── Render loop ───────────────────────────────────────────────────────────
let lastTime = performance.now();
function frame(now) {
  const dt = Math.min(50, now - lastTime);
  lastTime = now;
  // update logic added in later tasks
  renderer.render(scene, camera);
  requestAnimationFrame(frame);
}
requestAnimationFrame(frame);

// ── Bootstrap (intro click handler — wired in next task) ──────────────────
document.getElementById("intro").addEventListener("click", () => {});
canvas.addEventListener("click", () => {});

console.log("inochi-no-mizu v2 scaffold loaded");
</script>
</body>
</html>
```

- [ ] **Step 4: Verify in browser**

The localhost server is already running at `http://localhost:8765`. Reload that page.
Expected:
- A dark window with two horizontal bands (deep violet upper region above; warm-grey lower region below)
- "the cave listens" centered on screen
- Console shows `inochi-no-mizu v2 scaffold loaded`
- No errors. (If Three.js fails to load from esm.sh, you'll see import errors — this is the canary that the CDN strategy works.)

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(v2): Three.js scaffold + region placeholders (phase 1.1)"
```

### Task 1.2 — Port v1 audio (gesture-unlock, merge tic, fall plonk)

**Files:**
- Modify: `index.html` (add audio block inside the module script, between constants and scene setup, and wire intro click)

- [ ] **Step 1: Add audio block to module script**

Insert between the constants block (end of `DENSITY_PER_MEMBER`) and `// ── World layout ──`:

```javascript
// ── Audio (ported from v1) ────────────────────────────────────────────────
let audioCtx = null;
let masterGain = null;
let convolver = null;

function makeCaveIR(seconds, decay) {
  const rate = audioCtx.sampleRate;
  const length = Math.floor(seconds * rate);
  const buf = audioCtx.createBuffer(2, length, rate);
  for (let ch = 0; ch < 2; ch++) {
    const data = buf.getChannelData(ch);
    for (let i = 0; i < length; i++) {
      const t = i / length;
      data[i] = (Math.random() * 2 - 1) * Math.pow(1 - t, decay);
    }
  }
  return buf;
}

function initAudio() {
  if (audioCtx) return;
  const Ctx = window.AudioContext || window.webkitAudioContext;
  if (!Ctx) return;
  audioCtx = new Ctx();
  masterGain = audioCtx.createGain();
  masterGain.gain.value = 0.85;
  convolver = audioCtx.createConvolver();
  convolver.buffer = makeCaveIR(0.3, 3);
  masterGain.connect(convolver);
  convolver.connect(audioCtx.destination);
  S.audioReady = true;
}

function tone(freq, durationMs, peakGain, opts = {}) {
  if (!audioCtx) return;
  const now = audioCtx.currentTime;
  const dur = durationMs / 1000;
  const osc = audioCtx.createOscillator();
  osc.type = opts.type || "sine";
  osc.frequency.setValueAtTime(freq, now);
  if (opts.sweepTo) {
    osc.frequency.exponentialRampToValueAtTime(opts.sweepTo, now + dur);
  }
  const env = audioCtx.createGain();
  env.gain.setValueAtTime(0, now);
  env.gain.linearRampToValueAtTime(peakGain, now + Math.min(0.005, dur * 0.1));
  env.gain.exponentialRampToValueAtTime(0.0001, now + dur);
  osc.connect(env);
  env.connect(masterGain);
  osc.start(now);
  osc.stop(now + dur + 0.05);
}

function playMergeTic() {
  tone(1200, 80, 0.04, { sweepTo: 1000 });
}

function playFallPlonk(groupSize) {
  const baseFreq = 220 * Math.pow(4 / Math.max(2, groupSize), 0.4);
  const peak = 0.14 + 0.05 * Math.log(Math.max(2, groupSize));
  tone(baseFreq, 600, peak, { type: "sine" });
}
```

- [ ] **Step 2: Wire intro click to start audio**

Replace the placeholder `document.getElementById("intro").addEventListener(...)` line at bottom with:

```javascript
const introEl = document.getElementById("intro");
introEl.addEventListener("click", () => {
  initAudio();
  introEl.classList.add("gone");
  S.phase = "playing";
  // first round will start in next task
});
```

- [ ] **Step 3: Verify in browser**

Reload `http://localhost:8765`. Click "the cave listens".
Expected:
- Text fades out
- No audible tone yet (no merges happening yet)
- Console: no errors. `audioCtx.state === "running"` if you check in console

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(v2): port v1 audio (cave IR + merge tic + fall plonk)"
```

### Task 1.3 — Port v1 round generators

**Files:**
- Modify: `index.html` (add seedable RNG + 3 round generator functions, before `// ── Game state placeholder`)

- [ ] **Step 1: Add RNG and round generators**

Insert before `// ── Game state placeholder ──`:

```javascript
// ── RNG (seedable) ────────────────────────────────────────────────────────
let rngSeed = (Math.random() * 0xffffffff) >>> 0;
function mulberry32() {
  rngSeed |= 0; rngSeed = (rngSeed + 0x6D2B79F5) | 0;
  let t = Math.imul(rngSeed ^ (rngSeed >>> 15), 1 | rngSeed);
  t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t;
  return ((t ^ (t >>> 14)) >>> 0) / 4294967296;
}
const rng = mulberry32;

function gaussian(stddev = 1) {
  const u1 = Math.max(rng(), 1e-9);
  const u2 = rng();
  return Math.sqrt(-2 * Math.log(u1)) * Math.cos(2 * Math.PI * u2) * stddev;
}

// ── Round generators (ported from v1, scaled to world units) ──────────────
function pointsTetroCluster() {
  const pieces = [
    [[0,0],[1,0],[0,1],[1,1]],   // square
    [[0,0],[1,0],[2,0],[1,1]],   // T
    [[0,0],[1,0],[1,1],[2,1]],   // S
    [[0,0],[0,1],[0,2],[1,2]],   // L
  ];
  const piece = pieces[Math.floor(rng() * pieces.length)];
  const cell = 5.0;  // world units per tetro cell
  const cx = (rng() - 0.5) * UPPER_HALF_W * 0.4;
  const cy = (UPPER_TOP + UPPER_BOTTOM) / 2 + (rng() - 0.5) * (UPPER_TOP - UPPER_BOTTOM) * 0.2;
  let mxX = 0, mxY = 0;
  for (const [dx, dy] of piece) { if (dx > mxX) mxX = dx; if (dy > mxY) mxY = dy; }
  const ox = cx - (mxX / 2) * cell;
  const oy = cy - (mxY / 2) * cell;
  return piece.map(([dx, dy]) => ({ x: ox + dx * cell, y: oy + dy * cell }));
}

function pointsRandomSpread() {
  const n = 6;
  const pts = [];
  for (let i = 0; i < n; i++) {
    pts.push({
      x: (rng() - 0.5) * UPPER_HALF_W * 1.6,
      y: UPPER_BOTTOM + 4 + rng() * (UPPER_TOP - UPPER_BOTTOM - 8),
    });
  }
  return pts;
}

function pointsLongShot() {
  const n = 5;
  const cx = (rng() - 0.5) * UPPER_HALF_W * 0.4;
  const cy = (UPPER_TOP + UPPER_BOTTOM) / 2 + (rng() - 0.5) * (UPPER_TOP - UPPER_BOTTOM) * 0.15;
  const pts = [];
  for (let i = 0; i < n; i++) {
    pts.push({ x: cx + gaussian(3), y: cy + gaussian(3) });
  }
  // outlier
  pts.push({
    x: cx + (rng() < 0.5 ? -1 : 1) * (15 + rng() * 8),
    y: cy + gaussian(4),
  });
  return pts;
}

const ROUND_GENS = [pointsTetroCluster, pointsRandomSpread, pointsLongShot];
```

- [ ] **Step 2: Verify by inspection (no UI yet)**

Reload, paste in console: `[pointsTetroCluster(), pointsRandomSpread(), pointsLongShot()]`
Expected: 3 arrays of `{x, y}` points with x bounded by ~±UPPER_HALF_W, y inside [UPPER_BOTTOM, UPPER_TOP].

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(v2): port v1 RNG + 3 round generators"
```

### Task 1.4 — Drop entities (Three.js spheres) + retarget update

**Files:**
- Modify: `index.html` (add drop creation, update logic, click handler)

- [ ] **Step 1: Add drop helpers and round start logic**

Insert before `// ── Render loop ──`:

```javascript
// ── Drops ─────────────────────────────────────────────────────────────────
const dropGeometry = new THREE.SphereGeometry(DROP_RADIUS, 24, 16);
const dropMaterial = new THREE.MeshStandardMaterial({
  color: 0xcfe6ff,
  emissive: 0x4480c0,
  emissiveIntensity: 0.4,
  roughness: 0.4,
  metalness: 0.1,
});

// minimal lights for phase 1 (will be replaced in phase 2)
const ambient = new THREE.AmbientLight(0xffffff, 0.6);
scene.add(ambient);
const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
dirLight.position.set(5, 10, 8);
scene.add(dirLight);

function makeDrop(x, y) {
  const mesh = new THREE.Mesh(dropGeometry, dropMaterial.clone());
  mesh.position.set(x, y, 0);
  scene.add(mesh);
  return {
    mesh,
    x, y,
    targetX: x, targetY: y,
    vx: 0, vy: 0,
    bornAt: performance.now(),
    alpha: 1.0,
    groupId: null,
  };
}

function clearDrops() {
  for (const d of S.drops) scene.remove(d.mesh);
  for (const f of S.falling) scene.remove(f.mesh);
  for (const sp of S.splashes) scene.remove(sp.mesh);
  S.drops = []; S.falling = []; S.splashes = [];
}

function startRound(idx) {
  clearDrops();
  S.roundIdx = idx;
  S.roundStartTime = performance.now();
  const gen = ROUND_GENS[idx % ROUND_GENS.length];
  const pts = gen();
  for (const p of pts) S.drops.push(makeDrop(p.x, p.y));
  S.retargetTo = null;
}

function trueCentroid(drops) {
  if (!drops.length) return null;
  let sx = 0, sy = 0;
  for (const d of drops) { sx += d.x; sy += d.y; }
  return { x: sx / drops.length, y: sy / drops.length };
}
```

- [ ] **Step 2: Implement update loop for drops**

Insert before `// ── Render loop ──`:

```javascript
// ── Update logic ──────────────────────────────────────────────────────────
function updateDrops(dt) {
  const now = performance.now();
  const centroid = trueCentroid(S.drops);
  if (!centroid) return;

  // compute max distance from centroid (for retarget speed scaling)
  let maxDist = 0;
  for (const d of S.drops) {
    const dx = d.x - centroid.x, dy = d.y - centroid.y;
    const dist = Math.hypot(dx, dy);
    if (dist > maxDist) maxDist = dist;
  }
  maxDist = Math.max(1, maxDist);

  for (const d of S.drops) {
    // evaporation
    const age = now - d.bornAt;
    if (age < EVAP_FLAT_MS) d.alpha = 1.0;
    else if (age < EVAP_FLAT_MS + EVAP_DECAY_MS) {
      d.alpha = 1.0 - (age - EVAP_FLAT_MS) / EVAP_DECAY_MS;
    } else if (age < EVAP_TOTAL_MS) {
      d.alpha = Math.max(0, 0.05 * (1 - (age - EVAP_FLAT_MS - EVAP_DECAY_MS) / EVAP_CUT_MS));
    } else d.alpha = 0;
    d.mesh.material.opacity = d.alpha;
    d.mesh.material.transparent = true;

    // retarget
    if (S.retargetTo) {
      const dxc = d.x - centroid.x, dyc = d.y - centroid.y;
      const distFromCentroid = Math.hypot(dxc, dyc);
      const speedRatio = Math.max(RETARGET_MIN_SPEED_RATIO, 1 - distFromCentroid / maxDist);
      const speed = RETARGET_MAX_SPEED * speedRatio;

      const tx = S.retargetTo.x, ty = S.retargetTo.y;
      const dx = tx - d.x, dy = ty - d.y;
      const dist = Math.hypot(dx, dy);
      if (dist > ARRIVAL_RADIUS) {
        const nx = dx / dist, ny = dy / dist;
        d.x += nx * speed * dt;
        d.y += ny * speed * dt;
      }
    }

    d.mesh.position.set(d.x, d.y, 0);
  }

  // remove fully-evaporated drops
  S.drops = S.drops.filter(d => {
    if (d.alpha <= 0.01) {
      scene.remove(d.mesh);
      return false;
    }
    return true;
  });
}

// click handler — set retarget point
function screenToWorld(clientX, clientY) {
  const rect = canvas.getBoundingClientRect();
  const ndcX = ((clientX - rect.left) / rect.width) * 2 - 1;
  const ndcY = -(((clientY - rect.top) / rect.height) * 2 - 1);
  const worldX = ndcX * camera.right;
  const worldY = ndcY * camera.top;
  // apply inverse of camera tilt for the upper plane (z=0)
  return { x: worldX, y: worldY / Math.cos(CAMERA_TILT) };
}

canvas.addEventListener("click", (e) => {
  if (S.phase !== "playing") return;
  const w = screenToWorld(e.clientX, e.clientY);
  // only retarget if click is in upper region
  if (w.y < UPPER_BOTTOM) return;
  S.retargetTo = w;
});
```

- [ ] **Step 3: Wire intro click to start round 1**

Replace the existing intro click handler with:

```javascript
const introEl = document.getElementById("intro");
introEl.addEventListener("click", () => {
  initAudio();
  introEl.classList.add("gone");
  S.phase = "playing";
  startRound(0);
});
```

- [ ] **Step 4: Wire update into frame loop**

Replace the `frame()` function with:

```javascript
function frame(now) {
  const dt = Math.min(50, now - lastTime);
  lastTime = now;
  if (S.phase === "playing") {
    updateDrops(dt);
  }
  renderer.render(scene, camera);
  requestAnimationFrame(frame);
}
```

- [ ] **Step 5: Verify in browser**

Reload `http://localhost:8765`. Click "the cave listens", then click somewhere on the upper region.
Expected:
- 4 sphere-drops appear in a tetromino arrangement on the upper plane
- After clicking the upper plane, all drops drift toward your click point
- Drops near the centroid move faster than far drops (visible feel: the cluster glides while outliers crawl)
- After ~3.5s with no merge or fall, drops fade out and disappear
- No errors

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(v2): drop entities + retarget update + evaporation"
```

### Task 1.5 — Merge detection + fall + splash placeholder

**Files:**
- Modify: `index.html` (add merge logic, fall animation, splash placeholder)

- [ ] **Step 1: Add merge detection and trigger-fall logic**

Insert into `updateDrops()`, after the per-drop loop and before the filter-evaporated step:

```javascript
  // ── Merge detection ──────────────────────────────────────────────────
  // group drops within MERGE_DIST_FACTOR * DROP_RADIUS of any group member
  const groups = [];
  for (const d of S.drops) d.groupId = null;
  for (let i = 0; i < S.drops.length; i++) {
    const di = S.drops[i];
    if (di.groupId !== null) continue;
    const gid = groups.length;
    di.groupId = gid;
    const members = [di];
    let added = true;
    while (added) {
      added = false;
      for (let j = 0; j < S.drops.length; j++) {
        const dj = S.drops[j];
        if (dj.groupId !== null) continue;
        for (const m of members) {
          if (Math.hypot(dj.x - m.x, dj.y - m.y) < MERGE_DIST_FACTOR * DROP_RADIUS) {
            dj.groupId = gid;
            members.push(dj);
            added = true;
            break;
          }
        }
      }
    }
    groups.push(members);
  }

  // ── Fall trigger ─────────────────────────────────────────────────────
  // NOTE: guard with S.tapped so fall cannot fire before the player has committed a click.
  // NOTE: dist must be normalized by DROP_RADIUS to keep density score dimensionless/scale-independent.
  if (S.tapped) {
    for (const g of groups) {
      if (g.length < MIN_GROUP_FOR_FALL) continue;
      let density = 0;
      for (let i = 0; i < g.length; i++) {
        for (let j = i + 1; j < g.length; j++) {
          const dist = Math.max(0.001, Math.hypot(g[i].x - g[j].x, g[i].y - g[j].y) / DROP_RADIUS);
          density += 1 / dist;
        }
      }
      if (density >= g.length * DENSITY_PER_MEMBER) {
        triggerFall(g);
        break;
      }
    }
  }
```

- [ ] **Step 2: Add merge-tic detection (audio cue when groups grow)**

Track the previous frame's group sizes and play tic when a group gains a member. Add inside `updateDrops()` before the merge detection section, right after evaporation cleanup:

```javascript
  // (track which drops were grouped last frame for merge-tic audio)
  const prevGroups = new Map();
  for (const d of S.drops) prevGroups.set(d, d.groupId);
```

And after groups are computed:

```javascript
  // play merge tic when any drop newly joins a multi-drop group
  for (const g of groups) {
    if (g.length < 2) continue;
    let newJoin = false;
    for (const d of g) {
      if (prevGroups.get(d) === null || prevGroups.get(d) === undefined) {
        // was ungrouped this frame's start
        newJoin = true; break;
      }
    }
    if (newJoin && S.audioReady) playMergeTic();
  }
```

- [ ] **Step 3: Add `triggerFall` and falling-drop update**

Insert before `// ── Update logic ──`:

```javascript
function triggerFall(members) {
  if (S.audioReady) playFallPlonk(members.length);

  // compute group centroid as the launch point
  let sx = 0, sy = 0;
  for (const m of members) { sx += m.x; sy += m.y; }
  const cx = sx / members.length, cy = sy / members.length;

  // remove members from active drops; create one falling-drop entity
  for (const m of members) {
    scene.remove(m.mesh);
    const idx = S.drops.indexOf(m);
    if (idx >= 0) S.drops.splice(idx, 1);
  }

  const fallGeom = new THREE.SphereGeometry(DROP_RADIUS * 1.2, 24, 16);
  const fallMat = new THREE.MeshStandardMaterial({
    color: 0xcfe6ff,
    emissive: 0x4480c0,
    emissiveIntensity: 0.5,
    roughness: 0.3,
    metalness: 0.1,
  });
  const mesh = new THREE.Mesh(fallGeom, fallMat);
  mesh.position.set(cx, cy, 0);
  scene.add(mesh);

  S.falling.push({
    mesh,
    fromX: cx, fromY: cy,
    toX: cx + (Math.random() - 0.5) * 4,
    toY: LOWER_TOP - 2,  // land just below void/lower boundary
    bornAt: performance.now(),
    size: members.length,
  });
}

function updateFalling() {
  const now = performance.now();
  S.falling = S.falling.filter(f => {
    const t = (now - f.bornAt) / FALL_DURATION_MS;
    if (t >= 1) {
      scene.remove(f.mesh);
      makeSplash(f.toX, f.toY, f.size);
      return false;
    }
    // ease-in-quad gravity
    const e = t * t;
    f.mesh.position.x = f.fromX + (f.toX - f.fromX) * t;
    f.mesh.position.y = f.fromY + (f.toY - f.fromY) * e;
    return true;
  });
}

function makeSplash(x, y, size) {
  // placeholder — concentric ripple sprite
  const geom = new THREE.RingGeometry(0.5, 0.7, 32);
  const mat = new THREE.MeshBasicMaterial({
    color: 0xa0c8e0,
    transparent: true,
    opacity: 0.8,
    side: THREE.DoubleSide,
  });
  const ring = new THREE.Mesh(geom, mat);
  ring.position.set(x, y, 0.1);
  ring.rotation.x = -Math.PI / 2;
  scene.add(ring);
  S.splashes.push({ mesh: ring, bornAt: performance.now(), size });
}

function updateSplashes() {
  const now = performance.now();
  S.splashes = S.splashes.filter(sp => {
    const t = (now - sp.bornAt) / SPLASH_LIFE_MS;
    if (t >= 1) {
      scene.remove(sp.mesh);
      return false;
    }
    sp.mesh.scale.setScalar(1 + t * 6 * sp.size);
    sp.mesh.material.opacity = 0.8 * (1 - t);
    return true;
  });
}
```

- [ ] **Step 4: Wire updateFalling and updateSplashes into the frame loop**

Update `frame()`:

```javascript
function frame(now) {
  const dt = Math.min(50, now - lastTime);
  lastTime = now;
  if (S.phase === "playing") {
    updateDrops(dt);
    updateFalling();
    updateSplashes();
  }
  renderer.render(scene, camera);
  requestAnimationFrame(frame);
}
```

- [ ] **Step 5: Verify in browser**

Reload, click "the cave listens", click in the centre of the tetro cluster on the upper plane.
Expected:
- Drops glide toward click point
- When 3+ drops cluster tightly, you hear merge tics, then a fall plonk
- A larger drop falls visibly through the void to the lower region
- A ring expands outward at impact and fades

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat(v2): merge detection + fall animation + splash placeholder"
```

### Task 1.6 — Round transitions (3 rounds, fade)

**Files:**
- Modify: `index.html` (add end-of-round detection + cross-fade transition)

- [ ] **Step 1: Add round-end detection**

Insert after `triggerFall(g); break;` block but inside `updateDrops()`:

```javascript
  // round-end: all drops gone (evaporated or fell)
  if (S.phase === "playing" && S.drops.length === 0 && S.falling.length === 0 && S.splashes.length === 0) {
    if (now - S.roundStartTime > 1000) {  // small grace period after round start
      endRoundAndAdvance();
    }
  }
```

- [ ] **Step 2: Implement `endRoundAndAdvance`**

Insert before `// ── Update logic ──`:

```javascript
const fadeEl = document.getElementById("fade");

function endRoundAndAdvance() {
  if (S.phase !== "playing") return;
  S.phase = "transition";
  fadeEl.classList.add("in");
  setTimeout(() => {
    const nextIdx = S.roundIdx + 1;
    if (nextIdx >= S.roundTotal) {
      // game complete — fade in only, then idle
      S.phase = "complete";
      return;
    }
    startRound(nextIdx);
    S.phase = "playing";
    fadeEl.classList.remove("in");
  }, FADE_BETWEEN_MS / 2);
}
```

- [ ] **Step 3: Verify in browser**

Reload, complete a successful merge-and-fall in round 1, watch the transition.
Expected:
- After splash completes and ripples fade, screen fades to background
- Briefly black, then fades back with new drop configuration (round 2 = random spread)
- Round 2 → fade → Round 3 (long-shot)
- After round 3 ends (success or evaporation), screen stays faded — no further rounds

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(v2): 3-round structure with fade transitions"
```

### Task 1.7 — Phase 1 commit + verify ship-gate parity

**Files:** none new — verification only

- [ ] **Step 1: Run full playthrough**

Reload the page, click intro, play 3 rounds end-to-end.
Verify:
- All v1 mechanic behaviors preserved (centroid retarget, density-driven merge fall, evaporation, fade transitions)
- 3 rounds run with different drop arrangements
- At least one round produces a successful merge-and-fall on round 1 (tetro is forgivingly tight)
- Audio plays (merge tics + fall plonk)
- No console errors

This is the v1 ship-gate met in the v2 architecture. End of phase 1.

- [ ] **Step 2: Push branch (no PR yet)**

```bash
git push -u origin feat/v2-two-realms
```

---

## Phase 2 — Drop-interior shader, aurora background, drifting motes

End state: upper realm reads as non-human. Drops feel like windows onto another space.

### Task 2.1 — Aurora background shader

**Files:**
- Modify: `index.html` (replace `upperBg` with shader-material plane)

- [ ] **Step 1: Define aurora shader and replace upperBg material**

Find the `upperBg` block and replace with:

```javascript
const auroraVS = `
  varying vec2 vUv;
  void main() {
    vUv = uv;
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`;

const auroraFS = `
  varying vec2 vUv;
  uniform float uTime;

  // simple FBM for cloud bands
  float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }
  float noise(vec2 p) {
    vec2 i = floor(p); vec2 f = fract(p);
    float a = hash(i); float b = hash(i + vec2(1.0, 0.0));
    float c = hash(i + vec2(0.0, 1.0)); float d = hash(i + vec2(1.0, 1.0));
    vec2 u = f * f * (3.0 - 2.0 * f);
    return mix(a, b, u.x) + (c - a) * u.y * (1.0 - u.x) + (d - b) * u.x * u.y;
  }
  float fbm(vec2 p) {
    float v = 0.0; float a = 0.5;
    for (int i = 0; i < 5; i++) { v += a * noise(p); p *= 2.0; a *= 0.5; }
    return v;
  }

  void main() {
    vec2 uv = vUv;
    // base vertical gradient: deep violet → near-black
    vec3 base = mix(vec3(0.10, 0.05, 0.18), vec3(0.04, 0.02, 0.08), 1.0 - uv.y);

    // magenta + cyan accent ellipses, drifting slowly
    vec2 p1 = uv - vec2(0.30 + 0.04 * sin(uTime * 0.05), 0.70);
    float c1 = exp(-dot(p1 * vec2(2.5, 1.8), p1 * vec2(2.5, 1.8)) * 1.2);
    vec3 violet = vec3(0.55, 0.35, 0.78) * c1 * 0.55;

    vec2 p2 = uv - vec2(0.75, 0.40 + 0.03 * cos(uTime * 0.04));
    float c2 = exp(-dot(p2 * vec2(2.0, 2.3), p2 * vec2(2.0, 2.3)) * 1.3);
    vec3 cyan = vec3(0.16, 0.70, 0.78) * c2 * 0.45;

    // pink/gold haze in the middle
    float c3 = exp(-pow(uv.y - 0.5, 2.0) * 8.0) * 0.10;
    vec3 warm = vec3(1.0, 0.62, 0.78) * c3;

    // subtle FBM cloud breath
    float cloud = fbm(uv * 4.0 + vec2(uTime * 0.01, uTime * 0.008));
    vec3 cloudCol = mix(vec3(0.12, 0.08, 0.20), vec3(0.20, 0.10, 0.30), cloud) * 0.5;

    vec3 col = base + violet + cyan + warm + cloudCol * 0.4;

    gl_FragColor = vec4(col, 1.0);
  }
`;

const auroraMat = new THREE.ShaderMaterial({
  vertexShader: auroraVS,
  fragmentShader: auroraFS,
  uniforms: { uTime: { value: 0 } },
  depthWrite: false,
});
const upperBg = new THREE.Mesh(
  new THREE.PlaneGeometry(200, UPPER_TOP - UPPER_BOTTOM),
  auroraMat
);
upperBg.position.set(0, (UPPER_TOP + UPPER_BOTTOM) / 2, -10);
scene.add(upperBg);
```

- [ ] **Step 2: Update `auroraMat.uniforms.uTime` in frame loop**

Inside `frame()`, before `renderer.render(scene, camera)`:

```javascript
  auroraMat.uniforms.uTime.value = now * 0.001;
```

- [ ] **Step 3: Verify in browser**

Reload. The upper region should now show a slowly breathing aurora — deep violet base, magenta haze top-left, cyan haze middle-right, drifting cloud texture.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(v2): aurora background shader for upper realm"
```

### Task 2.2 — Drop-interior shader (galaxy inside silhouette)

**Files:**
- Modify: `index.html` (replace `dropMaterial` with custom ShaderMaterial)

- [ ] **Step 1: Define drop shader**

Add near the top of the module script, after world-layout constants:

```javascript
const dropVS = `
  varying vec3 vLocalPos;
  varying vec3 vNormal;
  void main() {
    vLocalPos = position;
    vNormal = normalize(normalMatrix * normal);
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
  }
`;

const dropFS = `
  varying vec3 vLocalPos;
  varying vec3 vNormal;
  uniform float uTime;
  uniform float uRealm;       // 0 = full DMT, 1 = clean water
  uniform float uAlpha;
  uniform vec3  uPalette[5];

  // 6-fold mandala pattern
  float mandala(vec2 p) {
    float r = length(p);
    float a = atan(p.y, p.x);
    float petals = cos(a * 6.0) * 0.5 + 0.5;
    float ring = smoothstep(0.4, 0.5, r) - smoothstep(0.55, 0.65, r);
    return petals * ring + smoothstep(0.18, 0.0, r) * 0.6;
  }

  // simple cloud noise
  float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }
  float cnoise(vec2 p) {
    vec2 i = floor(p); vec2 f = fract(p);
    float a = hash(i); float b = hash(i + vec2(1.0, 0.0));
    float c = hash(i + vec2(0.0, 1.0)); float d = hash(i + vec2(1.0, 1.0));
    vec2 u = f * f * (3.0 - 2.0 * f);
    return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
  }

  void main() {
    // sample interior using local sphere coords projected to 2D
    vec2 p = vLocalPos.xy * 0.5;

    // mandala layer (rotates slowly)
    float ma = mandala(vec2(
      p.x * cos(uTime * 0.3) - p.y * sin(uTime * 0.3),
      p.x * sin(uTime * 0.3) + p.y * cos(uTime * 0.3)
    ));

    // cloud layer (drifts)
    float cl = cnoise(p * 3.0 + vec2(uTime * 0.15, -uTime * 0.1));

    // iridescent palette cycle by hue
    float hue = mod(uTime * 0.05 + cl * 0.5, 1.0) * 5.0;
    int pi = int(floor(hue));
    float pf = fract(hue);
    vec3 c0 = uPalette[pi];
    vec3 c1 = uPalette[(pi + 1) % 5];
    vec3 dmt = mix(c0, c1, pf) * (0.4 + ma * 0.6 + cl * 0.3);

    // clean water (for high uRealm)
    float fres = pow(1.0 - max(0.0, dot(vNormal, vec3(0.0, 0.0, 1.0))), 2.0);
    vec3 water = mix(vec3(0.55, 0.75, 0.92), vec3(0.95, 0.98, 1.0), fres);

    vec3 col = mix(dmt, water, uRealm);

    // soft halo at silhouette edge
    float edge = pow(1.0 - max(0.0, dot(vNormal, vec3(0.0, 0.0, 1.0))), 3.0);
    col += vec3(0.6, 0.4, 0.9) * edge * 0.4 * (1.0 - uRealm);

    gl_FragColor = vec4(col, uAlpha);
  }
`;

const DROP_PALETTE = [
  new THREE.Vector3(1.0, 0.43, 0.71),   // magenta
  new THREE.Vector3(0.70, 0.53, 1.0),   // violet
  new THREE.Vector3(0.35, 0.78, 0.98),  // cyan
  new THREE.Vector3(0.30, 0.82, 0.88),  // teal
  new THREE.Vector3(1.0, 0.82, 0.40),   // gold
];

function makeDropMaterial() {
  return new THREE.ShaderMaterial({
    vertexShader: dropVS,
    fragmentShader: dropFS,
    uniforms: {
      uTime:    { value: 0 },
      uRealm:   { value: 0.0 },
      uAlpha:   { value: 1.0 },
      uPalette: { value: DROP_PALETTE },
    },
    transparent: true,
  });
}
```

- [ ] **Step 2: Use the shader for drops**

Replace the existing `dropMaterial` declaration:

```javascript
// (delete the old `const dropMaterial = new THREE.MeshStandardMaterial...`)
```

In `makeDrop()`, replace `dropMaterial.clone()` with `makeDropMaterial()`.

In `triggerFall()`, replace `fallMat` with `makeDropMaterial()`.

- [ ] **Step 3: Update drop uniforms in frame loop**

Inside `updateDrops()`, after `d.mesh.position.set(d.x, d.y, 0);`:

```javascript
    d.mesh.material.uniforms.uTime.value = now * 0.001;
    d.mesh.material.uniforms.uAlpha.value = d.alpha;
    d.mesh.material.uniforms.uRealm.value = 0.0;  // upper realm = full DMT
```

Inside `updateFalling()`, before the position update:

```javascript
    f.mesh.material.uniforms.uTime.value = now * 0.001;
    // realm morph based on y position in void
    const yProgress = (VOID_TOP - f.mesh.position.y) / (VOID_TOP - VOID_BOTTOM);
    f.mesh.material.uniforms.uRealm.value = Math.max(0, Math.min(1, yProgress));
```

(Note: in phase 4 we'll refine this morph; placeholder behavior here.)

- [ ] **Step 4: Verify in browser**

Reload, click intro, play.
Expected:
- Drops on upper plane have iridescent shifting interiors with rotating mandala patterns
- Drops have a soft halo at their edge
- When a drop falls through the void, the interior morphs from DMT-iridescent to whitish/clearer water-like as it descends

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(v2): drop-interior shader (mandala + cloud + iridescent)"
```

### Task 2.3 — Drifting motes (instanced billboards)

**Files:**
- Modify: `index.html` (add InstancedMesh of motes)

- [ ] **Step 1: Add motes**

Insert after the `upperBg` block:

```javascript
// ── Drifting motes ────────────────────────────────────────────────────────
const MOTE_COUNT = 100;
const moteGeom = new THREE.BufferGeometry();
const motePositions = new Float32Array(MOTE_COUNT * 3);
const moteSeeds = new Float32Array(MOTE_COUNT);
for (let i = 0; i < MOTE_COUNT; i++) {
  motePositions[i*3]     = (Math.random() - 0.5) * UPPER_HALF_W * 2.4;
  motePositions[i*3 + 1] = UPPER_BOTTOM + Math.random() * (UPPER_TOP - UPPER_BOTTOM);
  motePositions[i*3 + 2] = (Math.random() - 0.5) * 8;
  moteSeeds[i] = Math.random() * 1000;
}
moteGeom.setAttribute("position", new THREE.BufferAttribute(motePositions, 3));
moteGeom.setAttribute("seed", new THREE.BufferAttribute(moteSeeds, 1));

const moteMat = new THREE.ShaderMaterial({
  uniforms: { uTime: { value: 0 } },
  vertexShader: `
    attribute float seed;
    varying float vSeed;
    uniform float uTime;
    void main() {
      vSeed = seed;
      vec3 p = position;
      // slow brownian drift
      p.x += sin(uTime * 0.1 + seed * 0.7) * 0.6;
      p.y += cos(uTime * 0.08 + seed * 1.3) * 0.4;
      p.z += sin(uTime * 0.12 + seed * 2.1) * 0.3;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(p, 1.0);
      gl_PointSize = 1.5 + 1.5 * sin(uTime * 0.3 + seed);
    }
  `,
  fragmentShader: `
    varying float vSeed;
    void main() {
      vec2 d = gl_PointCoord - 0.5;
      float r = length(d);
      if (r > 0.5) discard;
      float a = (1.0 - r * 2.0) * 0.7;
      gl_FragColor = vec4(1.0, 1.0, 1.0, a);
    }
  `,
  transparent: true,
  depthWrite: false,
});
const motes = new THREE.Points(moteGeom, moteMat);
scene.add(motes);
```

- [ ] **Step 2: Update mote uTime in frame loop**

Inside `frame()`, alongside aurora uniform update:

```javascript
  moteMat.uniforms.uTime.value = now * 0.001;
```

- [ ] **Step 3: Verify in browser**

Reload. Look at the upper region. Expected: ~100 small white dots gently drifting with subtle Brownian motion. They should be subtle, not distracting.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(v2): drifting motes in upper realm"
```

### Task 2.4 — Phase 2 verify + commit

- [ ] **Step 1: Full playthrough**

Reload, play 3 rounds end-to-end. Verify:
- Aurora background breathes
- Drops have shifting iridescent interiors with mandala patterns
- Motes drift gently
- Mechanic is unchanged
- No frame-rate dips (should be smooth 60fps)

- [ ] **Step 2: Push**

```bash
git push
```

---

## Phase 3 — Sacred-geometry mandala companions, god-rays, volumetric haze

End state: upper realm reads as ethereal — independent geometric companions drift; god-rays cut diagonally; depth softens with haze.

### Task 3.1 — Three mandala wireframe companions

**Files:**
- Modify: `index.html` (add 3 wireframe meshes with drift + opacity pulse)

- [ ] **Step 1: Add mandala companions**

Insert after the motes block:

```javascript
// ── Sacred-geometry companions ────────────────────────────────────────────
function makeMandala(form, color) {
  let geom;
  if (form === "icosahedron") geom = new THREE.IcosahedronGeometry(4, 1);
  else if (form === "dodecahedron") geom = new THREE.DodecahedronGeometry(4, 0);
  else geom = new THREE.TorusKnotGeometry(3, 0.3, 64, 8, 2, 5);

  const wf = new THREE.WireframeGeometry(geom);
  const mat = new THREE.LineBasicMaterial({
    color, transparent: true, opacity: 0.18,
  });
  const lines = new THREE.LineSegments(wf, mat);
  return lines;
}

const mandalas = [
  { mesh: makeMandala("icosahedron", 0xc099ff), basePos: [-18, 35, -3], rotSpeed: 0.05, pulseSpeed: 0.7, pulseOffset: 0 },
  { mesh: makeMandala("dodecahedron", 0x5ac8fa), basePos: [22, 22, -5], rotSpeed: 0.04, pulseSpeed: 0.55, pulseOffset: 1.5 },
  { mesh: makeMandala("torusknot", 0xff9ec8), basePos: [0, 12, -7], rotSpeed: 0.035, pulseSpeed: 0.42, pulseOffset: 3.2 },
];
for (const m of mandalas) {
  m.mesh.position.set(...m.basePos);
  scene.add(m.mesh);
}
```

- [ ] **Step 2: Animate mandalas in frame loop**

Inside `frame()`:

```javascript
  for (const m of mandalas) {
    m.mesh.rotation.x += m.rotSpeed * 0.016;
    m.mesh.rotation.y += m.rotSpeed * 0.012;
    const pulse = 0.05 + 0.18 * (0.5 + 0.5 * Math.sin(now * 0.001 * m.pulseSpeed + m.pulseOffset));
    m.mesh.material.opacity = pulse;
  }
```

- [ ] **Step 3: Verify in browser**

Reload. Expected:
- 3 wireframe geometric forms slowly rotating in the upper realm at different positions
- Each fades in and out of opacity at its own rhythm
- They don't interact with the drops; they're just *there*

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(v2): sacred-geometry mandala companions"
```

### Task 3.2 — God-rays (diagonal additive quads)

**Files:**
- Modify: `index.html` (add 3 god-ray quads with shader)

- [ ] **Step 1: Add god-ray shader and meshes**

After the mandalas block:

```javascript
// ── God-rays ──────────────────────────────────────────────────────────────
const godrayMat = new THREE.ShaderMaterial({
  uniforms: { uTime: { value: 0 }, uTint: { value: new THREE.Color(0xffd0a0) } },
  vertexShader: `
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    varying vec2 vUv;
    uniform float uTime;
    uniform vec3  uTint;
    void main() {
      // soft vertical band; shimmers slowly
      float band = exp(-pow((vUv.x - 0.5) * 5.0, 2.0));
      float shim = 0.7 + 0.3 * sin(vUv.y * 30.0 + uTime * 0.3);
      float a = band * shim * 0.10;
      // fade at top and bottom
      a *= smoothstep(0.0, 0.15, vUv.y) * smoothstep(1.0, 0.85, vUv.y);
      gl_FragColor = vec4(uTint, a);
    }
  `,
  transparent: true,
  blending: THREE.AdditiveBlending,
  depthWrite: false,
});

function makeGodRay(x, angle, w = 8, h = 60, tint) {
  const mat = godrayMat.clone();
  mat.uniforms.uTime = godrayMat.uniforms.uTime;  // share time
  if (tint) mat.uniforms.uTint = { value: new THREE.Color(tint) };
  const m = new THREE.Mesh(new THREE.PlaneGeometry(w, h), mat);
  m.position.set(x, (UPPER_TOP + UPPER_BOTTOM) / 2, -8);
  m.rotation.z = angle;
  return m;
}

const godrays = [
  makeGodRay(-15,  0.30, 7, 60, 0xffd0a0),
  makeGodRay(  4, -0.25, 9, 65, 0xc8e0ff),
  makeGodRay( 22,  0.42, 6, 55, 0xffb0d8),
];
for (const r of godrays) scene.add(r);
```

- [ ] **Step 2: Update godray uTime**

Inside `frame()`, alongside other uniform updates:

```javascript
  godrayMat.uniforms.uTime.value = now * 0.001;
```

- [ ] **Step 3: Verify in browser**

Reload. Expected: 3 diagonal soft light shafts cutting through the upper realm with gentle shimmer. They should add depth without dominating.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(v2): god-rays in upper realm"
```

### Task 3.3 — Volumetric haze (depth-based fog on upper realm)

**Files:**
- Modify: `index.html` (add fog layer or shader-based haze)

- [ ] **Step 1: Add a haze quad in front of the upper background**

After the god-rays block:

```javascript
// ── Volumetric haze ───────────────────────────────────────────────────────
const hazeMat = new THREE.ShaderMaterial({
  uniforms: { uTime: { value: 0 } },
  vertexShader: `
    varying vec2 vUv;
    void main() { vUv = uv; gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0); }
  `,
  fragmentShader: `
    varying vec2 vUv;
    uniform float uTime;
    float hash(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }
    float n(vec2 p) {
      vec2 i = floor(p); vec2 f = fract(p);
      float a = hash(i); float b = hash(i + vec2(1.0, 0.0));
      float c = hash(i + vec2(0.0, 1.0)); float d = hash(i + vec2(1.0, 1.0));
      vec2 u = f * f * (3.0 - 2.0 * f);
      return mix(mix(a, b, u.x), mix(c, d, u.x), u.y);
    }
    void main() {
      float h = n(vUv * 2.5 + vec2(uTime * 0.02, 0.0));
      float a = smoothstep(0.4, 0.85, h) * 0.18;
      gl_FragColor = vec4(0.7, 0.65, 0.85, a);
    }
  `,
  transparent: true,
  depthWrite: false,
});
const haze = new THREE.Mesh(
  new THREE.PlaneGeometry(200, UPPER_TOP - UPPER_BOTTOM),
  hazeMat
);
haze.position.set(0, (UPPER_TOP + UPPER_BOTTOM) / 2, -7);
scene.add(haze);
```

- [ ] **Step 2: Update haze uTime in frame loop**

```javascript
  hazeMat.uniforms.uTime.value = now * 0.001;
```

- [ ] **Step 3: Verify in browser**

Reload. Expected: a soft drifting haze across the upper realm, adding depth and mistiness. Should feel like *atmosphere*, not a fog clouds.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(v2): volumetric haze layer in upper realm"
```

### Task 3.4 — Phase 3 verify + commit

- [ ] **Step 1: Full playthrough**

Verify all phase-2 elements still work; the upper realm now feels ethereal — mandalas drift, rays cut diagonally, haze breathes. Drops still play correctly.

- [ ] **Step 2: Push**

```bash
git push
```

---

## Phase 4 — Lower-realm photo-water, transition shader morph, splash particles

End state: the fall *means* something. Lower realm reads as recognizable water on dark stone.

### Task 4.1 — Lower-realm basin mesh + lighting

**Files:**
- Modify: `index.html` (replace `lowerBg` plane with bowl + add directional light for lower)

- [ ] **Step 1: Replace lowerBg with a stone basin**

Find the `lowerBg` block and replace:

```javascript
// ── Lower realm: stone basin ─────────────────────────────────────────────
const basinGeom = new THREE.CircleGeometry(28, 64);
const basinMat = new THREE.MeshStandardMaterial({
  color: 0x2a2522,
  roughness: 0.95,
  metalness: 0.0,
});
const basin = new THREE.Mesh(basinGeom, basinMat);
basin.position.set(0, (LOWER_TOP + LOWER_BOTTOM) / 2, -8);
basin.rotation.x = -Math.PI / 2 + CAMERA_TILT;
scene.add(basin);

// dark backdrop behind basin to fill view
const lowerBackdrop = new THREE.Mesh(
  new THREE.PlaneGeometry(200, LOWER_TOP - LOWER_BOTTOM),
  new THREE.MeshBasicMaterial({ color: 0x0d0c0b })
);
lowerBackdrop.position.set(0, (LOWER_TOP + LOWER_BOTTOM) / 2, -12);
scene.add(lowerBackdrop);

// warm overhead light for lower realm
const lowerLight = new THREE.DirectionalLight(0xffe4b8, 0.8);
lowerLight.position.set(0, 10, 5);
lowerLight.target.position.set(0, (LOWER_TOP + LOWER_BOTTOM) / 2, 0);
scene.add(lowerLight);
scene.add(lowerLight.target);
```

(Remove old `lowerBg` reference in scene.)

- [ ] **Step 2: Verify in browser**

Reload. Expected: a dark circular stone basin at the bottom of the screen, with a subtle warm light catching its surface.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(v2): lower realm basin mesh + warm directional light"
```

### Task 4.2 — Photo-real splash particles + ripple rings

**Files:**
- Modify: `index.html` (replace placeholder splash with particle burst + multiple ripples)

- [ ] **Step 1: Replace `makeSplash` with proper effect**

Find `makeSplash` and replace:

```javascript
function makeSplash(x, y, size) {
  const groupId = Symbol();
  const items = [];

  // 3 expanding ripple rings
  for (let i = 0; i < 3; i++) {
    const ringGeom = new THREE.RingGeometry(0.5, 0.65, 48);
    const ringMat = new THREE.MeshBasicMaterial({
      color: 0xb8d8ec, transparent: true, opacity: 0.0, side: THREE.DoubleSide,
      depthWrite: false,
    });
    const ring = new THREE.Mesh(ringGeom, ringMat);
    ring.position.set(x, y, 0.05 + i * 0.01);
    ring.rotation.x = -Math.PI / 2 + CAMERA_TILT;
    scene.add(ring);
    items.push({ kind: "ring", mesh: ring, delay: i * 80, baseScale: 1, finalScale: 4 + size * 1.2 });
  }

  // particle burst — 14 small spheres arc outward
  const pcount = 10 + size * 2;
  for (let i = 0; i < pcount; i++) {
    const pgeom = new THREE.SphereGeometry(0.25, 8, 6);
    const pmat = new THREE.MeshStandardMaterial({
      color: 0xb8d8ec, roughness: 0.3, metalness: 0.05,
      transparent: true, opacity: 1.0,
    });
    const p = new THREE.Mesh(pgeom, pmat);
    p.position.set(x, y, 0);
    scene.add(p);
    const angle = (i / pcount) * Math.PI * 2 + (Math.random() - 0.5) * 0.4;
    const speed = 0.06 + Math.random() * 0.04;
    items.push({
      kind: "particle", mesh: p,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed * 0.6 + 0.05,
      gravity: -0.0006,
    });
  }

  S.splashes.push({ groupId, items, bornAt: performance.now(), size });
}
```

- [ ] **Step 2: Update `updateSplashes` to handle the new structure**

Replace `updateSplashes`:

```javascript
function updateSplashes() {
  const now = performance.now();
  S.splashes = S.splashes.filter(sp => {
    const t = (now - sp.bornAt) / SPLASH_LIFE_MS;
    if (t >= 1) {
      for (const it of sp.items) scene.remove(it.mesh);
      return false;
    }
    for (const it of sp.items) {
      if (it.kind === "ring") {
        const dt = Math.max(0, t - it.delay / SPLASH_LIFE_MS);
        const localT = Math.min(1, dt / (1 - it.delay / SPLASH_LIFE_MS));
        it.mesh.scale.setScalar(it.baseScale + (it.finalScale - it.baseScale) * localT);
        it.mesh.material.opacity = 0.7 * (1 - localT) * Math.min(1, dt * 4);
      } else if (it.kind === "particle") {
        it.mesh.position.x += it.vx * 16;
        it.mesh.position.y += it.vy * 16;
        it.vy += it.gravity * 16;
        it.mesh.material.opacity = 1 - t * t;
      }
    }
    return true;
  });
}
```

- [ ] **Step 3: Verify in browser**

Reload, complete a fall. Expected:
- On impact, 3 expanding ripples on the basin (staggered)
- 12-15 small water particles arc outward and fall back, fading out
- Effect is visible against the dark basin

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(v2): photo-water splash with ripples + particle burst"
```

### Task 4.3 — Refine transition shader morph (drop interior in void)

**Files:**
- Modify: `index.html` (refine the `uRealm` calculation in `updateFalling`, ensure smooth morph)

- [ ] **Step 1: Refine realm-morph in updateFalling**

The placeholder set `uRealm` based on void-y-progress. Refine to morph from full-DMT (top of fall) to full clean-water (when fall completes):

In `updateFalling()`, replace the realm-uniform update lines:

```javascript
    f.mesh.material.uniforms.uTime.value = now * 0.001;
    const fallT = (now - f.bornAt) / FALL_DURATION_MS;
    // start morphing once drop crosses VOID_TOP, completed by VOID_BOTTOM
    const yPos = f.mesh.position.y;
    let realm = 0;
    if (yPos <= VOID_TOP && yPos >= VOID_BOTTOM) {
      realm = (VOID_TOP - yPos) / (VOID_TOP - VOID_BOTTOM);
    } else if (yPos < VOID_BOTTOM) {
      realm = 1.0;
    }
    f.mesh.material.uniforms.uRealm.value = Math.max(0, Math.min(1, realm));
    f.mesh.material.uniforms.uAlpha.value = 1.0;
```

- [ ] **Step 2: Verify in browser**

Reload, complete a fall. Watch the falling drop closely.
Expected:
- Drop starts the fall with full iridescent DMT interior
- As it crosses the void, the interior shifts visibly toward clearer water
- By the time it impacts the basin, it reads as a recognizable water blob (cyan-white tint, no surreal palette)

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(v2): transition shader morph in void (DMT → water)"
```

### Task 4.4 — Phase 4 verify + commit

- [ ] **Step 1: Full playthrough**

Reload, play 3 rounds. Verify:
- Upper realm: aurora, motes, mandalas, god-rays, haze all visible and breathing
- Drops have shifting iridescent interiors
- On fall: drop morphs to water mid-void
- Lower realm: basin visible, splash includes ripple rings + particle burst
- All v1 mechanic preserved
- 60fps target met (open DevTools perf monitor to check)

- [ ] **Step 2: Push**

```bash
git push
```

---

## Phase 5 — Polish, perf, browser smoke, ship

### Task 5.1 — Perf check + DPR clamp verification

- [ ] **Step 1: Open DevTools Performance, record 10 seconds of play**

Reload, click intro, play one full round while recording.

Expected: average frame ≥ 50fps on target hardware. If frame drops:
- Check if motes are too dense — reduce `MOTE_COUNT` to 60
- Check if shader pixels are too heavy — log `renderer.info.render.calls` and `triangles`
- Reduce `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))` to `1.5` if needed

- [ ] **Step 2: Note any tuning changes in commit; commit if changed**

### Task 5.2 — Chrome + Safari smoke

- [ ] **Step 1: Test in Safari**

Open `http://localhost:8765` in Safari. Play 3 rounds end-to-end. Verify:
- Three.js loads from esm.sh (Safari handles import maps in current versions)
- Aurora and drop shaders render (Safari has historically been picky on custom shaders)
- Audio gesture-unlock works
- No console errors

If Safari fails on import maps, log it and switch to a `<script type="module">` with explicit URL imports.

- [ ] **Step 2: Test in Chrome**

Open the same URL in Chrome. Verify everything works as in dev.

- [ ] **Step 3: If both pass, document; if either fails, fix and recommit**

### Task 5.3 — Update deban with v2 decisions

**Files:**
- Modify: `.deban/_index.md`, `.deban/roles/{pm,arch,dev,ux}.md`, `.deban/session-log.md`

- [ ] **Step 1: Append session-log entries**

Edit `.deban/session-log.md`, append:

```
2026-05-01 — V2 BRAINSTORM — Two-realm reframe approved. Upper realm = DMT/non-human (geometric+amorphous, fading); lower realm = recognizable water. Drop shape preserved as silhouette in upper, interior is "galaxy". Spec at docs/superpowers/specs/2026-05-01-inochi-v2-two-realms-design.md.

2026-05-01 — V2 BUILD COMPLETE — phases 1-5 implemented on feat/v2-two-realms. Three.js via esm.sh importmap, single HTML preserved, mechanic ported verbatim. Subsurface/bloom/refraction parked for future mini-challenges.
```

- [ ] **Step 2: Append decisions to relevant role files**

In `.deban/roles/arch.md ## Decisions`:

```
| 2026-05-01 | v2 render pipeline = Three.js via esm.sh importmap, inline GLSL shaders, single HTML preserved | Iteration justified the dependency; Canvas2D + SVG-filter capped what "ethereal" could be | [[dev]] [[devops]] |
| 2026-05-01 | World coords (orthographic): UPPER_TOP=50, VOID_TOP=5, VOID_BOTTOM=-7, LOWER_BOTTOM=-30; CAMERA_TILT=-25° | Per spec viewport proportions | [[dev]] |
```

In `.deban/roles/ux.md ## Decisions`:

```
| 2026-05-01 | Two realms with vertical bisection: upper DMT, void transition, lower photo-water | Reframe of original brief during v2 brainstorm | [[pm]] [[arch]] |
| 2026-05-01 | Aurora-cool palette for upper realm: deep violet base, magenta+cyan accents, hint of pink/gold | Saturated but not neon; surreal not gaudy | [[arch]] |
| 2026-05-01 | Sacred-geometry mandala companions: 3 visible, drifting independently of drops, opacity pulse 0.05–0.28 | Per spec; non-interactive decoration | [[dev]] |
```

In `.deban/roles/pm.md ## Decisions`:

```
| 2026-05-01 | v2 ship gate (additional): both realms visually distinct on first look, drop transformation visible mid-fall, v1 mechanic ship-gate still passes | Falsifiable, distinct from v1 | [[qa]] |
```

In `.deban/_index.md ## Key Decisions`:

```
- v2 = two-realm reframe (2026-05-01) — vertical bisection: DMT upper / void / photo-water lower [[arch]] [[ux]] [[pm]]
- Three.js via esm.sh importmap (2026-05-01) — single-HTML constraint preserved, dependency justified by iteration scope [[arch]]
- Subsurface/bloom/refraction parked for future mini-challenges (2026-05-01) — per-scene ethereal vocabulary [[ux]]
```

- [ ] **Step 3: Commit deban updates**

```bash
git add .deban
git commit -m "deban: log v2 brainstorm + build decisions"
```

### Task 5.4 — Open PR

- [ ] **Step 1: Push final branch**

```bash
git push
```

- [ ] **Step 2: Open PR via gh CLI**

```bash
gh pr create --title "feat(v2): two-realm visual rewrite (Three.js)" --body "$(cat <<'EOF'
## Summary
- Visual rewrite from Canvas2D + SVG metaball + CSS rotateX → Three.js orthographic 3D
- Two visually distinct realms: DMT-vocabulary upper (sacred geometry, aurora, motes, mandalas, god-rays, haze, iridescent drop interiors) and photo-real lower (stone basin, splash particles, ripples)
- Void transition: drops morph from DMT-substance to clean water as they fall
- v1 mechanic preserved verbatim (centroid, retarget, density-driven merge, evaporation, 3-round fade transitions)
- Single HTML file preserved; Three.js loaded via esm.sh importmap

## Spec
docs/superpowers/specs/2026-05-01-inochi-v2-two-realms-design.md

## Test plan
- [ ] Open http://localhost:8765 in Chrome, play 3 rounds end-to-end
- [ ] Click "the cave listens" — audio context unlocks
- [ ] Successfully merge drops in round 1 (tetro), watch fall + splash
- [ ] Verify shader morph in void (DMT → water)
- [ ] Open in Safari, repeat
- [ ] DevTools perf: ≥ 50fps average

## Parked for future iterations
- Subsurface drop-glow, bloom halo, refraction — earmarked as per-scene ethereal vocabulary for v3+
- Realm-differentiated audio (alien upper vs recorded water lower) — v3
EOF
)"
```

- [ ] **Step 3: Capture PR URL from output**

### Task 5.5 — Notify Gerald via Telegram

- [ ] **Step 1: Send Telegram**

```bash
source ~/.config/kainode/telegram.env
PR_URL="<paste from previous step>"
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage" \
  -d chat_id="496805857" \
  -d text="*inochi-no-mizu v2* — two-realm visual rewrite shipped on \`feat/v2-two-realms\`. PR: ${PR_URL}. Live: http://localhost:8765 (local) or pages will pick up after merge. Mechanic preserved, drop interiors are now galaxies, fall is a translation between realms." \
  -d parse_mode="Markdown"
```

- [ ] **Step 2: End of plan**

All phases complete. Spec → plan → code lifecycle done.

---

## Self-review (run after writing this plan)

**Spec coverage:**
- [x] Section 1 world structure → Task 1.1 (regions), 1.4 (drops in upper), 4.1 (basin), 4.2 (splash)
- [x] Section 2 visual treatment per realm — upper: 2.1 (aurora), 2.2 (drop interior), 2.3 (motes), 3.1 (mandalas), 3.2 (god-rays), 3.3 (haze); lower: 4.1 (basin), 4.2 (splash); void: 4.3 (transition shader)
- [x] Section 3 stack + scope — Task 1.1 (importmap), all phases ship reviewable commits, branching strategy in plan header, audio reused in 1.2, DPR clamp in 1.1 + 5.1
- [x] Out of scope — parked items called out in 5.3 deban update
- [x] Success criteria — Task 5.1 perf, 5.2 browser smoke, mechanic check in every phase

**Placeholder scan:** No TBDs/TODOs in the plan. All shaders have actual GLSL. All Three.js setup has actual JS. The only "TBD" is the PR URL captured at runtime in 5.4 → 5.5, which is correct.

**Type consistency:** drop material uses `makeDropMaterial()` consistently from task 2.2 onward. Constants (DROP_RADIUS, MERGE_DIST_FACTOR, etc.) are defined once in 1.1 and referenced throughout. Function names (`startRound`, `updateDrops`, `triggerFall`, `updateFalling`, `updateSplashes`, `endRoundAndAdvance`) are used identically across tasks.

**Scope check:** This is a single coherent rewrite, not multiple subsystems. One spec → one plan. The phases are sequential, each builds on the prior. No need to decompose further.
