# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used Claude throughout this project for a mix of orientation, hygiene, and devil's-advocate stress-testing — but the design reasoning and final positions on Comments 4 and 5 are my own.

- **Codebase orientation:** Used Claude to help read through `models.py`, `collection_service.py`, and `test_collection.py` before touching the six review comments, confirming the `verb_to_noun` naming pattern and the dedup/existence-check pattern used in `add_to_collection()`, which I then applied to `add_to_watchlist()`.
- **Git/environment troubleshooting:** Used Claude extensively to debug environment setup issues (a nested clone folder, a broken venv not actually activating, a fork missing `feature/watchlist` requiring an `upstream` remote), and to walk through the interactive rebase (`git rebase -i`) for rewriting commit history into conventional format.
- **Devil's advocate on Comment 4 (default visibility):** I drafted my own position (public by default) and reasoning first. Claude pointed out two factual issues in my draft: I had incorrectly claimed collections were public by default (there's no `public` field on `CollectionEntry` at all), and I had overstated that CineLog already has social/discovery features when it doesn't. I revised both points myself so the final argument accurately reflects the codebase, while keeping my original position.
- **Devil's advocate on Comment 5 (sort order):** I independently reached the same conclusion as the maintainer (date-added order) based on what a watchlist is actually for, then had Claude check my reasoning. It confirmed the consistency argument (that `get_collection()` already sorts by `date_added`) was factually accurate this time, and noted the tension in one of my claims about watchlist size versus alphabetical usefulness — I didn't need to revise the argument, since I'd already acknowledged that tradeoff.
- **Bug discovery during manual testing:** While writing the PR description, Claude prompted me to actually test the endpoints live with `curl` rather than just describing expected behavior. This surfaced two real bugs not covered by the six review comments: missing error handling in the watchlist route (returning `500` instead of `404`/`409`), and a missing `watchlist_entries` relationship on `Film` that caused `GET /watchlist/<user_id>` to fail with an `AttributeError`. Both are documented and fixed above.

I did not ask AI to write the Comment 4 or Comment 5 responses themselves — I wrote my own position and reasoning first in both cases, and used AI only to critique what I'd already written against the actual codebase.

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

## Additional Fixes Found During Manual Testing

While writing the PR description and manually testing the watchlist endpoints with `curl`, I found and fixed two bugs not covered by the six review comments:

1. **Missing error handling in `routes/watchlist/watchlist.py`:** `add_film()` never caught `FilmNotFoundError` or `AlreadyInWatchlistError`, so both would surface as an unhandled `500 Internal Server Error` instead of a clean, informative response. Added a `try/except` returning `404` for a nonexistent film and `409 Conflict` for a duplicate.
2. **Missing `watchlist_entries` relationship on `Film`:** `get_watchlist()` calls `entry.film.to_dict()`, but `Film` only declared a `collection_entries` backref (which is what makes `entry.film` work on a `CollectionEntry`) — there was no equivalent relationship wiring up `film` on `WatchlistEntry`. This meant `GET /watchlist/<user_id>` would raise `AttributeError: 'WatchlistEntry' object has no attribute 'film'` on any watchlist with at least one entry. No existing test caught this since the test suite only covers `add_to_watchlist`'s nonexistent-film case, not `get_watchlist()`. Fixed by adding `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to `Film`, matching the existing pattern for collections.

Both were confirmed fixed via live manual testing (`curl` against a running server) documented in the PR Description below, not just unit tests — this uncovered a real gap in test coverage (no `test_get_watchlist` exists) worth flagging as a follow-up.

## PR Description

### What this feature does

Adds a watchlist feature to CineLog, allowing users to save films they want to watch (distinct from the existing collection feature, which tracks films already watched). Includes a `WatchlistEntry` model, `add_to_watchlist()` / `get_watchlist()` service functions with deduplication and existence validation, and REST endpoints (`GET /watchlist/<user_id>`, `POST /watchlist/<user_id>/add`).

### Design decisions

**Default visibility (`public=True`):** Watchlists default to public. CineLog doesn't currently have social or discovery features, but a public default keeps that door open for the future without requiring a breaking schema migration later. See Comment 4 in this document for full reasoning and the acknowledged tradeoff.

**Sort order (date-added, not alphabetical):** `get_watchlist()` sorts by `date_added` descending (newest first), matching `get_collection()`'s existing pattern. A watchlist functions as an active queue of recent discoveries rather than a catalog to search alphabetically. See Comment 5 for full reasoning and the acknowledged tradeoff.

### Manual testing steps

1. Start the app using the Flask CLI (not `python3 app.py` directly — see note below):
```bash
   flask --app app run --debug
```
2. Create a test user and film directly via the app context (no user/film creation endpoints exist yet):
```bash
   python3 - << 'PYEOF'
   from app import create_app, db
   from models import User, Film
   app = create_app()
   with app.app_context():
       user = User(username="testuser", email="test@example.com")
       film = Film(title="Paddington 2", year=2017, genre="Comedy")
       db.session.add_all([user, film])
       db.session.commit()
       print("USER_ID:", user.id)
       print("FILM_ID:", film.id)
   PYEOF
```
3. Add the film to the watchlist — expect `201 CREATED` with the new entry as JSON:
```bash
   curl -i -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
```
4. Repeat the same request — expect `409 CONFLICT` with `{"error": "Film '<film_id>' is already on this user's watchlist"}`.
5. Retry with a nonexistent film ID (e.g. `00000000-0000-0000-0000-000000000000`) — expect `404 NOT FOUND` with a matching error message.
6. View the watchlist — expect `200 OK` with the film's full data plus `date_added` and `public`:
```bash
   curl -i http://127.0.0.1:5000/watchlist/<user_id>
```
7. Run the automated test suite: `pytest tests/ -v` — all 5 tests should pass.

**Note on running the app:** Use `flask --app app run --debug` rather than `python3 app.py` directly. Running the file directly causes Python to import `app.py` twice under two different module names (`__main__` and `app`), creating two separate `SQLAlchemy` instances — one of which never receives `init_app()`. This surfaces as `RuntimeError: The current Flask app is not registered with this 'SQLAlchemy' instance` on the first database call. The Flask CLI avoids this by importing the module consistently.

All four scenarios above (create, duplicate, nonexistent, view) were manually verified against a live running server and returned the expected status codes and payloads.


## Commit History Screenshot
<img width="1110" height="409" alt="Screenshot 2026-07-11 at 4 45 23 PM" src="https://github.com/user-attachments/assets/a62f0bb2-67d0-4d45-afdf-9ce641551517" />

