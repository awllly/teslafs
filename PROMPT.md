# Prompt: "Roadster Bird" — a Flappy Bird clone for the 2026 flying Tesla Roadster

Build a complete, self-contained web game — a faithful Flappy Bird clone — where the bird is Tesla's **second-generation Roadster** (the car revealed at SpaceX's McGregor, TX facility on October 15, 2026), soaring between New York City skyscrapers. It must run great in Safari on an iPhone (tap to fly) and on desktop (spacebar to fly).

## Core gameplay (must mimic Flappy Bird exactly)
- Constant forward scroll; the player taps/clicks/presses to apply an upward impulse to the Roadster.
- Gravity pulls it down continuously; each tap gives the same fixed upward velocity (no holding to sustain lift).
- Endless sequence of gap obstacles: on tap, score +1 and play a chime; hitting an obstacle or the ground/ceiling ends the run immediately.
- Score displayed top-center; persistent high score saved in `localStorage`.
- State machine: `ready` (title + instructions) → `playing` → `gameover` (score, best, restart). One tap/space restarts.
- Difficulty tuning identical in spirit to Flappy Bird: fixed gravity, impulse, scroll speed, and gap size — brutally but fairly hard. Include the subtle pre-first-tap bobbing "ready" state.

## Controls
- **iPhone/Safari:** tap anywhere to jump. Prevent double-tap zoom, text selection, scroll bounce, and the iOS tap highlight.
- **Desktop:** `Space`/`ArrowUp`/click jump; `Space` also starts/restarts.
- Debounce multi-touch so one tap = one flap.

## Skin & theme
### Player — the 2026 Tesla Roadster (not the Starman car)
Use the actual next-gen Roadster design language:
- **Body:** low, wedge-shaped coupe in **glossy white** (the reveal car was white), with a black lower aero section, dark glass canopy, and thin LED light bars. Silhouette inspired by the SR-71 Blackbird / F-35 and the white Lotus Esprit S1 — draw it cleanly with canvas shapes or inline SVG. Placeholder art is acceptable but must look intentional and recognizably like this car.
- **Active rear wing:** a large, deployable rear wing. Animate it pitching/raising as the car rises and flattening as it falls.
- **Thrusters:** four rear-mounted **SpaceX cold-gas thrusters**. On every flap, fire a short **downward/rearward blast of cold-gas** — a puff of white vapor/smoke and a brief blue-white glow — and kick up a small cloud. This is the in-fiction "wing flap."
- **Fan/suction system:** optional subtle underglow when skimming near the ground.
- **Pose:** bank nose-up on flap, pitch nose-down as it falls, matching Flappy Bird's angle-vs-velocity feel exactly.
- **Optional flair:** butterfly-door open/close shimmer on game start; Cybercab-style hubcaps on the wheels.

### Obstacles — Manhattan skyscrapers
- Instead of green pipes, use **New York skyscrapers**: a lower tower rising from the bottom, an upper tower hanging from the top, with the gap between them. The car flies *through the gap*.
- Vary facades, window grids, and rooftop details, but **keep the exact same collision geometry and gap size** as Flappy Bird so difficulty is unchanged.

### Background — New York
- Layered Manhattan skyline silhouettes with parallax, dusk/day gradient sky, drifting clouds, and a faint recognizable landmark (Empire State Building / Statue of Liberty). Parallax layers only — keep it cheap.
- Scrolling NYC street/roof level beneath, with Flappy Bird's ground-collision rules.

## Technical requirements
- **Single self-contained `index.html`** (inline CSS + JS; no build step, no external runtime deps). Assets as inline SVG/data-URIs so it works offline by double-click.
- **Canvas 2D** with `requestAnimationFrame` and **delta-time-based physics** (frame-rate independent across 60Hz/120Hz).
- **Responsive + retina:** size to viewport with `devicePixelRatio` scaling; handle portrait phone + desktop resize and orientation changes.
- **iOS Safari hardening:**
  - `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover, user-scalable=no, maximum-scale=1">`
  - Lock scroll/overscroll (`overflow: hidden`, `touch-action: none`, `position: fixed` with `100dvh` + `100vh` fallback).
  - Honor safe areas (`env(safe-area-inset-*)`) so the HUD clears the notch.
  - Prime Web Audio on the first user gesture so thrust/score/hit sounds play.
  - Pause cleanly on `visibilitychange`.
- **Audio:** lightweight synthesized WebAudio SFX (thruster whoosh for flap, chime for score, thud for hit) — no asset files. Corner mute toggle.
- **Performance:** pooled/reused obstacle objects, no per-frame allocations, no layout thrash; smooth on older iPhones.

## Polish (sensible defaults)
- Parallax city background, brief screen-shake + white-vapor flash on crash, semi-transparent overlays for ready/gameover.
- Touch-friendly restart button; respect `prefers-reduced-motion` where reasonable.
- Clean bold HUD using a system font stack.

## Deliverable & verification
- One `index.html` at project root, plus a short `README.md` (how to run: `open index.html`, or `python3 -m http.server` and open on phone via LAN).
- Verify before finishing: no console errors; tap and space both flap; score increments once per gap; collisions on both tower and ground; high score persists across reloads; no page zoom/scroll on iPhone.
- Do not ask follow-up questions — pick sensible defaults, implement fully, and note assumptions in the README.
