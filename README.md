# 🌍 World Clock — Team Timezones

A beautiful, responsive world clock with seven visualizations:

- **Flip Clock** — retro split-flap display
- **Analog Wall** — clock faces with smooth sweeping second hands and comet trails
- **Globe** — rotating Earth with a real day/night terminator (seasonal tilt),
  atmosphere glow, and the Moon at its true position with the correct phase
- **Map** — NASA Earth at Night with a live day/night terminator (real
  sub-solar position, seasonal bow), sun & moon markers at the points where
  they are overhead, and cursor-anchored wheel-zoom / drag-pan
- **Horizon Strip** — 24-hour daylight bar with a glowing sun and moons
- **Orbit** — cities orbit Earth as planets, positioned by their local solar time
- **Solar System** — the real thing: planets at their true astronomical
  positions today (J2000 orbital elements), real moon periods (Triton runs
  retrograde), asteroid belt, comet, adjustable time-lapse, a faded background
  clock, hover facts, and wheel-zoom / drag-pan

## Live Demo

👉 [imhickey.github.io/world-clock](https://imhickey.github.io/world-clock/)

## Features

- Six world timezones (US West, US East, Dublin, Israel, India, Sydney)
- **Time travel** — pause, speed up (to 512×), reverse, step, and scrub the
  clock; every view follows the simulated time
- **Hover info cards** on any city: UTC offset, difference from your local
  time, approximate sunrise/sunset, and a 9–18h call-window suggestion
- Ambient starfield with twinkle and an hourly shooting star
- Day/night indicators on all views
- ⚙️ Settings panel — every effect has a toggle (with per-group all on/off);
  `?fx=off` / `?fx=all` URL overrides; respects `prefers-reduced-motion`
- ❓ Help overlay (press `H`) with the full keyboard reference
- Share card: press `S` to download a 1200×630 snapshot image
- Keyboard shortcuts: `1–7` views · `Space` pause · `+/−` speed · `R` reverse
  · `,`/`.` step · `0` live · `S` share · `F` fps · `H` help
- Fully responsive — works on desktop, tablet, and phone
- View + settings preferences saved to localStorage
- Single self-contained HTML file — no build step, no dependencies

## Credits

- **Earth at Night imagery:** NASA Earth Observatory / NOAA NGDC ([public domain](https://commons.wikimedia.org/wiki/File:The_earth_at_night.jpg))
- Canvas techniques inspired by the SkiaSharp solar-system demo
- Planetary elements: JPL/Standish approximate mean elements (J2000)
- Built by Ian Hickey

## License

[MIT](LICENSE)
