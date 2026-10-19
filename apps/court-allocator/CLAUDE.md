# apps/court-allocator

Tennis court randomizer. See root `CLAUDE.md` for repo-wide conventions.

`AVAILABLE_COURTS` is 1-18 (everything the club can give us); the setup screen's
eighteen `.court-chip` toggles pick which of them are in play. The picked pool lives in
`selectedCourts` and is the one piece of state persisted (`localStorage`,
`courtAllocator:courts`, via `loadCourts()`/`saveCourts()`) — which courts are free
tends to hold for weeks, so re-picking every session is busywork. `loadCourts()`
filters stored values against `AVAILABLE_COURTS` and falls back to `[13,14,15,16]`
(the set that used to be hard-coded) on empty, blocked or corrupt storage.

`courtsNeeded(count)` is `ceil(count / PER_COURT)` clamped to the pool size, and
`setupCourts()` takes that many from the **front** of the pool (lowest numbers
first), so a short night fills from one end rather than an arbitrary order. When
the player count exceeds the pool's capacity the header says how many will be
waiting rather than silently under-allocating; an empty pool blocks `startBtn`
with an inline message. `renderPicker()` is re-run by `backToSetup()` so the chips
reflect the stored pool when you come back. The picker is a fixed
`repeat(6,minmax(0,1fr))` grid (18 courts = three full rows) so the chips always
read as the court numbers in order instead of reflowing into a ragged last row
(six per row down to 320px).

`assign(level)` is a uniform random pick among the sides `sidesFor(level)` offers;
with no level (Open mode) that's every side that still has room. `assign()` owns
the "nowhere to put this player" decision — the games court if there is one,
otherwise the all-full message. `chooseLevel()` used to make that call itself and
returned early with "All courts are full", which meant the games court was never
reached.

## Pairing modes

`mode` is `'off' | 'sbmm' | 'mix'`, persisted at `courtAllocator:mode`.
`loadMode()` migrates the old `courtAllocator:advanced` boolean (true -> `sbmm`).
A three-way `.mode-row` replaced the on/off `.switch` this used to be.

`MODES`/`MODE_NOTE` must stay declared **above** `let mode = loadMode()`.
They were below it, so `loadMode()` hit a temporal dead zone on `MODES`, and its
own `try/catch` — there for blocked localStorage — quietly turned that
`ReferenceError` into `'off'`. Both the saved mode and the legacy migration
silently did nothing, and nothing in the UI looked broken.

- **Open**: any player, any free side.
- **SBMM**: lands you with your own level. `courtLevel` maps court -> level key,
  **claimed lazily** — no level owns a court until the first player of that level
  lands on it, so an all-beginner night still spreads over the whole pool.
  `courtsNeeded()` opens at least `min(pool, 3)` courts in this mode so the three
  groups have somewhere of their own from the first tap.
- **Mixed**: your *partner* is the other end of the ladder (`PARTNER_OF`:
  B<->A, I with I), so each pair is about even and a beginner spends the night
  alongside a good player rather than against a wall of them. `sideLevels`
  (court+side -> the levels already standing there) is what makes this possible;
  `slots` only counts heads. It prefers a side holding one player of the
  complementary level, then an empty side (so partners are not doubled up by
  accident), then anything free.

**Level is a preference, never a gate.** Every branch of `sidesFor()` ends at
`freeSides(COURTS)`: a player who cannot have the pairing they asked for still
gets a spot. An uneven pair, or a court running as singles, beats sitting out.

## The games court

When the picked pool cannot seat everyone, `pickGamesCourt()` reserves one court
— 16 first, then 4 — as the games court: no sides, no cap, everyone who overflows
plays there together. It is only reserved if a normal court is left over, since
turning the only court into the games court helps nobody. The reserved court is
filtered out of `COURTS`, so normal play fills the rest first and `assign()`
falls through to `assignToGames()` once every side is full. While a games court
exists the night is never "full": `renderGrid()` skips the all-full state, so the
card and the level buttons never go dead.

## The result card

In SBMM and Mixed the three `.level-btn`s are shown from the moment the play
area opens, and the big line rests on an en dash rather than "TAP" — the buttons
are the thing to press, so a giant TAP would point at nothing. They used to
appear only after a tap, which took the card from 182px to 266px and back on
every single assignment; leaving them up for the whole mode is what keeps its
height fixed. `.result-sub` reserves two lines for the same reason, since the
longest result sentence wraps on a narrow phone and the shortest prompt does
not. `applyMode()` owns all of this, and in a levelled mode it also strips the
card body's `role`/`tabindex` — with the buttons always present the body has
nothing left to do. In Open mode the card is still the button and still says
TAP.

`#levelPicker` sits **inside** `.result-card`, under a hairline, so the question
and its three answers are one object. That is why the card is no longer the
button it used to be: a `role="button"` with buttons inside it is invalid, and a
click on a level would bubble to the card's own handler and re-open the question
it had just answered. The tap target is now an inner
`#resultTap`, with the picker as its sibling rather than its descendant, which
removes the problem structurally instead of patching it with
`stopPropagation()`. `.disabled`/`aria-disabled` for the all-full state moved
onto `#resultTap` with it; `.result-card` keeps only the box, the hover tint and
the `::before` court lines. The buttons carry their own translucent fill because
a flat `var(--court)` is the card's own colour and vanished against it.

Each `.level-btn` is a big letter (`.lv-key`) over a small word (`.lv-word`),
not one label at a tap-me size. Setting the whole label large meant
"Intermediate" could not fit three even columns on a narrow phone without
either shrinking every label or breaking the word across lines; the letter
carries the size instead and the word stays small underneath. The grid is
`repeat(3,minmax(0,1fr))` rather than `1fr 1fr 1fr`, because a plain `1fr`
will not shrink a column below its min-content and that one unbreakable word
made the three buttons 89/122/96 wide. The word's size is clamped down to
8.5px purely so "INTERMEDIATE" still fits at 320px.

`LEVELS` is `B`/`I`/`A` (name + short form for the court-map badge). In SBMM
`sidesFor()` picks, in order: a free side on a court this level already owns,
else both sides of the lowest-numbered unclaimed court, else any free side at
all, with the result sub saying they're sharing. `courtLevel`, `sideLevels` and
the games-court tally are all cleared by `setupCourts()` and `reset()`.
