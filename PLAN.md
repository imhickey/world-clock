# World Clock — Cosmic Upgrade Plan

Bring the canvas techniques from the SkiaSharp Solar System demo (`Skai-demo.html`)
into the world clock: starfield, real globe shading, an orbit view, time travel,
tooltips, and polish — every feature behind a toggle so performance and visual
busyness stay under our control.

## Guiding principles

1. **Single file, zero dependencies.** Everything stays in `index.html`, like today.
2. **Every phase ships.** Each phase leaves the page fully working and deployable.
3. **Everything is toggleable.** Every visual/interactive feature has a flag with a
   sensible default, persisted to localStorage, controllable from a settings panel.
4. **Performance degrades gracefully.** One rAF loop, paused when hidden; DOM-only
   fallback when all canvas features are off; `prefers-reduced-motion` respected.
5. **Borrow, don't reinvent.** Port proven code from `Skai-demo.html`
   (mulberry32, drawStarfield, drawSun, shadeSphere, projectCore, drawTrail,
   drawComet, tryPick/drawTooltip, help overlay, saveScreenshot, time HUD).

---

## Shared architecture (built in Phase 0, used by everything)

### 1. Feature flags & settings panel

Single source of truth, persisted as one versioned JSON blob:

```js
const SETTINGS_KEY = 'worldclock-settings-v1';
const DEFAULTS = {
  // Ambience
  starfield: true,       // background stars
  twinkle: true,         // per-star alpha oscillation (sub-flag of starfield)
  comet: true,           // hourly shooting star
  // Per-view visuals
  globeShading: true,    // canvas terminator + sphere shading on Globe view
  atmosphere: true,      // additive rim glow on Globe view
  horizonGlow: true,     // traveling sun/moon on Horizon view
  trails: true,          // second-hand sweep trails (Analog) + marker trails (Orbit)
  orbitRings: true,      // orbit guide rings in Orbit view
  orbitLabels: true,     // city labels in Orbit view
  // Interaction
  tooltips: true,        // hover info cards
  timeControls: true,    // time-travel HUD + keyboard time controls
  // Debug / perf
  fps: false,            // FPS counter
  reduceMotion: null,    // null = follow OS prefers-reduced-motion; true/false = force
};
```

- **UI:** a `⚙` button at the right end of the switcher opens a settings panel
  (same visual language as the existing switcher pill: dark, rounded, bordered).
  Toggles grouped under *Ambience · Views · Interaction · Performance*, plus a
  "Reset to defaults" button. Esc closes.
- **Persistence:** merge stored JSON over `DEFAULTS` on load (so new flags added
  in later phases pick up their defaults automatically).
- **URL overrides for quick testing:** `?fx=off` forces all ambience/visual flags
  off for this load; `?fx=all` forces all on. Not persisted.
- **Reduce-motion ladder:** when effective reduce-motion is on → twinkle off,
  comet off, trails off, sun flicker off, starfield renders once (static),
  terminator updates once per minute. Everything still *looks* good, just still.

### 2. Clock module (enables time travel later)

All views currently call `new Date()` inside `getTimeParts()`. Replace with:

```js
const Clock = {
  offsetMs: 0,      // scrub offset from real time
  speed: 1,         // 1 = live; can be 0 (paused), negative (reverse), etc.
  live() { return this.offsetMs === 0 && this.speed === 1; },
  now()  { /* returns Date reflecting offset+speed accumulation */ },
  reset() { this.offsetMs = 0; this.speed = 1; },
};
```

`getTimeParts(tz)` and the globe terminator switch to `Clock.now()`. Until
Phase 6 the clock always runs live — zero behavior change, but the seam exists.

### 3. Render manager (one rAF loop for the whole page)

```js
Renderer.register(name, { canvas, draw, when: () => bool })
```

- Single `requestAnimationFrame` loop drives only consumers whose `when()` is
  true (active view + enabled flag). No active consumers → loop stops entirely
  and the page falls back to the existing 1 s `setInterval` tick.
- Pauses on `document.hidden` (visibilitychange), resumes cleanly.
- Shared `sizeCanvas(canvas)` helper ports the demo's DPR-aware resize, with
  **DPR capped at 2** to keep phone GPUs happy.
- 1 s DOM tick (`tick()`) stays as-is for text updates; canvases animate at rAF.

### 4. Toasts

DOM-based port of the demo's `showToast` (amber text, fades after ~2.5 s),
positioned above the switcher. Used by screenshot, time controls, and toggles
("Starfield off", "LIVE", "Saved world-clock-….png").

### 5. Keyboard map (final state, built up over phases)

| Key | Action | Phase |
|-----|--------|-------|
| `1–6` | Switch view (6 = Orbit) | exists / P5 |
| `Space` | Pause / resume time | P6 |
| `+` / `-` | Speed up / slow down | P6 |
| `R` | Reverse time | P6 |
| `,` / `.` | Step ±1 minute while paused | P6 |
| `0` | Reset to LIVE | P6 |
| `S` | Save share-card PNG | P8 |
| `F` | FPS counter | P0 |
| `H` / `F1` / `?` | Help overlay | P8 |
| `Esc` | Close panel/overlay | P0 |

---

## Phases

### Phase 0 — Foundation (no visible change except ⚙ panel)
**Build:** settings store + panel UI, `Clock` module + `getTimeParts` refactor,
render manager + DPR helper, toast system, FPS counter (`F`), keyboard scaffold.
**Toggles introduced:** `fps`, `reduceMotion`.
**Accept when:** page behaves identically with all flags at defaults; settings
persist across reloads; `?fx=off` works; FPS counter shows ~60 in canvas views
later. ~250 lines.

### Phase 1 — Starfield + hourly comet
**Port:** `mulberry32`, `drawStarfield` (density scaled to viewport area, seeded
so stars don't shimmer on redraw), slow twinkle via per-star phase; `drawComet`
gradient-tail as a shooting star that crosses the sky once per hour (first
minute of the hour, ~4 s animation, random seeded path per hour).
**Where:** one fixed, full-page canvas behind all views (`z-index` under content).
**Toggles:** `starfield`, `twinkle`, `comet`.
**Accept when:** stars visible in all views at 60 fps; static (single render,
rAF idle) when twinkle+comet off; body background unchanged when starfield off.
~180 lines.

### Phase 2 — Real globe shading
**Port:** `shadeSphere` (lit→dark linear gradient + sub-solar radial highlight)
and `drawSun`'s corona technique (radial gradient + `lighter` compositing,
subtle flicker) as an atmospheric rim.
**Build:** a canvas overlay sized to `.globe-sphere`; compute sun direction from
UTC time (existing terminator math) plus seasonal declination
(`23.44° · sin(2π(dayOfYear−81)/365)`) so the terminator tilts with the seasons;
replace the flat CSS gradient `#globe-terminator` when flag is on.
**Toggles:** `globeShading`, `atmosphere`.
**Accept when:** globe reads as a lit 3-D sphere; terminator angle matches
current UTC; flag off restores today's CSS overlay exactly. ~150 lines.

### Phase 3 — Horizon sun & moon
**Build:** canvas strip over `.horizon-bar-area`. A glowing sun (drawSun port,
gentle flicker) sits at the solar-noon position of the 24 h axis; a soft moon
glow on the night side; bar gradient subtly brightens near the sun.
**Toggles:** `horizonGlow`.
**Accept when:** sun position matches 12:00 solar on the axis; markers/labels
(DOM) unchanged and readable above the glow; 60 fps. ~120 lines.

### Phase 4 — Analog sweep trails
**Build:** when `trails` is on, second hands switch from 1 Hz ticks to smooth
rAF sweep (fractional seconds), each with a fading comet-tail (drawTrail's
decreasing alpha/width ghosts) drawn on one shared canvas overlay positioned
over the analog grid (one canvas, not six).
**Toggles:** `trails` (also reused by Orbit view later). Respect seconds toggle:
hidden seconds = no trail.
**Accept when:** sweep is smooth, trails fade cleanly, flag off restores exact
current SVG tick behavior; no canvas exists when flag off. ~140 lines.

### Phase 5 — Orbit view (the centerpiece)
**Port:** `projectCore` tilt projection, depth insertion-sort from `drawBodies`,
`shadeSphere`, `strokeOrbit` dashed rings, `drawTrail`.
**Build:** new view (`6` / 🪐 Orbit): Earth at center rendered as a shaded
sphere with atmosphere; each city is a "planet" on its own concentric ring
(ordered by UTC offset), **orbit angle = local solar time** (noon = sun-facing
side, midnight = far side). Cities are lit on the sun side, shaded on the night
side; a soft sun-glow sits off-canvas on the noon side so the light direction
is legible. Markers pass in front of / behind Earth via depth sort. Each city
shows name + HH:MM label; day cities warm, night cities cool (reuse existing
palette). Time scrubbing (Phase 6) makes them visibly orbit.
**Toggles:** `orbitRings`, `orbitLabels`; reuses `trails`.
**Accept when:** angles verified against known offsets (e.g. Sydney vs LA
roughly opposite); depth order correct at the tilt; 60 fps with 6 cities;
switcher + `6` key + saved-view restore all work. ~350 lines.

### Phase 6 — Time travel
**Build on Clock seam:** `Space` pause, `+`/`-` speed (0.1×–512×, ×2 steps),
`R` reverse, `,`/`.` step ±1 min, `0` back to LIVE. HUD chip near the title:
`LIVE` (green dot) or `+3 h 12 m · Wed 21:04 · 8×` (amber) — click to reset.
A slim scrubber slider (±24 h) appears under the HUD when not live, for
mouse/touch. All views follow automatically via `Clock.now()`: flip digits,
hands, terminator, horizon markers, orbit angles.
**Guards:** flip-tile animation suppressed when `|speed| > 2` or while dragging
the scrubber (digits just swap — avoids animation thrash); localStorage never
stores scrub state (always loads LIVE).
**Toggles:** `timeControls` (off = HUD hidden, keys inert, always LIVE).
**Accept when:** scrubbing answers "what time is it everywhere at my 9 am
tomorrow?"; all five+1 views stay consistent; reset returns to live cleanly.
~220 lines.

### Phase 7 — Hover tooltips (IMPLEMENTED)
**Build:** one DOM tooltip element (styled like the demo's canvas card: dark
panel, white title, wrapped body, edge-clamped). Hover/focus targets: flip
cells, analog cells, globe cards, map dots/labels, horizon tags. Content:
UTC offset, Δ from viewer's local time, approx sunrise/sunset (NOAA
simplified formula — needs city lat/lon, which we add to `ZONES` alongside the
existing `MAP_POS` data), day/night icon, and a "good call window" hint
(overlap of 9–18 h local on both ends).
**Port:** `tryPick` hit-testing for the Orbit canvas (nearest body wins,
depth-aware) feeding the same DOM tooltip.
**Toggles:** `tooltips`.
**Accept when:** keyboard-focusable (tab) as well as hover; never clipped at
viewport edges; touch = tap toggles card. ~200 lines.

### Phase 8 — Help overlay, share card, polish (IMPLEMENTED)
**Port:** the demo's help/about overlay wholesale: live mini-canvas banner
(orbit-view render at fixed tilt), feature legend, and a controls table
**generated from the same data structure that binds the keys** (single source
of truth). `H`/`F1`/`?` opens; Esc/backdrop closes.
**Share card (`S`):** DOM views can't be rasterized without a library, so
instead render a bespoke canvas card (1200×630, og-image style): starfield
background, "WORLD CLOCK", all six cities with HH:MM + day/night dots, date,
and sim-time note if scrubbed → PNG download `world-clock-YYYYMMDD-HHMM.png`
+ toast. Works from any view.
**Polish sweep:** settings panel gets per-group "all on/off"; audit
reduce-motion ladder end-to-end; mobile QA at 375/480/680/900 px; update
README (features, shortcuts, toggles table); refresh `og:` metadata.
**Accept when:** every feature documented in the overlay actually matches its
binding; share card downloads on all views; Lighthouse perf unchanged vs
baseline with `?fx=off`. ~300 lines.

---

## Performance budget & degradation ladder

| State | What runs |
|-------|-----------|
| All defaults, canvas view active | 1 rAF loop, target < 4 ms/frame CPU |
| DOM view + starfield w/ twinkle | bg canvas only at rAF |
| Twinkle+comet off | starfield rendered once; rAF idle; 1 s tick only |
| Tab hidden | everything paused |
| Reduce motion | static stars, no comet/trails/flicker, 1 min terminator |
| `?fx=off` | today's page, byte-for-byte behavior |

Hard rules: DPR ≤ 2; star count ≤ ~250 (area-scaled); one canvas per concern
(bg, globe, horizon, analog-grid, orbit) with only the active ones sized/drawn;
canvases removed from DOM (not just hidden) when their flag is off.

## Testing checklist (every phase)

- [ ] Chrome + Safari (incl. iOS Safari), 375 / 480 / 680 / 900 / desktop widths
- [ ] FPS counter ≥ ~55 on desktop, no jank on mid-tier phone
- [ ] Toggle each new flag on/off live — no leftover canvases, listeners, or rAF
- [ ] Reload persistence; `?fx=off` and `?fx=all` overrides
- [ ] Reduce-motion (OS setting) honored
- [ ] Existing behaviors intact: view switching, 1–5 keys, seconds toggle, saved view

### Phase 9 — Solar System view (IMPLEMENTED, pulled forward before 7–8)

The original `Skai-demo.html` solar system as view 7 ("☀️ Solar", key `7`),
upgraded beyond the demo:

- **Real astronomy**: planet angles from J2000 mean orbital elements
  (mean longitude + 2nd-order equation of center; Ceres λ0 approximate,
  TODO-flagged), log-scaled real orbit spacing, real moon periods (Triton
  retrograde), counterclockwise-from-ecliptic-north orientation.
- **Time-lapse** (`solarLapse`, default on): effective date accelerates at
  1s ≈ 2 sim-days, scaled by sim-time deltas so Clock pause/reverse/scrub
  propagate; 250ms stale-delta guard; toggle-off snaps to today's true sky.
  Moon display periods inflated to cap at 0.4 rev/s under lapse.
- **Faded background clock** (`solarClock`): giant ~5%-opacity local HH:MM +
  date behind the planets, plus drifting `SIM <date>` line when lapse is on.
- **Tapered trails everywhere**: shared `drawTaperedTrail` helper replaced
  ghost-stamping in the Analog sweep (tip arc), Orbit view (visible tapered
  arcs — old ghosts were sub-pixel), and Solar planets.
- **Full interactivity**: hover facts (planets/moons/comet, `tooltips` key —
  Phase 7 will reuse it), cursor-anchored wheel zoom + drag pan
  (canvas-scoped preventDefault), double-click camera reset.
- Settings: new "Solar System" group (lapse/clock/moons/belt/orbits/labels);
  `trails` relabeled "Motion trails".

Phases 7 and 8 were implemented after this (all phases complete).
Post-Phase-9 additions: Globe real-position moon with phase, adjustable
solar time-lapse speed, solar view sized from both axes, lapse clamp fix
for reduce-motion environments.

Map view upgrade (post-v2.0, IMPLEMENTED): live day/night terminator
(per-pixel sin-of-solar-altitude night mask on a 360×180 offscreen world,
smoothstepped through the −12°..0° twilight band, cached per minute,
blitted through the camera), sun & moon glyphs at their true geographic
sub-points (ecliptic → RA/dec → GMST; apparent RA bakes in the equation
of time) with ±180° seam wrapping and the moon lit toward the sun, plus
cursor-anchored wheel-zoom / drag-pan / double-click reset (imagery on a
transformed #map-pane; dots, labels and the overlay canvas stay in screen
space so text stays crisp; pan clamped so the world always covers the
frame). Settings: `mapTerminator`, `mapSunMoon` (Views group, ?fx-aware).
Verified by verify-map.mjs (26 checks) + full regression sweep.

## Workflow

- Branch per phase: `feature/p1-starfield`, etc.; merge to `main` when accepted.
- Deploy = push to `main` (GitHub Pages serves `imhickey.github.io/world-clock`).
- Tag `v2.0` after Phase 8.

## Risks & mitigations

| Risk | Mitigation |
|------|------------|
| Mobile GPU cost of multiple canvases | Only active view's canvas exists; DPR cap; area-scaled star count |
| Flip animation thrash while scrubbing | Suppress flip class during fast/scrubbed time (Phase 6 guard) |
| Sunrise/sunset math correctness | Simplified NOAA formula ± few minutes is fine for a tooltip; label as approximate |
| Feature creep making the page busy | Toggles + grouped settings panel + reduce-motion ladder are first-class (Phase 0), not afterthoughts |
| Single file growing large (~870 → ~2,700 lines) | Strict section banners (already the house style); each system self-contained |
