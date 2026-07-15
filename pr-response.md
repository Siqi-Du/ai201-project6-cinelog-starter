# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

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