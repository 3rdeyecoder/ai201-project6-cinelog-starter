# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI for codebase orientation and to stress-test design arguments.

- Orientation: asked for summaries of `models.py` and `add_to_collection()` to understand patterns (naming, deduplication, UUID migration).
- Design review: drafted responses for Comments 4 and 5 and asked AI to play devil's advocate; revised arguments where counterpoints were valid.

All final decisions and text are my own and reference the code in this repository.

## Comment 1 — Rename
**What I did:**
- Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`.
- Updated the route import and call in `routes/watchlist/watchlist.py`.

**How I verified:**
- Ran the test suite (`pytest`) — existing collection tests and new watchlist tests passed.
- Searched the repo for remaining `save_to_watchlist` usages to ensure no call sites remain.

## Comment 2 — Deduplication
**What I did:**
- Implemented a deduplication check in `add_to_watchlist()` following the same pattern used by `add_to_collection()`:
  - Query `WatchlistEntry` for an existing row with the same `user_id` and `film_id`.
  - If present, raise `AlreadyInWatchlistError` (new exception type).

**How I verified:**
- Wrote and ran `tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises` and kept the existing collection tests passing.
- Manually inspected code and ran a repo-wide search for patterns to ensure consistency with `collection_service`.

## Comment 3 — Missing test
**What I did:**
- Added `tests/test_watchlist.py` with a test that mirrors the collection test style: `test_add_to_watchlist_nonexistent_film_raises`.

**How I verified:**
- Ran `pytest tests/test_watchlist.py` — test passed.

## Comment 4 — Default visibility
**My position:**
I recommend keeping `public=True` as the default for watchlist entries.

**Reasoning:**
- On CineLog, a watchlist is primarily a personal queue that many users also use discoverably (sharing or syncing across devices). Making entries public by default maximizes discoverability and reduces friction: users can save items quickly without thinking about visibility for each add.
- The codebase already defaults `public=True` in `WatchlistEntry` (see `models.py`), which is a backward-compatible, low-friction behavior.
- If some users prefer privacy, we can add an explicit `public` parameter to the `add_to_watchlist()` API (stretch goal) and update the route to accept it — this preserves the current default while enabling opt-in privacy.

**Tradeoff acknowledged:**
- Defaulting to public may surprise privacy-conscious users. The alternative (default private) would favor privacy but add friction for the common quick-save flow and complicate discoverability features.

I chose the current default because it aligns with the existing schema, minimizes friction, and can be changed later with a non-breaking API extension.

## Comment 5 — Sort order
**My position:**
I recommend sorting the watchlist alphabetically by film title by default, but expose an API parameter for callers who prefer other orders (e.g., `date_added`).

**Reasoning:**
- The watchlist concept in CineLog is a deferred queue: users save films to remember them later. Alphabetical order is stable and deterministic, which helps when scanning a long list and when tests assert deterministic outputs.
- The existing implementation on the feature branch orders by `Film.title.asc()`, which matches this reasoning and minimal-surprise behavior.

**Engagement with reviewer's point:**
- The reviewer suggested `date_added` (newest-first) to surface recent saves. That's a valid UX choice: it emphasizes recency and better supports “what did I just save?” flows.
- To satisfy both use cases, I propose keeping alphabetical order as the default (stable, easy to test) and adding an optional query parameter to `GET /watchlist/<user_id>` (e.g., `?sort=date_added`) that returns newest-first when requested.

This compromise preserves current behavior and testability while enabling the reviewer's preference via a small API addition.

## Comment 6 — Rebase
**What conflicted:**
When rebasing onto `origin/main`, the upstream branch did not include the `WatchlistEntry` model that the feature branch expects. This left `services/watchlist_service.py` referencing a symbol that wasn't defined on main.

**How I resolved it:**
- I added the `WatchlistEntry` model to `models.py` with `film_id` defined as `db.String(36)` (UUID) and a `public` boolean defaulting to `True`.
- Ensured the model follows the same patterns used elsewhere (UUIDs, `date_added` timestamp, `to_dict()` method).

**How I verified no conflict remains:**
- Ran the full test suite (`pytest`) — all tests passed (`5 passed`).
- Checked `git log --oneline origin/main..HEAD` to confirm the branch is rebased cleanly with no merge commits.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
