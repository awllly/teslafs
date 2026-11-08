# session_memory.md — Tesla Bird

> Hand-off notes for an AI agent (or human) who has **not** seen the build conversation.
> Read this top-to-bottom before changing anything. It covers what the project is, how it's
> built, every feature that was added, the invariants you must not break, and how to verify.

---

## 1. What this is

**Tesla Bird** (formerly "Roadster Bird") is a faithful **Flappy Bird clone** built as a **single,
self-contained `index.html`** (inline CSS + JS, no build step, no runtime dependencies, works
offline by double-clicking the file).

- The "bird" is the **2026 Tesla Roadster** (second-gen, "cyberfied" wedge, "Ano White"), soaring
  between **Manhattan skyscrapers** at dusk.
- Tap (iPhone) / click / `Space` / `ArrowUp` (desktop) to flap (cold-gas thrusters fire downward).
- Progress is scored in flavour units: **miles** (0.1 mi per tower) + **days survived**
  (elapsed-time clock).
- Crash = combustion: the Roadster bursts into **sprite-slice debris**, and a **spacesuit driver is
  flung out flailing**.
- The original spec is `PROMPT.md` (still in the repo). `README.md` is the user-facing doc.

**Current status:** feature-complete and verified. No known bugs. See §14 for likely next steps.

---

## 2. Repo files

| File | Purpose |
|---|---|
| `index.html` | **The whole game.** ~1680 lines, everything inline. This is the only file that ships. |
| `PROMPT.md` | Original design brief (kept for reference). |
| `README.md` | User-facing docs: how to run, features, assumptions, verify checklist. |
| `roadster-glm.svg` | **Source car art** (159 lines, `viewBox 0 0 920 340`). This is the sprite that was inlined into `index.html`. Not loaded at runtime. |
| `extracted_car.svg` | Build/reference artifact: the exact SVG the game builds and inlines, dumped to disk. Not loaded at runtime. Safe to delete. |
| `.DS_Store` | macOS junk. Ignore. |

> ⚠️ The working directory has been observed to change between turns (other tools/agents adding or
> removing files). Always re-`ls` before assuming a file exists.

---

## 3. How to run / verify

```sh
open index.html                                  # desktop
python3 -m http.server 8000                       # then open http://<lan-ip>:8000 on phone
```

Quick static checks (no browser needed):

```sh
# 1. syntax check of the inline script
node -e "const fs=require('fs');const m=fs.readFileSync('index.html','utf8').match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync('/tmp/rb.js',m[1]);"
node --check /tmp/rb.js && echo SYNTAX OK
```

For behavioural testing there is a **stubbed-DOM harness** recipe — see §13. There is no test
suite committed; the harness lives in a temp dir and must be recreated if needed.

---

## 4. `index.html` architecture

Everything is inside one IIFE `(function(){ 'use strict'; ... })();`. Rough section order:

1. **HTML head**: meta viewport (iOS-hardened), theme-color, inline CSS.
2. **Body**: `<canvas id="game">`, `#hud` (`#miles`, `#days`), `#mute` button, `#overlay`
   (`#panel-ready`, `#panel-over`).
3. **Script** (in this order):
   - Constants (§5).
   - DOM lookups + mutable state vars.
   - **Audio** synth: `initAudio`, `resumeAudio`, `sfxThruster`, `sfxScore`, `sfxHit`,
     `sfxExplode`, `sfxYelp`, `applyMute`.
   - **resize** (retina scaling, `view`, `groundY`, `carX`, `skyGrad`).
   - RNG helpers `mulberry32`, `hash3`.
   - **Parallax skyline** `makeLayer` → `farLayer`, `midLayer`, `nearLayer`; `clouds`.
   - **Obstacles** pool: `getObstacle`, `recycleObstacle`, `resetObstacles`.
   - **Particles** pool: `getPart`, `emitThruster`, `emitExplosion`, `updateParticles`.
   - **Crash debris**: `SPRITE_SHARDS`, `shardLocal`, `getDebris`, `recycleDebris`,
     `launchDebris`, `explodeCar`, `updateDebris`, `clearDebris`, `drawDebris`.
   - **Ejected driver**: `ejectDriver`, `updateDriver`, `driverLimb`, `rrect`, `drawDriver`.
   - **HUD/overlays**: `milesFor`, `daysFor`, `formatMiles`, `updateHud`, `showPanel`.
   - **Game flow**: `resetGame`, `startRun`, `die`, `flap`.
   - **Input**: `onPress` + `pointerdown`/`keydown` listeners; iOS gesture hardening;
     visibility/blur pause; resize listeners.
   - **Update**: `physics`, `updateObstacleMotion`, `checkCollision`, `update`.
   - **Drawing helpers**: `drawClouds`, `drawSkylineLayer`, `drawEmpire`, `drawTower`,
     `drawGround`, `drawParticles`, the **car sprite builder** `buildCarSprite` (IIFE), `drawCar`,
     `drawUnderglow`.
   - **Render**: `render` (draw order — see §9).
   - **Main loop**: `frame` (rAF + dt clamp).
   - **Boot**: `resize(); applyMute(); updateHud(); showPanel('ready'); requestAnimationFrame(frame);`

---

## 5. Constants (do not change casually)

```js
LOGICAL_H = 512          // world height in logical units; width derives from aspect ratio
GRAVITY   = 900          // u/s²
FLAP_V    = -272         // u/s impulse
MAX_FALL  = 540          // terminal velocity
SCROLL    = 128          // u/s world scroll
GAP       = 100          // vertical gap (0.195 * H — Flappy-identical ratio)
PIPE_W    = 56           // tower width
SPACING   = 194          // horizontal spacing between towers
GROUND_H  = 96           // street band height
MI_PER_TOWER   = 0.1     // towers → miles
SECONDS_PER_DAY = 3.0    // flight time → "DAY N"
IMG_W = 70               // drawn sprite width (logical units)
IMG_H = IMG_W*340/920    // ≈ 25.87, locks the SVG aspect
HIT_W = 42, HIT_H = 20   // collision box (smaller than sprite)
BASE_Y = LOGICAL_H*0.42
READY_SCROLL = SCROLL*0.55
```

**These were tuned to match Flappy Bird's feel** on a 512-tall reference screen. If you change
`GRAVITY`/`FLAP_V`/`GAP`/`SPACING`, difficulty changes for everyone — do it deliberately.

---

## 6. Coordinate / rendering model

- **Logical space**: height is always `512` units; `view.w = cssWidth / (cssHeight/512)`.
  So the world is resolution-independent (same difficulty on any screen).
- `render()` starts with `ctx.setTransform(scale*dpr, 0, 0, scale*dpr, 0, 0)` where
  `scale = cssHeight/LOGICAL_H` and `dpr = min(devicePixelRatio, 3)`.
- The **car is drawn mirrored** so its nose faces right (direction of travel):
  `translate(carX, car.y) → rotate(car.angle) → scale(-1,1) → drawImage(...)`.
- **`car.angle`**: derived from vertical velocity, `clamp(atan2(vy,520), -0.62, 1.35)` — nose-up on
  flap, nose-down on fall. Same idea for the sprite rotation.
- **Sprite math (important for debris):** the sprite's image coords `(u,v)` in `[0..920]×[0..340]`
  map to local car units via
  `lx = -(u-460) * k`, `ly = (v-170) * k`, where `k = IMG_W/920 = 0.07608…` (`shardLocal()`).
  Real wheel centres in local units: front `(+20.7, +1.75)`, rear `(-14.0, +1.75)`, radius `≈6.3`.

---

## 7. State machine & flow

`state ∈ 'ready' | 'playing' | 'gameover'` (+ a separate `paused` flag).

- **ready**: title panel; car bobs (`BASE_Y ± sin(t*2.6)*7`); world scrolls at `READY_SCROLL`.
- **playing**: full physics; `runTime` accrues; obstacles spawn/score; HUD updates.
- **gameover**: physics stops; car hidden (`carExploded = true`); debris + driver animate/fall;
  `#panel-over` ("WRECKED") is shown.

Flow functions:
- `startRun()` — resets score/runTime/HUD, sets `state='playing'`, first flap, hides panels.
- `die()` — sets gameover, explodes car, ejects driver, persists best, fills the game-over panel.
- `resetGame()` — back to `ready`, clears debris/driver/particles, resets HUD.
- `flap()` — only in `playing`: `car.vy = FLAP_V`, `headT = 1`, emits underbody thrust, plays SFX.

Restart behaviour: on gameover, a tap returns to **ready** (needs `now - gameOverAt > 450 ms`);
keyboard uses `> 300 ms`. A second tap starts the run. `#restart` button calls `resetGame()`
directly.

---

## 8. Input & iOS hardening

- **Flap input**: `window` `pointerdown` (mouse left-button filtered) → `onPress()`.
  `onPress` debounces multi-touch/duplicate input: ignores if `now - lastInput < 90 ms`.
  Ignores taps whose target is a `<button>` (so mute/restart work).
- **Keyboard**: `Space`/`ArrowUp` (and `e.key` variants); `preventDefault`; ignores `e.repeat`.
- **iOS/Safari**: meta viewport with `viewport-fit=cover, user-scalable=no, maximum-scale=1`;
  `touch-action:none`, `overscroll-behavior:none`, `-webkit-tap-highlight-color:transparent`,
  fixed body; gesture (`gesturestart/change/end`), `touchmove`, `dblclick`, `contextmenu` all
  `preventDefault`ed. HUD uses `env(safe-area-inset-*)`.
- **Pause**: `visibilitychange`/`blur` set `paused=true` (and suspend audio); `focus` resets the
  loop clock (`last = performance.now()`).

---

## 9. Render order (`render()`)

`sky gradient → screen-shake translate → clouds → Empire State (far parallax) → far/mid/near
skyline layers → towers → ground → underglow → debris → particles → car → driver → restore →
crash flash overlay`.

Particles use `globalCompositeOperation='lighter'` for fire and reset to `'source-over'` each
iteration.

---

## 10. The car sprite pipeline (important)

1. `buildCarSprite()` (an IIFE near the bottom) constructs the car SVG **as a JS string** from
   `roadster-glm.svg`, with `<use>` references **expanded** (SIL path + a 6-spoke `WHEEL` string +
   detail paths) so no `<use>`/`xlink` is needed.
2. It sets `img.src = 'data:image/svg+xml;charset=utf-8,' + encodeURIComponent(svg)`.
3. On `img.onload`: sets `carImgReady = true`, then **pre-renders a mirrored bitmap** into an
   offscreen canvas `carSprite` (`SPR_W×SPR_H` = 4× IMG size) for crisp debris slices, and sets
   `carSpriteReady = true`.
4. `drawCar()` draws `carImg` (the SVG image) with the mirror transform described in §6.
5. **Do not** reference `roadster-glm.svg` from HTML/CSS — the art is inlined to keep the file
   self-contained/offline. If you change the car art, change the **string in `buildCarSprite()`**
   (and update `roadster-glm.svg` + `extracted_car.svg` for consistency if you like).

---

## 11. Crash system

Triggered by `die()` (any tower/ground/ceiling collision):

- `carExploded = true` → `drawCar()` returns early (car hidden).
- `explodeCar()` — spawns debris pieces:
  - If `carSpriteReady`: iterate **`SPRITE_SHARDS`** (9 seam-following polygons + 2 whole-wheel
    circles, authored in **SVG image coordinates**) and map each into local space via `shardLocal()`.
  - Else: fall back to **`DEBRIS`** (legacy flat polygons) + 2 legacy wheels.
- `emitExplosion()` — fireball particles (`kind:'fire'`), smoke (`kind:'smoke'`), and a downward
  cold-gas burst from the underbody nozzles.
- `ejectDriver()` — spawns the flailing spacesuit driver (see below) + a thrust puff.
- `shake = reduceMotion ? 5 : 14`, `crashFlash = 1`; SFX: `sfxHit`, `sfxExplode`, `sfxYelp`.
- Best distance persisted here (`roadsterBirdBest`).

### Debris rendering (`drawDebris`)
- Sprite shards set `d.img = true`, `d.v` (local polygon) or `d.wheel`/`d.r`, plus centroid
  `d.cx/cy`. Draw = `translate(d.x,d.y) → rotate(d.rot) → translate(-cx,-cy) → clip(polygon|circle)
  → drawImage(carSprite, -IMG_W/2, -IMG_H/2, IMG_W, IMG_H)`. This makes every piece a literal slice
  of the car (exact colours/glass/window lines/wheels).
- Legacy pieces (fallback) fill a flat colour.
- Physics: gravity `GRAVITY*0.9`, drag, spin; settles on the street (`groundY-2`), fades in the last
  0.8 s, removed at `life<=0`. Pooled via `debrisPool`.

### Ejected driver (`drawDriver` / `updateDriver`)
- Single `driver` object: launches up-and-back with spin, up to 3 bounces, then settles.
- Drawn as a **spacesuit ragdoll**: two-segment limbs whose elbows/knees `windmill` from time-based
  sine phases, white helmet + dark visor, chest panel with red logo, life-support pack. `rrect()`
  is a local rounded-rect helper. `prefers-reduced-motion` shrinks the flail/spin.

### Thruster FX (`emitThruster`)
- On each flap, emits vapor from a row of 5 **underbody nozzles** (`THRUST_X = [-21,-10.5,0,10.5,21]`)
  blasting **straight down** (local +y, fanning outward), rotated by `car.angle`.
- `drawCar()` draws a flattened blue-white **underglow sheet** beneath the car sized to the plume.

---

## 12. Metrics & HUD

- **Miles** = `score * MI_PER_TOWER` (0.1 per tower) → shown big: `1.4 mi`.
- **Days** = `1 + floor(runTime / SECONDS_PER_DAY)` (starts at `DAY 1`) → shown small beneath.
  `runTime` accrues **only** in the `playing` branch, so it freezes at the crash.
- **`updateHud()`** writes the DOM only when a value changes (cached `hudMiles`/`hudDays`). Called
  on score events, each `playing` frame, and on reset. Reset caches to `-1` to force a repaint.
- **Best** is stored as the **tower count** in `localStorage['roadsterBirdBest']` and displayed as
  `FARTHEST` miles. (Deliberately kept the old key so existing bests survive the rename.)
- Game-over panel (`#panel-over`): heading **`WRECKED`**, cells `MILES` / `FARTHEST`, footer
  `Survived N days`, button `FLY AGAIN`.
- `localStorage['roadsterBirdMuted']` = `'1'`/`'0'`.

---

## 13. Testing harness recipe (temp, no browser)

There is no committed test suite. To test headlessly, create a **stubbed-DOM harness** in a temp
dir (e.g. `/var/folders/…/T/opencode/`). Essentials:

- Extract the script: `html.match(/<script>([\s\S]*?)<\/script>/)[1]`, run in `vm.createContext`.
- Stub `document` with: `getElementById` (memoize into a map), `addEventListener` (collect
  listeners so you can `dispatch('pointerdown', …)` and `'keydown'`), `hidden`, and
  **`createElement: () => ({width:0,height:0,style:{},getContext:()=>ctxStub})`** (needed by the
  offscreen sprite pre-render).
- Stub `Image` so the `src` setter synchronously calls `onload` (so `carImgReady`/`carSpriteReady`
  become true): `function(){ var s=this; Object.defineProperty(s,'src',{set:v=>{if(s.onload)s.onload();}}); }`
- Provide `encodeURIComponent`, `localStorage` (a plain object), `Math`, `Uint8Array`, etc.
- `ctx` can be a `Proxy` returning no-op functions; special-case `createLinearGradient`/
  `createRadialGradient` to return `{addColorStop(){}}`.
- **Clock gotcha:** the game uses `performance.now()` and the rAF timestamp from the *same* clock.
  If they differ, the game-over restart debounce (`now - gameOverAt`) and the input debounce break.
  Make `performance.now = () => clock` and drive frames with `frame(clock += dtMs)`.
- rAF: capture the callback and call it manually:
  `sandbox.requestAnimationFrame = cb => { rafCb = cb; return 1; }`.

A run then looks like: boot → `dispatch('pointerdown')` to start → step frames → assert DOM text
(`els.miles.innerHTML`, `els.days.textContent`, `els.finalMiles.textContent`, …) and panel
visibility. A simple autopilot that flaps when below the next tower's `gapY + ~20` (with `vy > -60`)
can rack up points to exercise scoring.

**Debug hook recipe:** to inspect internals during a test, string-replace the boot tail to expose a
probe, e.g. inject before the final `requestAnimationFrame(frame);`:
`window.__probe = function(){ return {score, runTime, state, car, obstacles, carX, groundY}; };`
Remove it afterwards — **do not leave debug hooks in the shipped file.**

---

## 14. Change log (chronological)

1. **Initial build** from `PROMPT.md`: single-file Flappy clone, canvas 2D, delta-time with 120 Hz
   substepping, NYC theme (parallax skyline, dusk sky, Empire State, street ground), skyscraper
   obstacles, WebAudio SFX, iOS hardening, `localStorage` best. Written as `Roadster Bird`.
2. **Car art overhaul** after researching the real 2026 Roadster design: faceted white wedge,
   V-crease hood, hidden pop-up lights, full-width rear bar, thin-spoke wheels, gold glass,
   titanium trim, scissor-door flourish (procedural canvas drawing). *(Later superseded.)*
3. **Crash combustion**: added explosion (fire/smoke particles), screen shake, flash, and debris
   pieces. *(Debris then used hand-authored flat polygons.)*
4. **Ejected driver**: flailing spacesuit ragdoll on crash, with a comic yelp SFX.
5. **Swapped the car to `roadster-glm.svg`**: inlined as a data-URI, drawn mirrored; removed the old
   procedural car drawing (and its now-dead `rr`/`carBodyPath`/`drawWheel`/`car.wing`/`doorT`).
6. **Underbody thrusters**: flap vapor now emits straight down from an underbody nozzle row, with an
   underglow sheet (matches the reveal video).
7. **Debris fix**: pieces are now **true slices of the pre-rendered sprite** (`carSprite` +
   `SPRITE_SHARDS`), so they match the car exactly; legacy polygons kept as a fallback.
8. **Metrics**: replaced the tower "Score" with **miles + days survived**; game-over heading became
   `WRECKED`; best relabeled `FARTHEST` (still stored as tower count).
9. **Rename** to **Tesla Bird** (tab title, start-screen `<h1>`, README title). Vehicle references and
   the `roadsterBirdBest` key intentionally unchanged.

---

## 15. Invariants & gotchas — read before editing

- **Single file, offline, no deps.** Everything must stay inline in `index.html`. No external
  fetches, no `<script src>`, no imported fonts/images (car art is a data-URI).
- **Keep the hitbox/difficulty stable.** `HIT_W/HIT_H`, the Flappy constants (§5), and collision
  logic (`checkCollision`) define difficulty. Art/sprite changes must **not** alter collision.
- **Sprite vs world orientation.** The car image faces **left**; the world moves right, so it's drawn
  with `scale(-1,1)`. Debris shards are authored in *image* coords and mapped with `shardLocal()`;
  if you re-author the art, recompute those mappings.
- **Debris depends on `carSprite`.** If you change the sprite, `SPRITE_SHARDS` may need retuning so
  pieces still line up with the car's features.
- **`localStorage` keys are load-bearing:** `roadsterBirdBest` (towers) and `roadsterBirdMuted`.
  Renaming keys wipes user data.
- **Performance:** obstacles, particles, and debris are **pooled**; avoid per-frame allocations in
  the update/render loop. Keep the skyline/parallax cheap.
- **`prefers-reduced-motion`** must keep degrading: screen-shake, explosion particle counts, driver
  flail/spin, cloud drift, door/thruster flourishes.
- **Verify after every change:** `node --check` on the inline script **and** run the stubbed-DOM
  harness through boot + play + crash + restart with no thrown errors. If you add DOM elements,
  make sure your harness's `getElementById` fallback still returns a usable stub.

---

## 16. Likely next steps / ideas (not started)

- Split the deployed rear wing out of the sprite into its own layer so it can **animate** (the old
  procedural wing pitched with velocity; the current SVG has it baked in).
- Per-shard wind/drag so debris and the driver interact with the parallax more convincingly.
- Optional: a small "trip computer" styling pass on the HUD, or a `Mission Elapsed Time` toggle.
- A committed, reproducible test harness (currently temp-only).
- Accessibility: a visible on-screen flap affordance for users who don't discover tap-to-flap.

---

## 17. Quick reference — where things live

| Concern | Function(s) in `index.html` |
|---|---|
| Physics / collision | `physics`, `updateObstacleMotion`, `checkCollision` |
| Game flow | `resetGame`, `startRun`, `die`, `flap`, `update` |
| Input | `onPress` + `pointerdown`/`keydown` listeners |
| Car drawing | `buildCarSprite` (IIFE), `drawCar` |
| Crash FX | `explodeCar`, `emitExplosion`, `drawDebris`, `ejectDriver`, `drawDriver` |
| Thruster FX | `emitThruster`, `updateParticles`, `drawParticles` |
| Metrics/HUD | `milesFor`, `daysFor`, `formatMiles`, `updateHud`, `showPanel` |
| World drawing | `drawClouds`, `drawSkylineLayer`, `drawEmpire`, `drawTower`, `drawGround` |
| Loop / frame | `frame`, `render` |
