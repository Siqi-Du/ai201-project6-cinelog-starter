# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI to act as a devil's advocate to stress-test my reasoning for the default visibility and sort order decisions. I provided my draft arguments and asked the AI what counterarguments a careful code reviewer would raise and what tradeoffs I might not be acknowledging. For the default visibility, the AI accurately pointed out that privacy concerns could cause users to disengage entirely if they feel their intended watch queue is broadcasted without explicit consent, prompting me to strengthen my tradeoff acknowledgment by suggesting clear UI indicators to balance the social discovery goals.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`. I then updated all corresponding call sites, specifically locating the one usage in `routes/watchlist/watchlist.py`.
**How I verified:** I ran a project-wide search to confirm that absolutely zero instances of `save_to_watchlist` remained in the codebase, and verified the functionality still worked.

## Comment 2 — Deduplication
**What I did:** Added logic to `add_to_watchlist()` to query `WatchlistEntry` and check if a film is already present before adding it, raising an `AlreadyInWatchlistError` if it is. I also updated the route in `routes/watchlist/watchlist.py` to catch this error and return a 409 status code. 
**How I verified:** I modeled the deduplication logic off the existing `add_to_collection()` function in `services/collection_service.py`. To verify it worked, I started the Flask development server and sent manual HTTP POST requests via `cURL`. The first request succeeded with a `201 CREATED`, and repeating the same request successfully returned a `409 CONFLICT` with the expected duplicate error message:

```bash
# First request succeeds with 201 Created
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": 1}'

# Second identical request fails with 409 Conflict
curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": 1}'
```

## Comment 3 — Missing test
**What I did:** Created a new file `tests/test_watchlist.py` and wrote the `test_add_to_watchlist_nonexistent_film_raises` test case to verify that an invalid film addition throws a `FilmNotFoundError`.
**How I verified:** I used `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` as my model, following the exact same fixture structure (`app`, `sample_user`). Finally, I ran `pytest tests/test_watchlist.py -v` to confirm the new test passed perfectly.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for watchlists.
**Reasoning:** CineLog thrives as a social platform where discovering films through peers is a core part of the experience. By defaulting watchlists to public, we optimize for network effects and organic discovery, encouraging users to share their anticipated films.
**Tradeoff acknowledged:** The primary tradeoff is user privacy. Some users might treat a watchlist as a private backlog and might not realize their anticipated films are broadcasted, potentially leading to disengagement if they feel exposed. To balance this, we should ensure the UI provides a clear toggle and explicitly indicates the public default when a user creates their first entry.

## Comment 5 — Sort order
**My position:** Change the default sort order to "date added" (newest first), as suggested.
**Reasoning:** A watchlist functions primarily as a queue of immediate intent. Users are most likely looking for the film they most recently decided to watch. Sorting by date added ensures immediate accessibility to fresh additions.
**Engagement with reviewer's point:** I agree completely with your point. Alphabetical sorting is great for an archival collection where a user is browsing their entire history, but for a dynamic watchlist, temporal relevance is far more important. I will implement this change to sort by `date_added` descending.

## Comment 6 — Rebase
**What conflicted:**
1. **Explicit Conflict (`.gitignore`):** Both branches added `.venv/` and `venv/`, but `main` also added `.pytest_cache/`, causing a standard merge conflict.
2. **Silent Failure (`models.py`):** Git automatically (and incorrectly) removed the `WatchlistEntry` class because `main` had heavily refactored the film IDs to UUIDs in that exact same section.

**How I resolved it:**
1. **`.gitignore`:** I manually resolved the conflict by keeping all the ignore entries (`.venv/`, `venv/`, and `.pytest_cache/`) and removing the Git conflict markers.
2. **`models.py`:** I manually restored the `WatchlistEntry` class back into the file, taking care to update the `film_id` column to a `String(36)` UUID so it correctly aligns with the new schema on `main`.

**How I verified no conflict remains:**
I ran `git add .gitignore` and `git rebase --continue`. After the rebase finished, I ran the full test suite with `pytest tests/ -v` to confirm all 5 tests passed and the restored `WatchlistEntry` functions flawlessly with the new UUIDs.
## PR Description
**Feature Overview**
This PR introduces the new Watchlist feature for CineLog, allowing users to add films they want to watch to a personal queue. It includes the `WatchlistEntry` database model and the `POST /watchlist/<user_id>/add` endpoint with built-in deduplication logic to prevent users from adding the same film twice.

**Design Decisions**
1. **Default Visibility:** Watchlists default to `public=True`. CineLog is a social platform, and this optimizes for network effects and organic film discovery among peers.
2. **Sort Order:** The watchlist is sorted by `date_added` descending (newest first). Since a watchlist is a dynamic queue of immediate intent, temporal relevance is far more important than alphabetical sorting.

**Manual Testing Instructions**
1. Start the Flask development server: `flask run`
2. In a separate terminal, add a film to a user's watchlist:
   ```bash
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
        -H "Content-Type: application/json" \
        -d '{"film_id": "1"}'
   ```
   *Expect: `201 Created`*
3. Send the exact same request again to test deduplication:
   *Expect: `409 Conflict` (Already in watchlist)*

## Git Log Screenshot
![Git log history](https://github.com/user-attachments/assets/8572d79e-efe2-4311-a6b0-a4e1a6bc9e47)