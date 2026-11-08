# Tesla Bird

A faithful **Flappy Bird** clone where the bird is Tesla's **2026 second-generation Roadster**,
soaring between Manhattan skyscrapers at dusk. Single file, no build step, no dependencies.

## Run it

Desktop:

```sh
open public/index.html
```

iPhone / iPad (over your LAN):

```sh
python3 -m http.server 8000 --directory public
```

Then, on the phone (same Wi-Fi), open `http://<your-computer-ip>:8000/`.
Find your IP with `ipconfig getifaddr en0` on macOS.

## Controls

- **iPhone / Safari:** tap anywhere to fire the cold-gas thrusters.
- **Desktop:** `Space`, `ArrowUp`, or click to fly. `Space` also starts and restarts.
- One tap = one flap (multi-touch is debounced).
- Corner button toggles sound (persisted).

## Gameplay

- `ready` (title + bobbing car) → `playing` → `gameover` (miles, farthest, restart). One tap restarts.
- **Progress is measured in real-life terms:** each tower you thread adds **0.1 mi** to your distance
  (`MI_PER_TOWER`), and a run clock counts **days survived** (`DAY 1`, `DAY 2`, … every 3 s —
  `SECONDS_PER_DAY`). The HUD shows `1.4 mi` big with `DAY 14` beneath it.
- The high score is **farthest distance**, saved in `localStorage` (`roadsterBirdBest`, stored as the
  tower count so existing bests carry over).
- Hit any tower, the roof line, or the ceiling and the run ends — the Roadster **combusts**:
  it bursts into flying body panels, wheels, wing and glass shards, wreathed in a fireball and a
  cold-gas smoke plume, with a deep boom, a screen shake and a white flash. The **driver is flung
  clear** in a flailing spacesuit (with a comic yelp) — he tumbles end-over-end, limbs windmilling,
  and bounces across the street. Debris and driver settle and then clear on restart.

## The car

The Roadster sprite is `roadster-glm.svg` — a wedge-shaped 2026 Roadster in "Ano White" with a
gold-glazed canopy, a deployed active rear wing, six-spoke wheels, a full-width taillight bar and
rear cold-gas thruster nozzles. That SVG is **inlined into `public/index.html` as a data-URI** (with the
`<use>` references expanded) so the game stays fully self-contained and offline. It is drawn onto
the canvas **mirrored** so the nose faces the direction of travel, and it pitches nose-up /
nose-down with vertical velocity exactly like Flappy Bird.

The **underbody cold-gas thrusters** fire a white vapor plume straight down (with a blue-white glow
sheet under the car) on every flap — the in-fiction "wing flap", echoing the reveal car's
thruster/fan system. A pop-up headlight flash fires on each flap, and there is an optional underglow
when skimming the rooftops. On a crash the sprite is replaced by exploding debris panels (see
Gameplay).

Art is canvas + inline SVG — no external image files.

## The world

- Dusk gradient sky, drifting clouds, three parallax skyline layers (with lit windows), and a
  faint Empire State Building drifting in the far distance.
- Obstacles are skyscrapers: an upper tower hanging from the top and a lower tower rising from the
  ground, with rooftop water towers / antennas / cornices. **Collision geometry and gap size are
  unchanged from Flappy Bird** — facade variety is cosmetic only.
- A scrolling NYC street band with dashed lane markings forms the ground.

## Technical notes

- Canvas 2D, `requestAnimationFrame`, delta-time physics with 120 Hz substepping, so it's
  frame-rate independent across 60 Hz / 120 Hz displays.
- Responsive + retina: sized to the viewport with `devicePixelRatio` scaling; handles resize and
  orientation changes. Difficulty is resolution-independent (fixed logical space of 512 units tall).
- iOS Safari hardening: locked scroll/overscroll, `touch-action: none`, fixed layout with safe-area
  insets, blocked double-tap zoom / gestures / text selection, Web Audio primed on first gesture,
  clean pause + audio suspend on `visibilitychange` / blur.
- Audio is synthesized with WebAudio (thruster whoosh, score chime, hit thud, explosion boom,
  comic yelp when the driver is ejected) — no asset files.
- Pooled obstacle, particle and crash-debris objects; no per-frame allocation of game entities.
- Crash debris are **clipped slices of the pre-rendered Roadster sprite** (the sprite is blitted once
  to an offscreen bitmap, then shattered along the car's real seams — panels, glass, wing, and both
  whole wheels), so the flying pieces match the car's actual shapes and colours exactly. A flat
  polygon set is kept as a fallback if the sprite hasn't decoded yet.

## Assumptions

- The 2026 Roadster is `roadster-glm.svg` (a white wedge next-gen coupe with a deployable wing and
  cold-gas thrusters); it is inlined as a data-URI and scaled to `IMG_W = 70` logical units. Art is
  stylized.
- Flappy Bird tuning constants were chosen to match the original's feel on a 512-unit-tall
  reference screen: gravity `900`, flap impulse `-272`, scroll `128`, gap `100` (≈0.195 H),
  pipe width `56`, pipe spacing `194`, terminal velocity `540`.
- On game over, one tap returns to the title screen (as in the original); a second tap starts
  the run. Keyboard restarts directly.
- `prefers-reduced-motion` reduces screen-shake, crash vapor, clouds and the explosion's
  fire/smoke count (the car still breaks apart into debris).

## Verify

- Open the console — no errors.
- Tap / click / `Space` all flap; one flap per tap.
- Distance increments exactly once per gap passed (`0.1 mi` each); `DAY N` ticks up on the run clock
  and freezes at the crash.
- Collisions: tower (upper + lower), ground, and ceiling all end the run.
- On any crash the Roadster shatters into debris that matches the car (real sprite slices) and
  tumbles and settles, the driver is flung out flailing, and both are cleared when you restart.
- Reload the page — the farthest distance persists.
- On iPhone: no zoom, no scroll bounce, no tap highlight.

## Deploy

Live at <https://teslafs.awllly.com>.

Vercel project `teslafs` (team `awlllys-projects`), built from this repo:
Framework Preset **Other**, no build command, Output Directory `public`.
