# apps/underpaint

Paint mixer, palette extractor, image-to-vector. See root `CLAUDE.md` for repo-wide conventions.

Dark "studio" theme (charcoal `--canvas`/`--surface` + gold `--accent`), Bitter+Karla fonts. Shared colour utils: `rgbToLab`/`labToRgb`, `kmeansLab`.

## Mixer
`paints` = {hex,parts,name?} list. Mixing is single-constant per-RGB-channel Kubelka-Munk (`reflectanceToKS`/`ksToReflectance`/`mixPigmentsKM` — same absorption/scattering theory as Mixbox, sized down since there's no real per-paint reflectance data, just arbitrary hex) instead of the old RYB-cube model; a `MEDIA` config + `applyMediumFinish(rgb, medium, dilution)` then applies a medium-specific finish on top of that shared pigment mix — oil (default) deepens/boosts chroma, gouache lightens+flattens (chalk fillers), watercolour blends toward paper-white by a `waterDilution` amount (0–0.9, its own slider, shown only for that medium) rather than mixing in white paint, since real watercolour lightens via transparency/dilution, not opaque tinting. `currentMedium`/`waterDilution` are set by the Medium button row (`.medium-btn`) and threaded through both `currentMix()` and `solveMixForTarget(hexes, targetHex, medium, dilution)`. `PALETTES` (3 presets) via `togglePalette()`. Match scored by `labDist`; `solveMixForTarget()` hill-climbs whole-number parts from random restarts to reach a target colour. `nearestColorName()` = Lab nearest-neighbour lookup. Note: literal fully-saturated digital primaries (e.g. `#0000FF`+`#FFFF00`) mix toward black rather than green under this model — real pigments never sit at 0/255 in a channel, so it only misbehaves on inputs no real paint tube produces; ordinary paint-like hexes (the built-in palettes, or anything picked to look like actual paint) mix correctly.

## Image→palette
Canvas → `kmeansLab()` → swatches by prevalence.

## Vector
k-means labels → `smoothLabels()` → `traceLabelBoundaries()` → `chaikin()`/`douglasPeucker()` → `buildSvgFromLabels()`, live+debounced (`scheduleVectorRender()`). Output size = source image dims (capped `VECTOR_OUTPUT_MAX_DIM`); "Detail" slider only affects analysis resolution. Legend embedded via `buildPaletteLegend()`, one row per used colour with its `nearestColorName()` (not hex) beside the swatch.
