# apps/scoreboard

Live/upcoming sport scores (ESPN public feeds). See root `CLAUDE.md` for repo-wide conventions.

`SOURCES` array (id/label/url/optional `fetch` override) drives everything: ATP/WTA (`fetchTennis`), AFL (default `fetchSource`, `site.api.espn.com/.../scoreboard`), rugby union — Super Rugby Pacific (league id `242041`) + Rugby Championship/Wallabies (`244293`), and cricket (`fetchCricketAustralia`). Every fetcher resolves to `{events, note}`; `normalizeEvent()` handles both team-style (`competitor.team`) and individual-style (`competitor.athlete`, tennis) shapes into a common `{state,statusText,competitors[]}` (now also `athleteId`), `renderMatchCard()` draws it; per-source try/catch means one dead feed doesn't take down the page.

ATP/WTA: `fetchTennis()` retries the bare scoreboard URL if the `dates=` range param yields nothing, then keeps Grand Slam/Masters/500-level events via substring match against `TENNIS_500PLUS_KEYWORDS` (checked against each event plus the response's own `leagues`/`season` metadata; ESPN's scoreboard has no tier field — review the list each season), then further filters to matches with a top-50 player (top-100 if Australian) via `fetchTennisRankings()` against ESPN's separate, unconfirmed Core API v2 rankings endpoint (tries a few candidate URLs, resolves `$ref` entries). Both the tier and rank filters fail open: if either would zero out a non-empty input, it shows the unfiltered input instead with an inline `note` — a filter matching nothing is treated as broken, not as an empty schedule.

Cricket has no standard scoreboard endpoint — `fetchCricketAustralia()` hits a different host's "personalised header" endpoint and `collectEmbeddedEvents()` walks the whole response tree for anything event-shaped (path unconfirmed), then keeps matches naming Australia; Big Bash/WBBL aren't wired up (no verified league id).

`hiddenSources` (chip toggles) persists to `localStorage`. Auto-refreshes every 60s while the tab is visible.
