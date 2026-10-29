# PR Response: feat: add watchlist feature

## PR Description

## Comment 1: Rename save_to_watchlist() to add_to_watchlist()

Renamed `save_to_watchlist()` to `add_to_watchlist()` to follow the project's `verb_to_noun` naming convention used by `add_to_collection()`.

Call sites were found with:

```
grep -rn "save_to_watchlist" . --exclude-dir=.venv --exclude-dir=.git --exclude-dir=__pycache__ --exclude-dir=.pytest_cache
```

That search returned three hits, and these files changed:

- `services/watchlist_service.py`: the function definition.
- `routes/watchlist/watchlist.py`: the import and the call in `add_film()`.

Running the same search after the change returned zero results. All existing tests still pass.

## Comment 2: Duplicate entries in add_to_watchlist()

I followed the pattern in `add_to_collection()` in `services/collection_service.py`. That function:

1. Checks that the film exists with `db.session.get(Film, film_id)` and raises `FilmNotFoundError` if it does not.
2. Queries for an existing entry matching both `user_id` and `film_id` with `CollectionEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`.
3. If a match is found, raises `AlreadyInCollectionError` without adding or committing anything.

Changes made to match it:

- Added `AlreadyInWatchlistError` to `services/watchlist_service.py`, defined the same way as `AlreadyInCollectionError`.
- `add_to_watchlist()` keeps the film check first, then runs the same two-column `filter_by(user_id=..., film_id=...).first()` query on `WatchlistEntry`. On a match it raises `AlreadyInWatchlistError("Film '<film_id>' is already in this user's watchlist")` and nothing is added or committed.
- `POST /watchlist/<user_id>/add` now catches `AlreadyInWatchlistError` and returns `409` with `{"error": "<message>"}`, the same way the collection route handles `AlreadyInCollectionError`.

Note: `CollectionEntry` also has a database-level `UniqueConstraint("user_id", "film_id")`. `WatchlistEntry` does not, so for watchlists the check is enforced only in the service layer. I did not change the model in this PR.

Verification:

- `pytest tests/ -v`: all tests pass.
- Manual double-add with the same `user_id` and `film_id` on an in-memory database:
  - Service: the first `add_to_watchlist()` call created an entry, the second raised `AlreadyInWatchlistError`, and the row count stayed at 1.
  - Endpoint: the first `POST /watchlist/<user_id>/add` returned `201`, the second returned `409` with `{"error": "Film '1' is already in this user's watchlist"}`, and the row count stayed at 1.

## Comment 3: Test for nonexistent film_id

I used `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` as the model.

Added `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`. It uses the same `app` and `sample_user` fixtures (in-memory SQLite), calls `add_to_watchlist()` with the placeholder ID `"00000000-0000-0000-0000-000000000000"`, and asserts that it raises `FilmNotFoundError`.

The test passes with `pytest tests/test_watchlist.py -v` and with the full suite `pytest tests/ -v`.

## Comment 4: Default watchlist visibility (public=True)

## Comment 5: Watchlist sort order

## Comment 6: Rebase onto main (UUID film IDs)

## AI Usage
