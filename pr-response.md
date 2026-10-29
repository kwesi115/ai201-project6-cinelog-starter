# PR Response: feat: add watchlist feature

## AI Usage

I used Claude Code to gather context from the codebase for Comments 4 and 5 (where `public` is set and used, how sorting works, and what each sort option would take to implement) and to stress-test my drafts by pointing out counterarguments and tradeoffs I had not addressed.

What I changed as a result: I kept both positions (private by default, newest first) and revised my wording between the first and final drafts. In Comment 4, I rephrased how users could still share their watchlists and how the tradeoff with discoverability is weighed. In Comment 5, I reframed alphabetical sorting as a possible additional option rather than a vague future feature. I also decided to implement both positions in code, with tests: the `public=False` default and the newest-first sort. The review raised points I did not change my drafts to address: that `public` is not yet enforced by any endpoint, that there is no endpoint yet for making a watchlist public, and that alphabetical order was my original implementation's choice rather than the maintainer's argument.

Other ways I used Claude Code in this project:

- Setup audit: checked my fork, branches, virtual environment, `.gitignore`, app startup, and tests. It found that my fork was missing `feature/watchlist`, and pushed the branch and committed `.gitignore`.
- Orientation: summarized how `add_to_collection()` handles naming, missing films, and duplicates, and the fixture and assertion patterns in `tests/test_collection.py`, so I could compare them against my own reading.
- Comment 1: ran the project-wide search for `save_to_watchlist` and made the rename.
- Comment 2: wrote the duplicate check in `add_to_watchlist()`, the `AlreadyInWatchlistError` class, and the 409 handling in the route, at my request, following the `add_to_collection()` pattern. It also ran the manual double-add check.
- Comment 3: wrote `tests/test_watchlist.py` from the pattern in `test_collection.py`.
- Comments 4 and 5: implemented the code changes for my positions (`public=False` default, newest-first sort) and their tests. The written responses are my own.
- Comment 6: ran the rebase onto `origin/main`, resolved the `models.py` conflict, updated the remaining integer ID references, and checked the result with grep, pytest, and curl.
- Found and fixed the 500 error on `POST /watchlist/<user_id>/add` for a nonexistent film.
- Drafted the factual notes for Comments 1, 2, 3, and 6 and the PR description below, which I reviewed.
- Rewrote the commit history with interactive rebase and checked every commit message against the Conventional Commits format.

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

Related fix: the service raised `FilmNotFoundError` correctly, but `POST /watchlist/<user_id>/add` did not catch it, so a nonexistent film returned a 500 error. The route now returns `404` with `{"error": "No film found with id '<film_id>'"}`, the same way the collection route does.

## Comment 4: Default watchlist visibility (public=True)

**My position:** I would change the default visibility to `public=False`, making watchlists private unless users explicitly choose to share them.

**Reasoning:** CineLog users may add films to their watchlists simply to keep track of what they want to watch, without necessarily wanting to share their interests with others. Setting the default to private gives users greater control over their information and avoids assuming that every user wants their activity to be publicly visible. Users can still choose to make their watchlists public when they want to share recommendations or connect with other film enthusiasts.

**Tradeoff acknowledged:** I recognize that keeping `public=True` as the default could encourage more interaction on CineLog by making watchlists easier to discover and explore. However, I believe user control over privacy should take priority over maximizing discoverability by default. Making watchlists private initially still allows users to participate in the social aspects of CineLog while ensuring that sharing is an intentional choice.

## Comment 5: Watchlist sort order

**My position:** I would sort watchlists by date added, with the most recently added films appearing first.

**Reasoning:** A watchlist is primarily intended to help users keep track of films they want to watch. Displaying the newest additions first allows users to quickly find films they recently saved, especially after receiving recommendations or discovering new releases. This ordering also makes the list more responsive to users' recent activity, rather than organizing films solely by their titles.

**Engagement with the reviewer's point:** I understand the argument for alphabetical ordering because it creates a predictable structure and can make specific films easier to find, particularly as a watchlist grows. However, alphabetical ordering prioritizes locating films by title, whereas date-added ordering prioritizes the user's most recent activity. For CineLog's watchlist feature, I believe the latter better supports the primary use case of tracking films a user intends to watch. If users eventually need more flexibility, alphabetical sorting could be introduced as an additional option rather than the default.

## Comment 6: Rebase onto main (UUID film IDs)

Rebased `feature/watchlist` onto `origin/main` with `git rebase origin/main`. `origin/main` contains `57ec3f9 refactor: migrate film IDs from integer to UUID`, which changed `Film.id`, `CollectionEntry.film_id`, and `WatchlistEntry.film_id` from `db.Integer` to `db.String(36)` in `models.py`. The pre-rebase HEAD was `67dfb5c`.

Conflicts:

- `models.py`, while replaying the commit that added the watchlist model and endpoint. That commit had added `index=True` to `WatchlistEntry.film_id` on the same line that main changed to `db.String(36)`. Resolved by keeping both changes: `db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False, index=True)`.

That was the only conflict. The other 11 commits, including the `public=False` change in `models.py`, replayed cleanly. After the rebase, `git diff` between the old and new branch tips showed only main's UUID refactor in `models.py`.

Changes after the rebase for UUIDs:

- `WatchlistEntry.film_id` was already `db.String(36)` from main's refactor, so the model needed no further change.
- `services/watchlist_service.py`: the `add_to_watchlist()` docstring said `film_id (int)`. Changed to `film_id (str): UUID of the film.`, matching `collection_service.py`.
- `routes/watchlist/watchlist.py`: the `add_film()` docstring said `Body: { "film_id": <int> }`. Changed to `"<uuid>"`, matching `routes/collection.py`. The route reads `film_id` from the JSON body, so there was no URL converter to change.
- `tests/test_watchlist.py`: no change needed. The nonexistent film test already uses the UUID-format ID `"00000000-0000-0000-0000-000000000000"`, the same one `test_collection.py` uses on main.
- Deleted the local `instance/cinelog.db` (gitignored) because it still had the integer schema, then reran `python seed.py`.

Verification:

- `grep` for `(int)`, `<int>`, `<int:`, and `db.Integer` on `film_id` in the watchlist service, route, test, and `models.py` returned zero results.
- `pytest tests/ -v`: 7 passed.
- With the app running, `POST /watchlist/<user_id>/add` with two seeded film UUIDs returned `201` for each with `"public": false`, a repeat POST returned `409`, and `GET /watchlist/<user_id>` returned both films with UUID IDs, newest first.
- `git log --merges origin/main..HEAD` printed nothing, so the branch history is linear with no merge commits.

## PR Description

### What this PR does

Adds a watchlist to CineLog: a list of films a user wants to watch later, separate from their collection of films they have already watched. `POST /watchlist/<user_id>/add` saves a film to the user's watchlist, and `GET /watchlist/<user_id>` returns the films on it. Adding a film that is already on the watchlist is rejected, and adding a film that does not exist returns a 404.

### Design decisions

**Default visibility: private (`public=False`).** New watchlist entries are private unless the user chooses to share them. Users may add films just to keep track of what they want to watch, without wanting to share their interests, so private by default gives them control over their information and makes sharing an intentional choice. The tradeoff is less discoverability and interaction than a public default would give. I decided user control over privacy should come first.

**Sort order: date added, newest first.** `GET /watchlist/<user_id>` returns the most recently added films first. A watchlist is mainly for keeping track of films a user intends to watch, and newest first lets them find films they just saved, for example after a recommendation. Alphabetical order is more predictable for finding a specific title in a long list, but I decided recent activity better fits the main use case. Alphabetical sorting could be added later as an option rather than the default.

### How to test manually

These steps use a bash shell (Git Bash on Windows, or Terminal on macOS/Linux).

1. Create and activate a virtual environment:
   ```bash
   python -m venv .venv
   source .venv/Scripts/activate   # Windows (Git Bash)
   source .venv/bin/activate       # macOS/Linux
   ```
2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Start from a fresh database and seed it. If you have an `instance/cinelog.db` from before the UUID refactor, deleting it is required. Note the `user_id` that `seed.py` prints.
   ```bash
   rm -f instance/cinelog.db
   python seed.py
   ```
4. Start the app (it runs at `http://127.0.0.1:5000`):
   ```bash
   python app.py
   ```
5. In a second terminal, set the user ID from step 3 and look up two film UUIDs:
   ```bash
   USER_ID=<user_id from seed.py>
   curl -s http://127.0.0.1:5000/films/
   FILM_A=<id of "Alien">
   FILM_B=<id of "Spirited Away">
   ```
6. Add the first film. Expect `201` and `"public": false`:
   ```bash
   curl -i -X POST -H "Content-Type: application/json" -d "{\"film_id\": \"$FILM_A\"}" http://127.0.0.1:5000/watchlist/$USER_ID/add
   ```
7. Add the second film. Expect `201`:
   ```bash
   curl -i -X POST -H "Content-Type: application/json" -d "{\"film_id\": \"$FILM_B\"}" http://127.0.0.1:5000/watchlist/$USER_ID/add
   ```
8. Add the first film again. Expect `409` with `{"error": "Film '<id>' is already in this user's watchlist"}`:
   ```bash
   curl -i -X POST -H "Content-Type: application/json" -d "{\"film_id\": \"$FILM_A\"}" http://127.0.0.1:5000/watchlist/$USER_ID/add
   ```
9. Add a film that does not exist. Expect `404` with `{"error": "No film found with id '00000000-0000-0000-0000-000000000000'"}`:
   ```bash
   curl -i -X POST -H "Content-Type: application/json" -d '{"film_id": "00000000-0000-0000-0000-000000000000"}' http://127.0.0.1:5000/watchlist/$USER_ID/add
   ```
10. Get the watchlist. Expect `200` with two films: Spirited Away first (added most recently), then Alien, each with `"public": false`:
    ```bash
    curl -i http://127.0.0.1:5000/watchlist/$USER_ID
    ```
11. Stop the app with Ctrl+C and run the tests. Expect all 7 to pass:
    ```bash
    pytest tests/ -v
    ```
