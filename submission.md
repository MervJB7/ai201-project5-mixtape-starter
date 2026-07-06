# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude for codebase orientation (explaining what each service/route file does and tracing the rate-a-song call chain), for help understanding suspicious lines of code once I'd already found them (e.g. what `today.weekday()` returns and why, what `[:-1]` does to a list), and for help figuring out git/pytest workflow issues along the way (activating a venv correctly on Windows, getting stuck in the `less` pager on `git log`, remembering to `git add` before `git commit`). For each bug, I located the suspicious code myself first, then used Claude to help verify my understanding before writing the fix.

## Codebase Map

**`app.py`** — Flask application factory (`create_app`). Configures the SQLAlchemy database URI, initializes the `db` extension, registers the four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()`.

**`models.py`** — All SQLAlchemy models and three association tables:
- `User` — has `listening_streak`, `last_listened_at`, and a self-referential many-to-many `friends` relationship.
- `Song` — has `shared_by` (the user who shared it) and a many-to-many `tags` relationship via `song_tags`.
- `ListeningEvent` — one row per listen, timestamped.
- `Rating` — one row per (user, song) pair, unique-constrained.
- `Playlist` — many-to-many `songs` relationship via `playlist_entries`, which carries `position`, `added_by`, `added_at` — playlists are ordered.
- `Notification` — generic `(user_id, notification_type, body, read)` record.

**`services/`** — All business logic. Every function takes IDs in and returns model instances or `to_dict()` output:
- `streak_service.py` — `record_listening_event` + `update_listening_streak` (day-comparison logic).
- `feed_service.py` — `get_friends_listening_now` (recency-windowed) and `get_activity_feed` (un-windowed).
- `search_service.py` — `search_songs`, `get_song`.
- `notification_service.py` — `create_notification`, and two call sites meant to use it: `add_to_playlist` and `rate_song`.
- `playlist_service.py` — `create_playlist`, `get_playlist_songs`, `get_playlist`, `get_user_playlists`.

**`routes/`** — Thin blueprints: parse the request, call one service function, format the JSON response, translate `ValueError` into 400/404.

**`seed_data.py`** — Populates 5 users with friendships, 25 songs (0/1/3+ tags), 3 playlists of 5–7 songs, and listening events at both recent (~10–20 min) and older (2+ hr) timestamps.

### Data flow: a friend rates your shared song

1. `POST /songs/<song_id>/rate` with `{user_id, score}` → `routes/songs.py::rate()`.
2. `rate()` reads `user_id` and `score` from the JSON body and calls `notification_service.rate_song(user_id, song_id, int(score))`.
3. `rate_song` validates the score range, loads the `Song` and rating `User`, checks for an existing `Rating` for that `(user_id, song_id)` pair, creates or updates it, and commits.
4. This is where Issue #4 lives — `add_to_playlist` in the same file performs its action and then, as a separate explicit step, calls `create_notification` to tell the sharer. `rate_song` never had that second step.

### Pattern I noticed

Routes never contain business logic — every route parses input, calls exactly one service function, and maps `ValueError` to an HTTP error code. Inside `notification_service.py`, notifying the sharer is something each action function has to explicitly opt into by calling `create_notification` itself; it isn't automatic. `add_to_playlist` opts in, `rate_song` didn't — same file, same shape, one missing call.

---

## Root Cause Analysis

### Issue #1: My listening streak keeps resetting

**How I reproduced it:** Ran `pytest tests/test_streaks.py -v` before making changes. `test_streak_increments_on_sunday` failed:
```
update_listening_streak(u, sunday)
>       assert u.listening_streak == 2  # Should increment, not reset
E       assert 1 == 2
```

**How I found the root cause:** Opened `services/streak_service.py` and read `update_listening_streak`. The increment branch was:
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
```
The `today.weekday() != 6` condition had no comment explaining it. Checking what `.weekday()` returns per day confirmed Sunday is `6`, meaning this condition is `False` specifically on Sundays.

**The root cause:** The increment branch requires both `days_since_last == 1` and `today.weekday() != 6`. Since `weekday()` returns `6` on Sunday, that second condition is always false on Sundays, so a genuinely consecutive listen falls into the `else` branch and resets the streak to 1 instead of incrementing it.

**My fix and side-effect check:** Removed `and today.weekday() != 6`, leaving `elif days_since_last == 1:`. Re-ran `pytest tests/test_streaks.py -v` — all 5 passed:
```
tests/test_streaks.py::test_streak_starts_at_1_for_new_user PASSED
tests/test_streaks.py::test_streak_increments_on_consecutive_day PASSED
tests/test_streaks.py::test_streak_does_not_double_count_same_day PASSED
tests/test_streaks.py::test_streak_resets_after_skipped_day PASSED
tests/test_streaks.py::test_streak_increments_on_sunday PASSED
```

---

### Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it:** There's no failing test for this one — I confirmed it by reading `feed_service.py` against the seed data rather than a pytest failure. `seed_data.py` creates a batch of listening events ~10–20 minutes old and a separate batch 2+ hours old for friends of `nova`. Any friend whose only qualifying event was in the 2+ hour batch would still be returned by `get_friends_listening_now` as "listening now," since the filter window was wide enough to include it.

**How I found the root cause:** Opened `services/feed_service.py`. `get_friends_listening_now` filters `ListeningEvent.listened_at >= cutoff`, where `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` and `RECENT_THRESHOLD = timedelta(hours=24)`. A 24-hour window doesn't match a feature meant to show who's listening *right now* — the seed data's own comments describe qualifying "recent" events as within the past 30 minutes, which is the window the feature was actually designed around.

**The root cause:** `RECENT_THRESHOLD` was set to 24 hours instead of a short live-listening window, so any friend who listened at any point in the last day shows up as "listening now," even hours after they actually stopped. The dedup/ordering logic was correct — only the threshold constant was wrong.

**My fix and side-effect check:** Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. Confirmed `get_activity_feed` (which uses `.limit()`, not `RECENT_THRESHOLD`) was unaffected by this change. Ran the full suite afterward — still 13/13 passing, since no existing test targets this feed directly.

---

### Issue #3: The same song keeps showing up twice in search

**How I reproduced it:** `pytest tests/test_search.py` passes all 5 tests even without this fix — I didn't just take that as "no bug." Reading the query directly explains why: `search_service.py::search_songs` joins `Song` to `song_tags` with no filter on `song_tags`, which at the raw SQL level produces one row per matching tag for a song. Whether that surfaces as duplicate entries depends on how SQLAlchemy materializes the result (full ORM entities vs. raw columns), which is consistent with the issue description calling this behavior conditional/inconsistent rather than always reproducing.

**How I found the root cause:** Opened `search_service.py`:
```python
db.session.query(Song)
  .outerjoin(song_tags, Song.id == song_tags.c.song_id)
  .filter(...)
  .all()
```
The join to `song_tags` isn't used by any `.filter()` clause — the `tags` list in the output dict comes from the separate `Song.tags` relationship inside `to_dict()`, not from this join. A join to a many-to-many table with no filter on it only has one effect: it multiplies a song's row once per matching tag.

**The root cause:** The unnecessary join to `song_tags` causes SQL-level row fan-out proportional to a song's tag count. Songs with 0–1 tags never fan out, so the bug only shows up for songs with multiple tags, and even then only depending on how the ORM materializes results.

**My fix and side-effect check:** Removed the `.outerjoin(song_tags, ...)` call and added `.distinct()` as a safeguard, since tags are already available independently via `Song.tags`. Re-ran `tests/test_search.py` — all 5 still passed, confirming tag data is still correctly returned without the join.

---

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** Ran this directly in a Python shell:
```python
sharer = nova, rater = darius, song = one of nova's shared songs
before: 1
rate_song(rater.id, song.id, 5)
after: 2
```
Wait — my fix was already applied when I ran this, which is why the count went up. The bug (before the fix) was that this same script would show `before: 1` / `after: 1` — no new notification created despite a different user rating the song.

**How I found the root cause:** Compared `add_to_playlist` and `rate_song` in `notification_service.py`, since the brief said this bug was architectural rather than a typo. `add_to_playlist` performs its action (append the song to the playlist), then as an explicit second step calls `create_notification(user_id=song.shared_by, ...)`, guarded by `if song.shared_by != added_by_user_id`. `rate_song` only performed its action (save/update the `Rating` row) and returned — there was no equivalent second step.

**The root cause:** Notifying the sharer is implemented as an explicit call each action function has to make on its own, not a shared/automatic side effect. `add_to_playlist` included that call; `rate_song` was written without it, so ratings saved successfully but never generated a notification.

**My fix and side-effect check:** Added a `create_notification` call to `rate_song` after `db.session.commit()`, guarded by `song.shared_by != user_id` so self-ratings don't notify the user about themselves. Verified with the before/after script above — `before: 1`, `after: 2`, confirming a new notification is created when a different user rates the song. Confirmed `add_to_playlist`'s existing behavior was untouched by re-running the full test suite (13/13 passing).

---

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:** Ran `pytest tests/test_playlists.py -v` before making changes:
```
tests/test_playlists.py::test_playlist_returns_all_songs FAILED
tests/test_playlists.py::test_playlist_returns_songs_in_order FAILED
tests/test_playlists.py::test_empty_playlist_returns_empty_list PASSED

E       AssertionError: assert 4 == 5
E       AssertionError: assert ['Track 1', '...3', 'Track 4'] == ['Track 1', '...4', 'Track 5']
```

**How I found the root cause:** Opened `playlist_service.py::get_playlist_songs`. The query (joining `Song` to `playlist_entries`, ordered by `position`) was correct. The return line was:
```python
return [song.to_dict() for song in songs[:-1]]
```
`[:-1]` directly contradicts the function's own docstring, which says it returns all songs in the playlist.

**The root cause:** After correctly querying and ordering all songs, the return statement discards the last song via `songs[:-1]` before returning, regardless of playlist length — a 5-song playlist always reports 4.

**My fix and side-effect check:** Changed the return line to `[song.to_dict() for song in songs]`. Re-ran `pytest tests/test_playlists.py -v` — all 3 passed:
```
tests/test_playlists.py::test_playlist_returns_all_songs PASSED
tests/test_playlists.py::test_playlist_returns_songs_in_order PASSED
tests/test_playlists.py::test_empty_playlist_returns_empty_list PASSED
```
The empty-playlist case was unaffected either way, since slicing an empty list with `[:-1]` was already a no-op.

---

## Test Results

Full suite after all 5 fixes:
```
tests/test_playlists.py::test_playlist_returns_all_songs PASSED
tests/test_playlists.py::test_playlist_returns_songs_in_order PASSED
tests/test_playlists.py::test_empty_playlist_returns_empty_list PASSED
tests/test_search.py::test_search_returns_matching_songs PASSED
tests/test_search.py::test_search_no_duplicates_single_tag_song PASSED
tests/test_search.py::test_search_no_duplicates_multi_tag_song PASSED
tests/test_search.py::test_search_no_duplicates_no_tag_song PASSED
tests/test_search.py::test_search_returns_empty_for_no_match PASSED
tests/test_streaks.py::test_streak_starts_at_1_for_new_user PASSED
tests/test_streaks.py::test_streak_increments_on_consecutive_day PASSED
tests/test_streaks.py::test_streak_does_not_double_count_same_day PASSED
tests/test_streaks.py::test_streak_resets_after_skipped_day PASSED
tests/test_streaks.py::test_streak_increments_on_sunday PASSED

13 passed in 0.65s
```

## Commit Log

```
d51b03f fix: shrink listening-now window from 24h to 30min so stale listens don't appear live
763e048 fix: remove unnecessary song_tags join causing row fan-out for multi-tag songs
014d743 fix: notify song sharer when a friend rates their song (was silent)
34f5718 fix: stop streak from resetting every Sunday due to inverted weekday check
18edc1e fix: stop dropping the last song in every playlist due to [:-1] slice
2dfdeaa Add .gitignore file and update README with setup instructions
7b64551 initial commit
```

## Screenshot
<img width="1387" height="222" alt="logs" src="https://github.com/user-attachments/assets/00dadde6-249b-4911-b367-dd914012dee4" />

