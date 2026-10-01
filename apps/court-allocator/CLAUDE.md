# apps/court-allocator

Tennis court randomizer. See root `CLAUDE.md` for repo-wide conventions.

`AVAILABLE_COURTS` is 13-18 (everything the club can give us); the setup screen's
six `.court-chip` toggles pick which of them are in play. The picked pool lives in
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
`repeat(6,minmax(0,1fr))` grid so the chips always read as the court numbers in
order instead of reflowing into a ragged last row (fits in one row down to 320px).

`assign()` is unchanged: a uniform random pick among sides that still have room.
