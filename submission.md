# submission.md

## Codebase Map

### Main files and their roles

- **`app.py`**: Flask app factory; sets up the app and the database connection (SQLAlchemy `db` instance is defined/initialized here and imported by everything else).
- **`models.py`**: Defines all database tables via SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`. Also defines three association (join) tables: `friendships` (user↔user), `song_tags` (song↔tag), and `playlist_entries` (playlist↔song, with extra columns `position`, `added_by`, `added_at`).
- **`routes/`**: Thin HTTP layer. Each file defines a Flask Blueprint with endpoints that parse the incoming request, call a service function, and format the JSON response. No business logic lives here.
  - `routes/songs.py`: search, song detail, rating, and listening-event endpoints.
  - `routes/playlists.py`: playlist creation, detail, and song management endpoints.
  - `routes/users.py`: user profiles, streaks, notifications.
  - `routes/feed.py`: friends' listening activity feed.
- **`services/`**: All business logic lives here. Routes call into these; the bugs live in this layer.
  - `services/streak_service.py`: listening streak calculation logic (Issue #1).
  - `services/feed_service.py`: "Friends Listening Now" feed logic (Issue #2).
  - `services/search_service.py`: song search logic (Issue #3).
  - `services/notification_service.py`: creates and retrieves notifications; also contains `rate_song()` (Issue #4) and `add_to_playlist()`.
  - `services/playlist_service.py`: playlist creation and retrieval, including `get_playlist_songs()` (Issue #5).
- **`seed_data.py`**: populates the database with test data for local development.
- **`tests/`**: existing test files for streaks, search, and playlists.

### Pattern I noticed

Every route delegates immediately to a service function, routes only handle request parsing and response formatting, while all business logic (queries, validation, side effects like notifications) lives in `services/`. This means when something is wrong at an endpoint, the fix is almost never in the route file itself; you trace back to the service function it calls.

### Data flow: rating a song (and why the notification doesn't fire)

`POST /songs/<song_id>/rate` is handled in `routes/songs.py`, which pulls `user_id` and `score` from the request body and calls `rate_song(user_id, song_id, score)`, imported from `services/notification_service.py`.

Inside `rate_song()`: it validates the score is 1–5, looks up the `Song` and `User`, checks for an existing `Rating` (there's a `UniqueConstraint` on `user_id` + `song_id`, so a user can only have one rating per song, re-rating updates the existing row instead of creating a new one), then commits and returns the `Rating`.

Notably, `rate_song()` never calls `create_notification()`. Compare this to `add_to_playlist()` in the same file: that function does its main job (appending the song to the playlist), commits, and *then* explicitly calls `create_notification(...)` to notify the song's original sharer. `rate_song()` follows the same overall shape (save the thing → commit) but is missing that final notify step entirely. This lines up with Issue #4, ratings save fine, but no notification is ever created because the code that would create one simply isn't there.

### Data flow: viewing a playlist's songs (and why the last song is missing)

`GET /playlists/<playlist_id>/songs` is handled in `routes/playlists.py`, which calls `get_playlist_songs(playlist_id)` in `services/playlist_service.py`.

That function queries `Song`, joined against the `playlist_entries` association table on `song_id`, filtered to the given `playlist_id`, and ordered ascending by the `position` column, so far, correct. But the return statement is:

```python
return [song.to_dict() for song in songs[:-1]]
```

`songs[:-1]` is a Python slice that drops the last item of the (correctly-ordered) list before converting to dicts. Since `position` is ascending, the last item is always the most recently added song, matching Issue #5's report exactly ("the missing one is always whatever was added most recently"). The docstring even claims "this function returns all songs in the playlist," which directly contradicts the slice below it.

## Root Cause Analysis

### Issue #1: My listening streak keeps resetting

**How I reproduced it:** Set a user's `last_listened_at` to a Saturday with `listening_streak = 12`, then called `record_listening_event` with `now` set to the following Sunday (one day later). The streak reset to 1 instead of incrementing to 13.

**How I found the root cause:** Traced `POST /songs/<id>/listen` in `routes/songs.py` to `record_listening_event()` in `streak_service.py`, which calls `update_listening_streak()`. Read the conditional logic line by line and checked what `datetime.weekday()` actually returns.

**The root cause:** The condition `days_since_last == 1 and today.weekday() != 6` was meant to detect a valid consecutive day, but `weekday()` returns 6 for Sunday in Python. So on any Sunday, even with exactly one day since the last listen, the second condition evaluates to `False`, and the code falls through to the `else` branch, resetting the streak to 1 instead of incrementing it.

**My fix and side-effect check:** Removed the `today.weekday() != 6` condition entirely, leaving `elif days_since_last == 1:` as the only check needed to detect a consecutive day. Verified the fix by re-running the reproduction case (Saturday → Sunday) and confirming the streak now increments correctly, and also checked Monday→Tuesday and same-day cases still behave as before.

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** Added a song to a playlist as a different user than the sharer and confirmed a notification appeared. Then rated a different shared song as a different user and checked the sharer's notifications, nothing new appeared.

**How I found the root cause:** Traced `POST /songs/<id>/rate` to `rate_song()` in `notification_service.py`. Compared it line-by-line against `add_to_playlist()` in the same file, which follows a "do the action, then call `create_notification()`" pattern.

**The root cause:** `rate_song()` saves or updates the `Rating` row and commits, but never calls `create_notification()` afterward. Unlike `add_to_playlist()`, which explicitly notifies the song's sharer after its main action, `rate_song()` has no equivalent step, the notification-creation call was simply never added.

**My fix and side-effect check:** Added a call to `create_notification()` at the end of `rate_song()`, guarded so the sharer isn't notified about their own rating (matching the same guard used in `add_to_playlist()`). Verified by re-rating a song and confirming a notification now appears, and confirmed rating your own song still creates no notification.

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:** Fetched a playlist's songs, counted them, added one more song, and re-fetched, the newly added song was missing and the count still matched what was there before.

**How I found the root cause:** Traced `GET /playlists/<id>/songs` to `get_playlist_songs()` in `playlist_service.py`. The query itself correctly joins and orders songs by `position` ascending, but the return statement slices the list.

**The root cause:** `return [song.to_dict() for song in songs[:-1]]` drops the last element of the correctly-ordered list before converting to dicts. Since songs are ordered ascending by `position`, the last element is always the most recently added song, so it's silently excluded from every response.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs` so the full ordered list is returned. Verified by re-fetching a playlist after adding a new song and confirming all songs now appear, including the newest, and confirmed ordering is still correct.