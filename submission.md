# Mixtape Bug Hunt Submission

## AI Usage
I used GitHub Copilot (GPT-5.3-Codex) primarily for codebase orientation, call-chain tracing, and root-cause verification.

- During orientation, I asked for module summaries and traced route -> service -> model data flow to build a map before touching bug fixes.
- During debugging, I used AI assistance to compare similar code paths (for example, working playlist notification flow vs. missing rating notification flow) and to sanity-check date/time edge-case reasoning.
- I verified every diagnosis by reading the source directly and reproducing behavior with tests or controlled app-context scripts before applying a fix.
- One limitation I observed: plausible AI explanations are not enough by themselves; reproduction and source verification were necessary before making changes.

## Codebase Map (Orientation First)
### Main files and roles
- `app.py`: Flask app factory, SQLAlchemy initialization, and blueprint registration (`songs`, `playlists`, `users`, `feed`).
- `models.py`: SQLAlchemy schema for all core entities (`User`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, `Tag`) plus join tables (`friendships`, `song_tags`, `playlist_entries`).
- `routes/`: HTTP layer only. Routes parse inputs and call service functions; they mostly avoid business logic.
  - `routes/songs.py`: search, song detail, rating, listen endpoints.
  - `routes/playlists.py`: playlist creation, detail, list songs, add song.
  - `routes/users.py`: user profile, streak, notifications, mark-read.
  - `routes/feed.py`: friends listening-now feed and activity feed.
- `services/`: business logic layer where all 5 listed bugs live.
  - `services/streak_service.py`: streak updates tied to listening events.
  - `services/feed_service.py`: listening-now recency filtering + activity feed.
  - `services/search_service.py`: song search query logic.
  - `services/notification_service.py`: create/retrieve notifications, rating and playlist side effects.
  - `services/playlist_service.py`: playlist creation and ordered song retrieval.
- `tests/`: service-level regression coverage for streak/search/playlist behavior.
- `seed_data.py`: deterministic-ish local dataset with users, songs, tags, playlists, listening events, and sample notifications.

### Data flow example: user rates a song and should notify original sharer
1. `POST /songs/<song_id>/rate` hits `rate()` in `routes/songs.py`.
2. Route validates JSON payload (`user_id`, `score`) and calls `rate_song(user_id, song_id, score)`.
3. `services/notification_service.py:rate_song()` validates score and entities, upserts a `Rating`, commits.
4. Expected side effect: if rater is not the original sharer, create a `Notification` for `Song.shared_by`.
5. `GET /users/<user_id>/notifications` in `routes/users.py` calls `get_notifications()` to return notifications.

### Pattern observed
A consistent architecture is used: routes do request/response concerns while services perform business logic and persistence. This made bug tracing faster by following route entrypoints into exactly one service function per feature.

## Root Cause Analysis

### 1) Issue #1 - My listening streak keeps resetting
1. Issue number and title
- Issue #1 - My listening streak keeps resetting

2. How I reproduced it
- Reproduced with existing test `tests/test_streaks.py::test_streak_increments_on_sunday`.
- Sequence: call `update_listening_streak` on Saturday then Sunday for the same user.
- Observed pre-fix behavior: streak was reset to `1` on Sunday instead of incrementing to `2`.

3. How I found the root cause
- Navigation path: route (`routes/songs.py` listen endpoint) -> `record_listening_event()` -> `update_listening_streak()` in `services/streak_service.py`.
- The confidence point was the explicit conditional `days_since_last == 1 and today.weekday() != 6`, which special-cased Sunday and directly matched the failing scenario.

4. The root cause
- The increment condition incorrectly excludes Sundays (`weekday() == 6`).
- For a valid consecutive day transition Saturday -> Sunday (`days_since_last == 1`), the code falls into the reset branch, causing an incorrect streak reset.

5. Your fix and side-effect check
- Fix: removed the Sunday exclusion and incremented streak whenever `days_since_last == 1`.
- Why it works: streak logic should be based on consecutive-day distance, not specific weekday values.
- Side-effect checks:
  - Ran streak tests to ensure same-day no-double-count and skipped-day reset still behave correctly.
  - Verified Sunday case now increments.

### 2) Issue #5 - The last song in a playlist never shows up
1. Issue number and title
- Issue #5 - The last song in a playlist never shows up

2. How I reproduced it
- Reproduced with existing tests:
  - `tests/test_playlists.py::test_playlist_returns_all_songs`
  - `tests/test_playlists.py::test_playlist_returns_songs_in_order`
- Observed pre-fix behavior: playlist returned 4 songs instead of 5 and was missing the final track.

3. How I found the root cause
- Navigation path: `GET /playlists/<playlist_id>/songs` in `routes/playlists.py` -> `get_playlist_songs()` in `services/playlist_service.py`.
- The confidence point was the final return expression using `songs[:-1]`, which always drops the last list item.

4. The root cause
- The function intentionally (but incorrectly) slices the query result with `[:-1]` before serializing.
- Because Python slicing excludes the last element in that form, every playlist response omits the final song regardless of playlist size.

5. Your fix and side-effect check
- Fix: changed return logic from iterating `songs[:-1]` to iterating all `songs`.
- Why it works: it returns the complete ordered query result without truncation.
- Side-effect checks:
  - Ran playlist tests for total count, ordering, and empty-playlist behavior.
  - Confirmed order remains position-based while now including the last track.

### 3) Issue #4 - Playlist notification works, rating notification missing
(To be completed with full 5-field RCA in the commit for Issue #4.)

### 4) Issue #2 - Friends Listening Now shows people from yesterday
(To be completed with full 5-field RCA in the commit for Issue #2.)
