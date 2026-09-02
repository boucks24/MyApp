# CLAUDE.md

Static single-page app shelf (e.g. GitHub Pages). No `package.json`, no build/lint/test tooling.

## Files
- `index.html` — landing page tile grid (`.menu-grid`/`.tile`), links to `apps/<name>/`
- `apps/531-bbb` — 5/3/1 lift tracker
- `apps/court-allocator` — tennis court randomizer
- `apps/rally-clipper` — video rally/shot detector + clip exporter
- `apps/underpaint` — paint mixer, palette extractor, image-to-vector
- `apps/scoreboard` — live/upcoming sport scores (ESPN public feeds)

Each app is **one self-contained `index.html`** (inline `<style>`+`<script>`, no external JS/npm). Only external resources: Google Fonts `<link>`s; `rally-clipper` also lazy-loads ffmpeg.wasm + an optional ML model; `scoreboard` fetches live JSON directly from ESPN's public (unofficial, keyless) scoreboard endpoints — the only app that talks to a live external API rather than working purely on local/user-supplied data.

## Git
Personal site, no PRs/review — commit and push straight to `main`.

## Verify before commit
```
python3 -m http.server 8080   # preview at /apps/<app>/
node -e "new Function(require('fs').readFileSync('apps/<app>/index.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1])"   # JS parses
```
No test suite. For interactive changes, load in a real browser (Playwright Chromium if available) and exercise the feature.

## Conventions
- New app: CSS vars in `:root`, `.back-link` to `../../`, one `<script>` at end of `<body>`
- Colour theme + Google Font are per-app, deliberately not shared
- No server persistence — `localStorage` only; file inputs (photo/video) processed client-side only

## Per-app internals
- **531-bbb**: `route` var + `render()`/`go()` swap views (Setup/Home/Train/History/Progress). State (`settings`/`state`/`history`) in `localStorage` via `load()`/`save()`. Plate math: `plateBreakdown`, `diffPlateCounts`, `buildSessionPlateStages`. `history.json` is a static export, unused at runtime.
- **court-allocator**: `courtsNeeded`/`setupCourts`/`assign`, no persistence.
- **rally-clipper**: `analyzeMotion()` (canvas diff) + `analyzeAudio()` (Web Audio) run in parallel, optional ML via `loadScript`/`getMlModels`/`analyzeML`. `buildSegments()` fuses signals → review via `renderTimeline`/`renderShotList` → `getFfmpeg()` cuts/exports, all client-side.
- **underpaint**: shared colour utils (`rgbToLab`/`labToRgb`, RYB model: `RYB_CORNERS`/`buildRybLut`/`rgbToRyb01`/`rybToRgb01`, `kmeansLab`).
  - Mixer: `paints` = {hex,parts,name?} list, `mixPaintsRgb()` blends in RYB space (parts are whole numbers, 0 allowed). `PALETTES` (3 presets) via `togglePalette()`. Match scored by `labDist`; `solveMixForTarget()` hill-climbs whole-number parts from random restarts to reach a target colour. `nearestColorName()` = Lab nearest-neighbour lookup.
  - Image→palette: canvas → `kmeansLab()` → swatches by prevalence.
  - Vector: k-means labels → `smoothLabels()` → `traceLabelBoundaries()` → `chaikin()`/`douglasPeucker()` → `buildSvgFromLabels()`, live+debounced (`scheduleVectorRender()`). Output size = source image dims (capped `VECTOR_OUTPUT_MAX_DIM`); "Detail" slider only affects analysis resolution. Legend embedded via `buildPaletteLegend()`.
- **scoreboard**: `SOURCES` array (id/label/url/optional `accept` filter) drives everything — each maps to an ESPN `site.api.espn.com/apis/site/v2/sports/<sport>/<league>/scoreboard` feed (ATP, WTA, AFL, NRL, A-League M/W). `fetchSource()` pulls + filters raw events, `normalizeEvent()` handles both team-style (`competitor.team`) and individual-style (`competitor.athlete`, tennis) shapes into a common `{state,statusText,competitors[]}`, `renderMatchCard()` draws it; per-source try/catch means one dead feed doesn't take down the page. ATP/WTA are filtered to 500+/Masters/Slam level via substring match against `TENNIS_500PLUS_KEYWORDS` (ESPN doesn't expose a tier field) — review that list each season. `hiddenSources` (chip toggles) persists to `localStorage`. Auto-refreshes every 60s while the tab is visible. Cricket, rugby union and other Australian representative fixtures aren't wired up — no verified ESPN endpoint code for them yet.

## Response style
Optimize for low token usage. Be brief: no preamble, no restating the request, no summarizing what you're about to do. Skip a wrap-up unless something needs flagging. Reference code as `file:line` instead of pasting it back. Use this file's "Per-app internals" instead of re-reading a whole app file when a targeted `Grep`/`Read` will do.
