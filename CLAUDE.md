# CLAUDE.md

Static single-page app shelf (e.g. GitHub Pages). No `package.json`, no build/lint/test tooling.

## Files
- `index.html` — landing page: a full-screen vertical scroll deck, one panel per app (`.panel`/`.art`), ordered by last used and wrapping around at both ends. Panels are built at parse time from the `APPS` array; each app's art is a set of plain `<i>` boxes styled by an `art-*` class, with its shape count in `SHAPES`.
- `apps/531-bbb` — 5/3/1 lift tracker
- `apps/court-allocator` — tennis court randomizer
- `apps/rally-clipper` — tennis analysis: auto rally/shot detector + clip exporter, video↔.fit sync, .fit route/wellness analysis (formerly the standalone Route Mapper app, merged in), and a pose-similarity photo montage builder
- `apps/underpaint` — paint mixer, palette extractor, image-to-vector
- `apps/scoreboard` — live/upcoming sport scores (ESPN public feeds)
- `apps/sketchpad` — basic freehand drawing canvas
- `apps/swatch-finder` — look up an Ohuhu Honolulu marker by code/name; shows its colour and location on the source swatch-sheet photo
- `apps/motion-replay` — rebuild the pose in a video clip as a skinned 3D figure on a court (MediaPipe Pose + three.js); the capsule rig it started as is now only the fallback when the model will not load

Each app folder has its own `apps/<name>/CLAUDE.md` with that app's implementation internals — read only the one for the app you're touching, not all of them. This file covers only repo-wide conventions.

Each app defaults to **one self-contained `index.html`** (inline `<style>`+`<script>`, no external JS/npm) — simplest thing that works for a small app. That's a default, not a hard rule: this is plain static hosting with no build step, so an app is free to split across multiple files (separate data file, separate binary assets) whenever that's the better structure — weigh it primarily by what's most token-efficient for Claude Code to read and edit, not by a one-file dogma. Concretely: don't inline large binary/base64 payloads (photos, big generated datasets) into the HTML — put them in their own files (real `.jpg`s, a `.json`/`.js` data file) and reference them normally. A browser loads an external file exactly as fast as an inline data URI, but an inline copy means every future edit to the app's actual logic first has to load megabytes of irrelevant base64 as text tokens. Keep the app's logic (HTML/CSS/JS) as the one file people think of as "the app"; split off only the heavy, rarely-edited data/assets alongside it in the same `apps/<name>/` folder. `swatch-finder` is the concrete example: its 4 reference photos are plain `.jpg` files and its 320-swatch dataset is a separate `.json`, both sitting next to `index.html` in `apps/swatch-finder/`, not base64-inlined.

Only external resources: Google Fonts `<link>`s; `rally-clipper` also lazy-loads an optional ML model (TensorFlow.js, from unpkg) and draws map raster tiles fetched live from keyless, attributed Esri endpoints (World Dark Gray, World Imagery) — its video export engine (Mediabunny, for WebCodecs-based MP4 demux/mux) is vendored locally as `apps/rally-clipper/mediabunny.min.js` rather than CDN-loaded, so export itself needs no network access; `scoreboard` fetches live JSON directly from ESPN's public (unofficial, keyless) scoreboard endpoints; `motion-replay` dynamically imports three.js and `@mediapipe/tasks-vision` from jsdelivr and fetches a Pose Landmarker `.task` model from `storage.googleapis.com` on first run (its video never leaves the device, but the reconstruction step needs a connection) — these are the only apps that talk to a live external service rather than working purely on local/user-supplied data.

## Git
Personal site, no PRs/review. Never use branches — always commit and push straight to `main`.

## Model
Use `opusplan` for this repo.

## Verify before commit
```
python3 -m http.server 8080   # preview at /apps/<app>/
node -e "new Function(require('fs').readFileSync('apps/<app>/index.html','utf8').match(/<script>([\s\S]*)<\/script>/)[1])"   # JS parses
```
No test suite. For interactive changes, load in a real browser (Playwright Chromium if available) and exercise the feature.

## Conventions
- New app: CSS vars in `:root`, `.back-link` to `../../`, one `<script>` at end of `<body>`
- Colour theme + Google Font are per-app, deliberately not shared
- No server persistence — `localStorage` (or IndexedDB for anything a string can't hold, e.g. rally-clipper's file handles) only; file inputs (photo/video) processed client-side only

## Response style
Optimize for low token usage. Be brief: no preamble, no restating the request, no summarizing what you're about to do. Skip a wrap-up unless something needs flagging. Reference code as `file:line` instead of pasting it back. Use the relevant `apps/<name>/CLAUDE.md` instead of re-reading a whole app file when a targeted `Grep`/`Read` will do.
