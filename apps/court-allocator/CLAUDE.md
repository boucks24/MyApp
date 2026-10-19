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

`assign()` is unchanged: a uniform random pick among sides that still have room.

## Advanced mode

Off by default, toggled by `#advancedToggle` on the setup screen and persisted
separately (`courtAllocator:advanced`, `loadAdvanced()`/`saveAdvanced()`). When
on, tapping the result card doesn't assign — `tapCard()` calls `askLevel()`,
which puts the card into a question ("?" / "Pick your level below") and shows
`#levelPicker`'s three `.level-btn`s. `chooseLevel(key)` then calls
`assign(key)`; `awaitingLevel` keeps a second tap on the card from re-asking.
With advanced off, `tapCard()` calls `assign(null)` and nothing changes.

`LEVELS` is `B`/`I`/`A` (name + short form for the court-map badge).
`courtLevel` maps court -> level key and is **claimed lazily**: no level owns a
court until the first player of that level lands on it, so an all-beginner night
still spreads over the whole pool. `sidesFor(level)` picks, in order: a free side
on a court this level already owns, else both sides of the lowest-numbered
unclaimed court, else (every court owned by someone else) any free side, with
the result sub saying they're sharing. `assign()` sets `courtLevel[court]` on
the claim and appends the level sentence. Both `courtLevel` and the picker are
cleared by `setupCourts()` and `reset()`.

`courtsNeeded()` opens at least `min(pool, 3)` courts in advanced mode, so the
three groups have somewhere of their own from the first tap.
