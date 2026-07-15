# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how AI tools were used during this project -->

## Comment 1 — Rename
> "`save_to_watchlist()` should follow the project's naming convention. Compare with `add_to_collection()` — the pattern here is `verb_to_noun`. Please rename to `add_to_watchlist()` and update all call sites."

**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, matching the `verb_to_noun` pattern already used by `add_to_collection()`, `remove_from_collection()`, and `get_collection()` in `services/collection_service.py`. Also updated the docstring's first line from "Save a film..." to "Add a film..." so it stays consistent with the new name.

**How I verified:**
Before renaming, I ran `grep -rn "save_to_watchlist" --include="*.py" .` to find every reference rather than relying on memory or a single file search. It returned three lines: the function definition itself, the import in `routes/watchlist/watchlist.py`, and the single call site (`entry = save_to_watchlist(...)`) in the same file. I updated all three, then re-ran the same grep — it returned nothing, confirming no stale references were left. I also re-ran `pytest tests/ -v` after the change to confirm nothing else in the codebase broke (the existing collection tests aren't affected by this rename, but this confirms imports across the app still resolve correctly).

## Comment 2 — Deduplication
> "What happens if a user calls this with a film that's already on their watchlist? The current implementation would add a duplicate entry. Please handle this case."

**What I did:**
Looked at `add_to_collection()` in `services/collection_service.py` first: it queries for an existing `CollectionEntry` with the same `user_id`/`film_id` before inserting, and raises a dedicated `AlreadyInCollectionError` if one is found. I followed the identical pattern in `add_to_watchlist()` — added a new `AlreadyOnWatchlistError` exception (scoped to `watchlist_service.py`, not reused from `collection_service.py`, since a duplicate watchlist entry is a distinct condition from a duplicate collection entry) and a `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` check before creating the entry. If a match exists, the function raises before touching the database instead of inserting a duplicate row.

I also noticed the route (`routes/watchlist/watchlist.py`) had no try/except around `add_to_watchlist()` at all — unlike `routes/collection.py`, which catches `FilmNotFoundError` (404) and `AlreadyInCollectionError` (409). Without that, both the pre-existing `FilmNotFoundError` and my new `AlreadyOnWatchlistError` would have surfaced as unhandled 500s instead of clean API responses. I added the same try/except pattern to the watchlist route so duplicate-add and missing-film both return proper JSON errors (409 and 404, respectively) instead of crashing.

**How I verified:**
Ran a manual script that added the same film to a user's watchlist twice in a row inside an app context — the first call succeeded and returned an entry, the second raised `AlreadyOnWatchlistError` with the expected message instead of silently creating a second row. I also confirmed the pre-existing `FilmNotFoundError` path still raises correctly for a well-formed but nonexistent film_id. Ran `pytest tests/ -v` afterward to confirm the collection tests (which exercise the same underlying db session/app factory) were unaffected.

## Comment 3 — Missing test
> "Please add a test for the case where `film_id` doesn't exist in the database. Look at the existing tests in `test_collection.py` — the pattern is there."

**What I did:**
Created `tests/test_watchlist.py`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. I copied the same `app` and `sample_user` fixtures (isolated in-memory SQLite app, per-test setup/teardown via `db.create_all()`/`db.drop_all()`), since watchlist tests need the exact same app-context lifecycle as collection tests. The new test, `test_add_to_watchlist_nonexistent_film_raises`, calls `add_to_watchlist()` with a well-formed but nonexistent `film_id` and asserts it raises `FilmNotFoundError` via `pytest.raises`, matching the assertion style used for `add_to_collection`.

At the time I wrote this (before Comment 6's rebase), `Film.id` on `feature/watchlist` was still an integer, so I used a plainly-nonexistent integer (`999999`) as the fake ID rather than the UUID string `test_collection.py` uses. This is flagged in the Comment 6 entry below — I updated this fixture to a well-formed-but-absent UUID string once the rebase brought in the UUID model change, so it stays consistent with the collection test pattern going forward.

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` in isolation to confirm the new test passes on its own, then ran the full `pytest tests/ -v` to confirm it didn't affect or get affected by the collection tests (they share the same app factory and db fixtures, so this also verified no fixture name collisions).

## Comment 4 — Default visibility
> "I notice watchlists default to `public=True`. We don't have a documented decision on default visibility for user lists. Before I can approve this, I need you to add a note to your PR description explaining your reasoning. I want to make sure we're being intentional here, not just inheriting a default."

**My position:**
The original implementation defaulted `WatchlistEntry.public` to `True`. I changed it to `False` — new watchlist entries are private unless a user explicitly opts in to sharing them.

**Reasoning:**
A CineLog watchlist is primarily a personal planning tool. Adding a film means "I may want to watch this later," which is a weaker and more tentative action than logging a film as watched or rating it. Users may save films because they are curious, because someone recommended them, or simply because they do not want to forget the title. I do not think that action should automatically be treated as something the user intended to publish.

This matters specifically for CineLog because the app supports both personal tracking and community discovery. The social value is real, but visibility should follow the user's intent rather than be inferred from the fact that the platform is social. A user who wants recommendations from friends or wants others to browse their list can still make the watchlist public. Making privacy the default protects users who see the watchlist as a private queue, while still allowing social participation through an explicit visibility choice.

It also avoids an asymmetrical mistake. If the default is private, a user may temporarily miss out on some social discovery until they change the setting. If the default is public, a user may unknowingly expose a list they assumed was personal. The second mistake is harder to reverse because the information may already have been viewed.

**Tradeoff acknowledged:**
The cost of a private default is reduced passive discovery. Public-by-default watchlists would immediately create more visible recommendation data, make profiles feel more active, and help users discover films through other people's interests without requiring extra setup.

I am accepting that reduction in engagement because I think social sharing should be intentional rather than automatic. The default should optimize for the least surprising behavior, not simply for the largest amount of public content. CineLog can still encourage discovery by offering a clear visibility toggle (see the stretch feature below) or asking users whether they want to share their watchlist when they create or edit it.

**Anticipated counterargument (self-reviewed):** A reviewer could push back that CineLog explicitly bills itself as a community app, so defaulting to private undermines a core product value. That's a fair objection — but it conflates *permitting* social discovery with *silently assuming consent* to participate in it. Tentative, forward-looking data ("films I might watch") warrants clearer intent before publishing than completed, backward-looking activity ("films I did watch") — which is a distinction the current `CollectionEntry` model doesn't even need to make, since it has no visibility field at all.

## Comment 5 — Sort order
> "I'd prefer watchlists to default to 'date added' order rather than alphabetical. Most users want to see what they added recently. I'm open to discussion if you see it differently — but let's make a decision and document it."
>
> (A second reviewer, Dani-risingBW, agreed on the same line: "Thinking as user who would want to go to the oldest movie in their watchlist that they added because they finally want to mark it off... Same case for if a user wants to see their most recent watchlist entry. So I agree.")

**My position:**
I agree with changing the default sort order to date added, newest first — implemented in `get_watchlist()` via `.order_by(WatchlistEntry.date_added.desc())`, replacing the previous `.order_by(Film.title.asc())`. I'd phrase the reasoning more narrowly than "most users want to see what they added recently," though.

**Reasoning:**
The maintainer's conclusion makes sense, but the statement by itself assumes a user preference without showing why that preference matters for CineLog specifically. The stronger reason is that a watchlist functions as an active queue of future choices. When users return to it, they are often trying to recover a film they recently discovered, recommended, or saved. Newest-first ordering preserves that recent context and makes the last action easy to verify.

Recency is especially useful for a watchlist because the entries have not yet become completed records. A watched collection (`get_collection()`, which already sorts newest-first) is more archival: users may want to browse it by title, rating, genre, or viewing date because they are reviewing their history. A watchlist is more operational — it helps answer "What was that movie I just saved?" or "What have I recently been considering watching?" Date-added order supports those questions better than alphabetical order, and it also keeps the watchlist consistent with how the collection is already sorted.

Alphabetical sorting is predictable, but it disconnects the list from the sequence in which the user built it. Immediately after adding a film, the user may have to search through the entire list to confirm it was saved. With newest-first ordering, the result of the action appears at the top.

**Engagement with reviewer's point:**
I agree that recently added films are likely to be important, but recency is not the only valid watchlist workflow. Some users may want oldest-first ordering so they can work through films that have been sitting in their backlog the longest — Dani-risingBW's comment on the PR raised exactly this case ("go to the oldest movie in their watchlist... finally want to mark it off"). Others may prefer alphabetical order when searching a large list.

For that reason, I support newest-added-first as the *default*, not as the only meaningful order. It's the best zero-configuration behavior because it reflects the user's latest action and works without requiring any additional input. A future improvement could let users choose newest-added, oldest-added, or alphabetical order while keeping newest-added as the default.

**Anticipated counterargument (self-reviewed):** A reviewer could argue oldest-first better serves the "clear my backlog" use case Dani-risingBW mentioned. That's valid, but it argues for offering a sort *control*, not for changing the *default* — oldest-first requires the user to already have backlog-clearing intent, while newest-first gives useful, low-friction feedback to every user immediately after every add, which is the more common and lower-cost default behavior.

## Comment 6 — Rebase
> "A refactor merged to `main` that changed film IDs from integers to UUIDs. Your watchlist code still references integer IDs. Please rebase on `main` and update accordingly."

**What conflicted:**
`feature/watchlist` branched off `main` before two things landed: a `.gitignore` addition and `refactor: migrate film IDs from integer to UUID` (commit `07ca580`). I ran `git fetch origin main` then `git rebase origin/main`.

The `.gitignore` conflict resolved itself — git detected my branch's own `chore: add .gitignore for generated files` commit had an identical patch to the one already on `main` and silently skipped it ("skipped previously applied commit"), so no duplicate or conflict markers appeared.

The real conflict was in `models.py`, on the commit that changed the `public` default. `main`'s version of `models.py` has no `WatchlistEntry` class at all (it's new work introduced by this branch), while my branch's `Film.id` was still `db.Column(db.Integer, primary_key=True, autoincrement=True)` and `WatchlistEntry.film_id` was `db.Column(db.Integer, db.ForeignKey("film.id"))` — both predating the refactor. Git couldn't auto-merge because both sides touched the tail end of the `CollectionEntry`/`Film` region of the file.

**How I resolved it:**
Kept the incoming `WatchlistEntry` class (it doesn't exist on `main`, so there was nothing to merge it against) and changed `film_id = db.Column(db.Integer, ...)` to `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False)` to match how `CollectionEntry.film_id` was already refactored on `main`. I also removed a stale docstring note at the top of `models.py` that described it as "the post-refactor state on main" — that phrasing only made sense before the branches were combined.

After the rebase completed, I grepped for anything still assuming integer film IDs and found two docstring references (`services/watchlist_service.py`'s `film_id (int)` note and `routes/watchlist/watchlist.py`'s `Body: { "film_id": <int> }`) plus my own Comment 3 test, which used `fake_film_id = 999999` (a valid choice on the pre-rebase integer schema, but no longer representative post-refactor). I updated all three to be UUID-consistent, matching `test_collection.py`'s convention of a well-formed-but-absent UUID (`"00000000-0000-0000-0000-000000000000"`).

**How I verified no conflict remains:**
`grep -n "^<<<<<<<\|^=======\|^>>>>>>>" models.py` returned nothing after resolving. `git log --merges --oneline origin/main..HEAD` returned nothing, confirming a linear history with no merge commits. I ran `pytest tests/ -v` (all 5 pass) and also manually exercised the flow end-to-end in a Python shell against the post-rebase code: created a `Film` and confirmed `film.id` is now a UUID string, called `add_to_watchlist()` with that real UUID, confirmed the entry was created with `public=False`, confirmed calling it again correctly raised `AlreadyOnWatchlistError`, and confirmed `get_watchlist()` returns the entry correctly serialized.

## Stretch — remove_from_watchlist()
**What I did:**
Added `remove_from_watchlist(user_id, film_id)` to `services/watchlist_service.py` and wired it to `DELETE /watchlist/<user_id>/remove` in `routes/watchlist/watchlist.py`, with the same request body shape (`{ "film_id": "<uuid>" }`) as the existing add endpoint.

**How it follows existing patterns:**
Directly mirrors `remove_from_collection()` in `services/collection_service.py`: look up the entry by `user_id`/`film_id` via `.filter_by(...).first()`, raise a dedicated not-found exception (`NotOnWatchlistError`, following the same naming as `NotInCollectionError`) if nothing matches, otherwise `db.session.delete(entry)` + commit and return `True`. The route follows `routes/collection.py`'s `remove_film` pattern too — validates `film_id` is present in the body, catches the service exception, and returns 404 with a JSON error message rather than letting it surface as an unhandled 500.

**Test coverage:**
Two tests in `tests/test_watchlist.py`: `test_remove_from_watchlist_removes_entry` (adds then removes a film, asserts the return value is `True` and the `WatchlistEntry` row is actually gone from the DB) and `test_remove_from_watchlist_not_on_watchlist_raises` (removing a film that was never added raises `NotOnWatchlistError` instead of silently succeeding or crashing).

## Stretch — Second test
**What it covers:**
`test_add_to_watchlist_duplicate_raises` in `tests/test_watchlist.py`, modeled on `test_add_to_collection_duplicate_raises`. It adds a film to a user's watchlist, adds the same film again, asserts the second call raises `AlreadyOnWatchlistError`, and then queries the DB directly to confirm only one `WatchlistEntry` row exists (not two).

**Why I chose this case:**
Comment 3 only required a test for the nonexistent-film path. The Comment 2 deduplication logic I wrote (`add_to_watchlist` raising `AlreadyOnWatchlistError` on a repeat add) had zero test coverage — I'd only verified it manually via a one-off script while implementing it, not with a real automated test. Since duplicate-prevention is the actual behavior change Comment 2 asked for, and it's the collection service's second required test case per `CONTRIBUTING.md` ("a test for duplicate/conflict handling"), it was the most valuable gap to close — it protects the specific bug the reviewer flagged (a duplicate entry silently being created) from regressing.

## Stretch — Visibility toggle endpoint
**What I did:**
Added a `public` parameter to `add_to_watchlist(user_id, film_id, public=False)` in `services/watchlist_service.py`, and to the `POST /watchlist/<user_id>/add` route — the request body now accepts an optional `"public"` boolean alongside `"film_id"`.

**Default behavior:**
If `public` is omitted from the request body, it defaults to `False` (`data.get("public", False)`), matching the private-by-default decision from Comment 4. This means the default behavior is unchanged for any existing caller that doesn't know about the new field — they still get a private entry, they just now also have the option to opt in to public at creation time instead of needing a separate update call.

**How a caller uses it:**
```
POST /watchlist/<user_id>/add
{ "film_id": "<uuid>" }              → creates a private entry (public: false)

POST /watchlist/<user_id>/add
{ "film_id": "<uuid>", "public": true } → creates a public entry (public: true)
```
Verified manually: called `add_to_watchlist()` once with no `public` argument (got `public=False`) and once with `public=True` explicitly (got `public=True`), confirming both the default and the override work as intended. Ran the full test suite afterward — all 8 tests still pass.

## PR Description

### What this feature does
Adds a watchlist to CineLog — a list of films a user wants to watch later, separate from their collection of films already watched. It follows the same shape as the existing collection feature:

- `GET /watchlist/<user_id>` — returns a user's watchlist, newest-added first.
- `POST /watchlist/<user_id>/add` — adds a film to a user's watchlist. Body: `{ "film_id": "<uuid>", "public": false }` (`public` is optional, defaults to `false`).
- `DELETE /watchlist/<user_id>/remove` — removes a film from a user's watchlist. Body: `{ "film_id": "<uuid>" }`.

Adding a film that's already on the watchlist returns `409` instead of creating a duplicate entry. Adding a film that doesn't exist, or removing one that isn't on the watchlist, returns `404`.

### Design decisions
- **Default visibility (Comment 4):** New watchlist entries default to **private** (`public=False`), not the original `public=True`. A watchlist represents tentative future interest, not a completed action like a logged/rated film, so it shouldn't be published without explicit intent — a caller can still set `public: true` on add, or the field could be exposed as an update endpoint later. See the full Comment 4 write-up above for the reasoning and tradeoff.
- **Sort order (Comment 5):** `get_watchlist()` now returns entries newest-added first (`date_added` descending), matching the existing `get_collection()` behavior, instead of alphabetical by title. A watchlist is an active queue people return to shortly after adding to it, so recency is more useful than alphabetical browsing as a default. See the full Comment 5 write-up above for the reasoning and the engagement with the maintainer's original point.

### How to manually test
1. From the repo root, with the virtualenv activated: `python app.py` (starts the app at `http://127.0.0.1:5000`).
2. Create a user and a film via the existing endpoints (or use the seed data / `films.py` and collection endpoints, since there's no dedicated user-creation endpoint — insert directly via a Python shell if needed, e.g. `from app import create_app, db; from models import User, Film`).
3. Add a film to the watchlist:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
   ```
   Confirm the response is `201` and the returned entry has `"public": false`.
4. Repeat the same request — confirm it now returns `409` with an error message, and the watchlist still only has one entry for that film.
5. Add another film with `"public": true` in the body — confirm the returned entry has `"public": true`.
6. `GET /watchlist/<user_id>` — confirm both entries come back, with the most recently added film first.
7. `DELETE /watchlist/<user_id>/remove` with `{"film_id": "<film_uuid>"}` for one of the films — confirm `200` and that a follow-up `GET` no longer shows it.
8. Repeat the same `DELETE` — confirm it now returns `404` instead of silently succeeding.
9. Try `POST /watchlist/<user_id>/add` with a well-formed but nonexistent `film_id` (e.g. `"00000000-0000-0000-0000-000000000000"`) — confirm `404`.
10. Run the automated suite: `pytest tests/ -v` — all 8 tests (4 collection, 4 watchlist) should pass.

## AI Usage
I used AI (Claude Code) throughout this project as an implementation assistant and orientation tool, not as the author of the two design decisions:

- **Codebase orientation:** Before touching any review comment, I had it read `models.py`, `services/collection_service.py`, and `tests/test_collection.py` and summarize the naming conventions, the deduplication pattern, and the test fixture structure, so I understood what "follow the existing pattern" actually meant before implementing the watchlist equivalents.
- **Fetching the real review comments:** Rather than working from a paraphrase, I had it pull the actual six comments (and the two other students' replies visible on the PR) from the GitHub API, so my responses would be grounded in the maintainer's literal wording, including the second reviewer's agreement on Comment 5.
- **Rebase/conflict mechanics:** I used it to run the rebase, resolve the `models.py` conflict against the actual UUID refactor already merged to `main`, and grep for any remaining stale integer-ID references (docstrings, a test fixture) after the rebase completed.
- **Comments 4 and 5 (design decisions):** I wrote my own position and reasoning for both the default-visibility and sort-order decisions myself, in my own words, before asking AI for anything. I then asked it to act as a devil's advocate: "what counterargument would a careful reviewer raise against this position?" For Comment 4, it raised that CineLog markets itself as a community app, so a private default could be read as undermining that value — I addressed this by sharpening my argument to distinguish *permitting* social discovery from *silently assuming consent* to it, rather than softening my position. For Comment 5, it raised that an oldest-first default would better serve users trying to clear a backlog — a real point, which I addressed by explaining why that supports adding a sort *control* rather than changing the *default*. Both counterarguments and my responses to them are included inline in the Comment 4 and Comment 5 sections above, rather than being silently absorbed into the "final" text.
- **Commit hygiene:** Before finalizing, I had it review my `git log --oneline` output and flag any messages that weren't conventional-commit format or that bundled multiple logical changes. It caught one real issue — a commit whose message said "default new watchlists to private" but whose diff actually reintroduced the entire `WatchlistEntry` model (an artifact of resolving the rebase conflict) — which I then split into two commits: one for the UUID/rebase fix (Comment 6) and one for the actual visibility-default change (Comment 4).

### `git log --oneline` screenshot
<!-- Paste screenshot here before submitting -->
