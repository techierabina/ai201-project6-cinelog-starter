# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, matching the `verb_to_noun` convention used by `add_to_collection()`. Updated the one call site in `routes/watchlist/watchlist.py` (both the import and the function call).
**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` across the repo to confirm no remaining references, then ran `pytest tests/ -v` to confirm all existing tests still pass.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a deduplication check inside `add_to_watchlist()`, following the same pattern as `add_to_collection()` in `collection_service.py`. Before creating a new `WatchlistEntry`, the function now queries for an existing entry matching `user_id` and `film_id`, and raises `AlreadyInWatchlistError` if one is found.
**How I verified:** Ran `pytest tests/ -v` to confirm the existing collection tests still pass. (A dedicated test for this dedup logic is added in Comment 3/the stretch test.)

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled on `test_add_to_collection_nonexistent_film_raises` from `test_collection.py`. Wrote `test_add_to_watchlist_nonexistent_film_raises`, which calls `add_to_watchlist()` with a film_id that doesn't exist and asserts it raises `FilmNotFoundError`. One difference from the collection test: since `Film.id` is still an integer on this pre-rebase branch (the UUID migration hasn't been rebased in yet), I used an integer fake ID (`999999`) instead of a UUID string, to match the actual current type of `film_id`.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` to confirm the new test passes in isolation, then ran the full `pytest tests/ -v` suite to confirm it doesn't interfere with the existing collection tests.

## Comment 4 — Default visibility
**My position:** I would keep watchlists public by default.
**Reasoning:** CineLog doesn't currently have social or discovery features — no feed, no friends graph, no way to browse other users' watchlists. But keeping the default public means the door stays open for those features later without needing a breaking schema change (flipping millions of existing rows from private to public down the line is a much messier migration than the reverse). A public default is a bet on where the product is headed, not a reflection of what's built today.
**Tradeoff acknowledged:** A watchlist reveals what someone *wants* to watch, which can feel more personal than a collection of things they've already watched — intent hasn't been "socially vetted" the way a finished, rated film has. If a user isn't clearly told their watchlist is public, they could be surprised or exposed without meaning to be. I'd mitigate this by surfacing the visibility setting explicitly in the UI and making it easy to toggle to private — but I'd still keep public as the default, since the goal is to support future discovery, not to hide the feature by default and never validate demand for it.

## Comment 5 — Sort order
**My position:** I agree with the maintainer's preference and changed the default sort order from alphabetical to date-added (newest first).
**Reasoning:** A watchlist functions as an active queue rather than a catalog — when a user opens it, they're usually thinking about what they recently discovered and wanted to watch, not searching for a specific title. Alphabetical order is most useful for quickly checking whether a specific film is already on the list, but that's not the primary use case for most watchlists, which tend to be small enough to scan either way. Switching to date-added also creates consistency with `get_collection()`, which already sorts by `date_added` descending — giving users a predictable ordering pattern across the app, even though the meaning of the timestamp differs (date watched vs. date added).
**Engagement with reviewer's point:** I'm not just deferring to the maintainer's stated preference — I independently landed on the same conclusion based on what a watchlist is actually for. That said, I acknowledge the tradeoff: alphabetical is objectively better for quickly locating a specific title, and date-added order means older entries can effectively "sink" and become harder to find as a watchlist grows — which is exactly the scenario where alphabetical order becomes more useful, not less. The long-term ideal is probably user-selectable sorting (alphabetical, newest, oldest, rating), but for a sensible default, recency better matches the primary use case.

## Comment 6 — Rebase
**What conflicted:** Rebasing `feature/watchlist` onto `upstream/main` completed with no textual merge conflicts, but that was misleading — `main`'s refactor commit had migrated `Film.id` and `CollectionEntry.film_id` from `db.Integer` to `db.String(36)` (UUID), and since `WatchlistEntry` never existed on `main` (it only lived on my branch), git silently dropped the `WatchlistEntry` class entirely from `models.py` during the rebase instead of flagging a conflict. Running `pytest tests/ -v` immediately after the rebase surfaced this as `ImportError: cannot import name 'WatchlistEntry' from 'models'`.
**How I resolved it:** Re-added the `WatchlistEntry` class to `models.py`, updating `film_id` from `db.Integer` to `db.String(36)` to match the new UUID pattern used by `Film.id` and `CollectionEntry.film_id`. I also found and fixed two remaining integer assumptions that weren't caught by the import error: a stale docstring in `add_to_watchlist()` still describing `film_id` as `int`, and a test fixture in `test_watchlist.py` using `fake_film_id = 999999` (an integer) instead of a UUID string. The test had been passing anyway because SQLite doesn't strictly enforce the type mismatch, which meant the test was passing for a coincidentally-wrong reason rather than a correct one.
**How I verified no conflict remains:** Ran `pytest tests/ -v` to confirm all 5 tests pass, then grepped for any remaining integer-style film IDs (`grep -n "film_id" services/watchlist_service.py` and `grep -n "fake_film_id" tests/test_watchlist.py`) to confirm both were updated to UUID strings. Also confirmed `git log --oneline` shows no merge commits — the rebase replayed cleanly as a linear history.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
