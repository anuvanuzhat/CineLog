# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention already used by `add_to_collection()`. Updated the single call site in `routes/watchlist/watchlist.py` (the POST `/watchlist/<user_id>/add` handler) to use the new name.

**How I verified:**
Searched the whole project for the old name (`grep -r "save_to_watchlist" .`) and confirmed no remaining references. Ran the full test suite (`pytest -v`) to confirm nothing else broke.

## Comment 2 — Deduplication
**What I did:**
Added a check at the start of `add_to_watchlist()` that queries for an existing `WatchlistEntry` matching the given `user_id` and `film_id` before creating a new one. Following the same pattern as `add_to_collection()` (which raises `AlreadyInCollectionError`), I defined a new `AlreadyOnWatchlistError` exception and raise it when a duplicate is detected, rather than silently ignoring the request or using a generic exception type.

**How I verified:**
Manually called the endpoint twice with the same `user_id`/`film_id` and confirmed the second call raises `AlreadyOnWatchlistError` while the watchlist_entry table retains only one row. Added `test_add_to_watchlist_duplicate_raises` in `test_watchlist.py` asserting this behavior, and ran `pytest tests/test_watchlist.py -v` to confirm it passes alongside the rest of the suite.


## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py` and added `test_add_to_watchlist_nonexistent_film_raises`, following the same fixture and assertion structure as `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`. The test calls `add_to_watchlist()` with a nonexistent `film_id` and asserts that `FilmNotFoundError` is raised. Also added `test_add_to_watchlist_creates_entry` and `test_add_to_watchlist_duplicate_raises` to cover the Comment 2 deduplication fix.

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` and confirmed the test passes.

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->