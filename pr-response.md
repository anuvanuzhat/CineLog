# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Asked AI to polish up my existing arguments.

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
I'm keeping `public=True` as the default for watchlists.

**Reasoning:**
CineLog is a social, discovery-driven film tracking app, and a new user
joining a platform built around that premise should reasonably expect
their activity — including their watchlist — to be visible to others by
default. Defaulting to public also supports the platform's core value:
letting friends and followers discover what a user wants to watch without
requiring a manual privacy toggle on every list.

**Tradeoff acknowledged:**
The real cost of this default is that a user may not think carefully
about what ends up on a public watchlist, and could end up embarrassed
by having personal or unexpected picks visible to everyone before
they've considered turning the list private. This is a genuine privacy
risk for users who assume their activity is private until they say
otherwise.

## Comment 5 — Sort order
**My position:**
Rather than defaulting purely to alphabetical or date-added, I'm sorting
the watchlist by genre (with a secondary alphabetical sort within each
genre group).

**Reasoning:**
A collection is a record of films a user has already watched — recency
matters there because it reflects an actual timeline of activity, similar
to a diary. A watchlist is different: it's a list of films a user intends
to watch, and every entry carries equal weight regardless of when it was
added — nothing is "more done" than anything else. Grouping by genre
better matches how someone actually uses a watchlist: browsing for
something to watch based on mood, rather than remembering what they
recently added.

**Engagement with reviewer's point:**
I agree that recency is the right default for the collection view, where
the maintainer's reasoning holds — most users do want to see what they
recently watched. But I don't think that same logic transfers to the
watchlist, since a watchlist isn't a timeline of actions, it's an
undifferentiated queue of intentions. Genre grouping addresses the
underlying need (helping a user quickly find something worth watching)
more directly than either alphabetical or date-added would on their own.

## Comment 6 — Rebase
**What conflicted:**
`git rebase origin/main` initially conflicted on `.gitignore` (both branches
added one independently). While resolving that and subsequent conflicts,
the `WatchlistEntry` class was inadvertently dropped from `models.py`
entirely — likely lost when resolving the conflict introduced by main's
UUID refactor, which changed `Film.id` from an integer to a
`db.String(36)` UUID. This wasn't caught immediately because it didn't
surface as a conflict marker; the class was just missing.

**How I resolved it:**
Used `git reflog` to locate my branch's tip commit from immediately
before the rebase began, then used `git show <commit>:models.py` to
recover the original `WatchlistEntry` definition. Re-added it to
`models.py`, updating `film_id` from `db.Column(db.Integer, ...)` to
`db.Column(db.String(36), ...)` to match the new UUID-based `Film.id`.
Also updated `add_to_watchlist()`'s docstring and
`test_add_to_watchlist_nonexistent_film_raises`'s fake film ID from an
integer to a UUID string to match.

**How I verified no conflict remains:**
Ran `grep -rn "<<<<<<<" .` across the repo to confirm no leftover
conflict markers. Ran the full test suite (`pytest -v`) to confirm all
tests pass, including the recreated watchlist model. Ran
`git log --oneline --graph` to confirm a linear history with no merge
commits.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

## What this does

Adds a watchlist feature — users can save films they want to watch later,
separate from their collection of already-watched films.

- New `WatchlistEntry` model
- `add_to_watchlist(user_id, film_id)` — adds a film, raises `FilmNotFoundError`
  if the film doesn't exist, `AlreadyOnWatchlistError` if it's already saved
- `get_watchlist(user_id)` — returns a user's watchlist

## Design decisions

**Visibility default (`public=True`):** CineLog is social and discovery-driven,
so watchlists default to public to support friends/followers discovering what
someone wants to watch. Tradeoff: users may not realize their list is visible
unless they check.

**Sort order:** [FILL IN: alphabetical / date-added / genre — state which and one sentence why]

## How to test

```bash
python app.py

# Add a film to a watchlist
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
  -H "Content-Type: application/json" \
  -d '{"film_id": "<film_id>"}'

# View the watchlist
curl http://127.0.0.1:5000/watchlist/<user_id>

# Duplicate add -> should error, not create a second entry
# Nonexistent film_id -> should error with "film not found"

pytest -v   # all tests should pass
```