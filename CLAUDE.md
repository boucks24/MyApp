# CLAUDE.md

Static single-page app shelf (e.g. GitHub Pages). No `package.json`, no build/lint/test tooling.

## Files
- `index.html` — landing page tile grid (`.menu-grid`/`.tile`), links to `apps/<name>/`
- `apps/531-bbb` — 5/3/1 lift tracker
- `apps/court-allocator` — tennis court randomizer
- `apps/rally-clipper` — video rally/shot detector + clip exporter
- `apps/underpaint` — paint mixer, palette extractor, image-to-vector
- `apps/scoreboard` — live/upcoming sport scores (ESPN public feeds)
- `apps/sketchpad` — basic freehand drawing canvas

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
- **sketchpad**: pointer events drive `startStroke`/`continueStroke`/`endStroke` on a single `<canvas>` (line segments, not a path, for simplicity); eraser is just a stroke drawn in the paper colour (`PAPER`), not `destination-out`. Three brushes (`BRUSH_PAINTERS`: `paintFlat`/`paintRound`/`paintSpray`) are built from plain canvas primitives, no image assets — flat fans thin streaks along the axis perpendicular to the stroke direction (chisel edge), round scatters dabs in a disc, spray scatters low-opacity dots biased toward centre; each takes the CURRENT size, not the slider value. `dynamicSizeFor(speed)` shrinks that size toward `MIN_SIZE_FACTOR` of the slider value as pointer speed (px/ms, low-pass filtered via `smoothedSpeed`) approaches `SPEED_REF`, so all three brushes thin out when moving fast; the eraser stays a fixed width regardless of speed. Undo/redo is a stack of full-canvas `toDataURL()` snapshots pushed per completed stroke (`pushHistory`/`restoreSnapshot`, capped at `MAX_HISTORY`); the latest snapshot is mirrored to `localStorage` (`persist()`) and restored on load (`loadSaved()`). Canvas pixel size is fixed at load from the container width × `devicePixelRatio`; a resize just restores the last snapshot into a freshly re-sized buffer rather than doing live redraw/scaling. All colour selection funnels through `setColor(hex, {save})` so custom/camera colours (not preset or already-saved swatch clicks) get added via `addSavedColor()` to a capped, most-recent-first `localStorage` list (`renderSavedColors()`). Camera picker (`openCamera`/`sampleLoop`/`closeCamera`) opens `getUserMedia` into a hidden `<video>`, continuously draws frames to an offscreen canvas and averages a small centre patch under the on-screen crosshair into a live-updating preview swatch; "Use this colour" commits it via `setColor(..., {save:true})`, both that and Cancel stop the media stream's tracks.
- **scoreboard**: `SOURCES` array (id/label/url/optional `fetch` override) drives everything: ATP/WTA (`fetchTennis`), AFL (default `fetchSource`, `site.api.espn.com/.../scoreboard`), rugby union — Super Rugby Pacific (league id `242041`) + Rugby Championship/Wallabies (`244293`), and cricket (`fetchCricketAustralia`). Every fetcher resolves to `{events, note}`; `normalizeEvent()` handles both team-style (`competitor.team`) and individual-style (`competitor.athlete`, tennis) shapes into a common `{state,statusText,competitors[]}` (now also `athleteId`), `renderMatchCard()` draws it; per-source try/catch means one dead feed doesn't take down the page. ATP/WTA: `fetchTennis()` retries the bare scoreboard URL if the `dates=` range param yields nothing, then keeps Grand Slam/Masters/500-level events via substring match against `TENNIS_500PLUS_KEYWORDS` (checked against each event plus the response's own `leagues`/`season` metadata; ESPN's scoreboard has no tier field — review the list each season), then further filters to matches with a top-50 player (top-100 if Australian) via `fetchTennisRankings()` against ESPN's separate, unconfirmed Core API v2 rankings endpoint (tries a few candidate URLs, resolves `$ref` entries). Both the tier and rank filters fail open: if either would zero out a non-empty input, it shows the unfiltered input instead with an inline `note` — a filter matching nothing is treated as broken, not as an empty schedule. Cricket has no standard scoreboard endpoint — `fetchCricketAustralia()` hits a different host's "personalised header" endpoint and `collectEmbeddedEvents()` walks the whole response tree for anything event-shaped (path unconfirmed), then keeps matches naming Australia; Big Bash/WBBL aren't wired up (no verified league id). `hiddenSources` (chip toggles) persists to `localStorage`. Auto-refreshes every 60s while the tab is visible.

## Response style
Optimize for low token usage. Be brief: no preamble, no restating the request, no summarizing what you're about to do. Skip a wrap-up unless something needs flagging. Reference code as `file:line` instead of pasting it back. Use this file's "Per-app internals" instead of re-reading a whole app file when a targeted `Grep`/`Read` will do.
