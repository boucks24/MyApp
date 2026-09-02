# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A static "app shelf" — a personal collection of small single-page tools, meant to be served as-is (e.g. GitHub Pages) with no build step. There is no `package.json`, no bundler, no test runner, and no linter configured anywhere in the repo.

```
index.html                        landing page: dark-themed tile grid linking to each app
apps/531-bbb/index.html           5/3/1 Boring-But-Big lift tracker
apps/531-bbb/history.json         a manually exported data snapshot from that app (not read at runtime)
apps/court-allocator/index.html   randomly assigns players to tennis courts
apps/rally-clipper/index.html     detects rallies/shots in an uploaded tennis video, exports the clips
apps/underpaint/index.html        paint colour mixing, palette extraction, image-to-vector simplifier
```

Every app is a **single self-contained `index.html`** with inline `<style>` and inline `<script>` — no external JS files, no npm dependencies. The only external resources are Google Fonts `<link>` tags and, in `rally-clipper`, dynamically loaded scripts for ffmpeg.wasm and an optional ML model.

## Git workflow

This is a personal, single-maintainer site served straight from `main` (e.g. via GitHub Pages) — there's no PR/review process. Commit and push changes directly to `main`, not a feature branch.

## Commands

There is no build/lint/test tooling. To preview changes locally, serve the repo root with any static file server and open the relevant path, e.g.:

```
python3 -m http.server 8080
# then visit http://localhost:8080/ or http://localhost:8080/apps/<app>/
```

Since each app is one HTML file, verify JS changes parse before committing:

```
node -e "
const fs = require('fs');
const html = fs.readFileSync('apps/<app>/index.html','utf8');
const m = html.match(/<script>([\s\S]*)<\/script>/);
new Function(m[1]);
console.log('JS parses OK');
"
```

For anything interactive, the most reliable check is loading the page in a real browser (Playwright's Chromium works if available) and exercising the feature — there is no automated test suite to fall back on.

## Cross-app conventions

- New links from the root shelf go in `index.html`'s `.menu-grid`, following the existing `.tile` pattern (each links to `apps/<name>/`).
- New apps should follow the established single-file structure: CSS variables in `:root` for the palette, a `.back-link` back to `../../`, and all logic in one `<script>` block at the end of `<body>`.
- Each app has its own colour theme (CSS custom properties) and font pairing (Google Fonts) — these are intentionally different per app, not shared.
- No app persists data to a server; anything saved (e.g. 531-bbb's settings/history) uses `localStorage` only, and anything computed from user files (photos, videos) happens entirely client-side.

## Per-app architecture notes

**apps/531-bbb** — Hand-rolled SPA with no router library: a `route` variable plus `render()`/`go()` swap between Setup/Home/Train/History/Progress views. All state (`settings`, `state`, `history`) lives in `localStorage` via `load()`/`save()`. Barbell plate math (`plateBreakdown`, `diffPlateCounts`, `buildSessionPlateStages`) computes what plates to add/remove between sets, not just the total weight. `history.json` is a static export artifact, not consumed by the app.

**apps/court-allocator** — Pure client-side randomizer (`courtsNeeded`, `setupCourts`, `assign`), no persistence.

**apps/rally-clipper** — Multi-signal pipeline: `analyzeMotion()` (canvas frame-diffing via `requestVideoFrameCallback`) and `analyzeAudio()` (Web Audio) run in parallel, optionally combined with an ML model loaded dynamically (`getMlModels`/`analyzeML`) via `loadScript`. `buildSegments()` fuses these signals into shot segments, which the user can review (`renderTimeline`/`renderShotList`) before `getFfmpeg()` loads ffmpeg.wasm to cut and export the selected clips — entirely in-browser, nothing uploaded.

**apps/underpaint** — Three tabs sharing one set of colour-math utilities at the top of the script (`rgbToLab`/`labToRgb`, an RYB paint-mixing model built from `RYB_CORNERS`/`buildRybLut()`/`rgbToRyb01`/`rybToRgb01`, and `kmeansLab()` for palette extraction):
  - **Colour mixer** — `paints` is the in-memory list of {hex, parts, name?} being mixed; `mixPaintsRgb()` does the RYB-space blend. `PALETTES` holds three named reference palettes (Primary Triad, Zorn, Earth & Landscape) toggleable via `togglePalette()`. A target colour (practice vs. "solve" mode) is compared against the current mix via Lab distance (`labDist`); `solveMixForTarget()` does a randomized/local-search reverse-solve to find a parts ratio that best reaches a target colour. `nearestColorName()` maps an arbitrary hex to the closest name in the `COLOR_NAMES` table (Lab-space nearest neighbour) for a friendlier display than raw hex.
  - **Image to palette** — draws the uploaded image to a canvas, runs `kmeansLab()` over its pixels, and lists the resulting swatches by prevalence.
  - **Simplify to vector** — same k-means-in-Lab approach, then `smoothLabels()` denoises the label map, `traceLabelBoundaries()` walks region edges, `chaikin()`/`douglasPeucker()` smooth and simplify the traced paths, and `buildSvgFromLabels()` emits the final SVG (rendered live, debounced, as the controls change — see `scheduleVectorRender()`). Output is always sized from the source image's own dimensions (capped by `VECTOR_OUTPUT_MAX_DIM`), independent of the "Detail" slider, which only controls internal analysis resolution. A colour-swatch legend is appended into the SVG itself via `buildPaletteLegend()` so it travels with SVG/PNG exports.
