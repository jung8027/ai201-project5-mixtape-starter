# Mixtape Bug Hunt — Submission

## AI Usage

*(This section will be finalized in Milestone 4. Noting Milestone 1 usage now while it's fresh: Claude Code was used to read `app.py`, `models.py`, every file in `routes/` and `services/`, and `seed_data.py` directly via the file-reading tools — not to guess where bugs were, but to build the map below. No fix code has been written yet.)*

## Codebase Map

**`app.py`** — Flask application factory (`create_app`). Initializes a single global `SQLAlchemy` instance (`db`), configures the SQLite URI (`mixtape.db` by default), registers four blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and calls `db.create_all()` inside an app context. Must be started with `FLASK_APP=app:create_app flask run` — running `python app.py` double-imports SQLAlchemy and breaks.

**`models.py`** — Defines all SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`. Three association tables handle many-to-many relationships: `friendships` (symmetric user-to-user, inserted as two rows per friendship), `song_tags` (song-to-tag), and `playlist_entries` (playlist-to-song, which also carries a `position` column, `added_by`, and `added_at` — so playlist order is explicit, not insertion order). Every model has a `to_dict()` used by routes to serialize JSON responses. IDs are UUID strings generated via `generate_uuid()`.

**`routes/`** — Thin HTTP layer, one blueprint per resource area:
- `songs.py` — search, get-by-id, rate, and record-a-listen endpoints. Delegates to `search_service`, `notification_service.rate_song`, and `streak_service.record_listening_event`.
- `playlists.py` — create playlist, get playlist metadata, list/add songs. Delegates to `playlist_service` and `notification_service.add_to_playlist`.
- `users.py` — get user, get streak, get/mark-read notifications. Delegates to `streak_service.get_streak` and `notification_service`.
- `feed.py` — friends-listening-now and activity-feed endpoints. Delegates to `feed_service`.

Every route follows the same shape: parse/validate request data, call exactly one service function, catch `ValueError` and turn it into a 4xx JSON error. No business logic lives in `routes/` — it's all parsing and response formatting.

**`services/`** — All business logic:
- `streak_service.py` — `record_listening_event()` creates a `ListeningEvent` then calls `update_listening_streak()`, which compares `now.date()` against `user.last_listened_at.date()` to decide whether to no-op (same day), increment by 1 (exactly one day has passed and today is not Sunday, per `today.weekday() != 6`), or reset to 1 (any other gap).
- `feed_service.py` — `get_friends_listening_now()` pulls the user's friend IDs, queries `ListeningEvent` rows with `listened_at >= datetime.now(timezone.utc) - RECENT_THRESHOLD` (a 24-hour `timedelta`), dedupes to one event per friend. `get_activity_feed()` is the non-recency-filtered version (most recent N events regardless of age).
- `search_service.py` — `search_songs()` does a case-insensitive `ilike` match on title/artist, outer-joined against `song_tags` to pull tags in the same query.
- `notification_service.py` — `create_notification()` is the single low-level constructor that inserts a `Notification` row. `add_to_playlist()` appends the song to the playlist, commits, and then calls `create_notification()` to tell the original sharer (skipping it if the adder is the sharer). `rate_song()` saves or updates a `Rating` row (upsert on the `user_id`+`song_id` unique constraint) and commits.
- `playlist_service.py` — `create_playlist()`, `get_playlist()`, `get_user_playlists()`, and `get_playlist_songs()` (joins `playlist_entries` and orders by `position` ascending, then returns the song list).

**`seed_data.py`** — Drops and recreates all tables, then inserts 5 users with pre-set friendships and streak values, 10 tags, 25 songs with varying tag counts (0, 1, 3+), 3 playlists, and listening events spanning the past two weeks. This is the fixture data all reproduction steps in Milestone 2 will run against.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py` — existing tests covering `streak_service`, `search_service`, and `playlist_service` respectively. No test file exists yet for `feed_service` or `notification_service`.

### Data flow — a user rates a song

`POST /songs/<song_id>/rate` → `routes/songs.py:rate()` validates `user_id`/`score` are present, casts score to `int`, and calls `notification_service.rate_song(user_id, song_id, score)`. That function validates the score is 1–5, looks up the `Song` and `User`, checks for an existing `Rating` for that `(user_id, song_id)` pair (upsert), commits, and returns the `Rating`. The route serializes it via `rating.to_dict()` and returns 201.

### Data flow — a user adds a song to a playlist

`POST /playlists/<playlist_id>/songs` → `routes/playlists.py:add_song()` → `notification_service.add_to_playlist(playlist_id, song_id, added_by)`, which appends the song to `playlist.songs` if not already present, commits, and then — if the adder isn't the original sharer — calls `create_notification()` to notify `song.shared_by`. The route returns a plain success message rather than serializing a model.

### Patterns noticed

- **Routes never touch models directly** (except trivial lookups in `users.py:get_user`) — all real logic lives in `services/`, and routes only parse input, call one service function, and format the response.
- **Errors are communicated via `ValueError`** raised in the service layer and caught in the route as a 4xx JSON response — there's no custom exception hierarchy.
- **All models expose `to_dict()`** and routes always serialize through it rather than building JSON ad hoc.
- **Association tables carry extra columns when needed** (`playlist_entries.position`/`added_by`/`added_at`) instead of introducing a separate join model class.
- **Notifications are created explicitly by whichever service function triggers them** — there's no event/observer system; a service function calls `create_notification()` directly at the point where it decides a notification is warranted.

## Issue Selection Plan

All five issue descriptions (from the README table and project brief) have been read:

1. Listening streak keeps resetting — `streak_service.py`
2. Friends Listening Now shows people from yesterday — `feed_service.py`
3. Same song shows up twice in search — `search_service.py`
4. Missing notification on song rating — `notification_service.py`
5. Last song in a playlist never shows up — `playlist_service.py`

Planned order for the required 3 (with intent to attempt all 5 as stretch): start with #5 and #3, since both look like tightly-scoped, verifiable logic errors localized to a single function; then #1, which requires reasoning about a specific weekday boundary condition. #2 and #4 will follow if time allows — #2 needs careful reasoning about the recency-window definition, and #4 requires comparing the rating path against the playlist-add path line by line before changing anything, per the hint in the brief.

No code has been changed yet. Next step is Milestone 2 — reproducing each chosen bug against the seeded data before touching any fix code.

## Root Cause Analysis

Reproduction (Milestone 2) confirmed 4 of the 5 issues against the seeded database: #1, #2, #4, and #5. Issue #3 (duplicate search results) could not be reproduced — the code performs an unfiltered `outerjoin` against `song_tags` that does produce duplicate rows at the raw SQL level for multi-tag songs, but SQLAlchemy 2.0's ORM automatically de-duplicates full-entity results by primary key before `search_songs()` returns them (verified via direct calls, the live endpoint, broad queries across the whole seed catalog, and the repository's own pre-existing test, which passes despite its comment expecting a failure). Since there's no observable bug to fix, #3 is excluded here in favor of fixing all 4 of the issues that did reproduce.

### Issue #1 — Listening streak keeps resetting

**How I reproduced it:** Set a test user's `last_listened_at` to a Saturday (`2026-06-27 20:00 UTC`) with `listening_streak = 5`, then called `update_listening_streak(user, now)` with `now` set to the very next day, Sunday `2026-06-28 09:00 UTC` — exactly one calendar day later, which should count as a consecutive-day listen. The streak dropped to `1` instead of incrementing to `6`. Repeating the same one-day gap between any other pair of consecutive weekdays incremented correctly, isolating the failure to gaps that land on a Sunday.

**How I found the root cause:** `record_listening_event()` in `services/streak_service.py` delegates all streak math to `update_listening_streak()`, so that's the only function in scope. Reading it top to bottom, the branch that decides whether to increment is `elif days_since_last == 1 and today.weekday() != 6: user.listening_streak += 1`. The function's own docstring states the rule as "If the user listened yesterday: streak increments by 1" — no exception for any particular weekday is mentioned anywhere in the docstring or the broader spec. The `and today.weekday() != 6` clause is the only thing standing between the documented behavior and the observed behavior, which is what made me confident this was the exact cause rather than just a suspicious area.

**The root cause:** Python's `date.weekday()` returns `6` for Sunday. The increment branch required `days_since_last == 1 and today.weekday() != 6`, so whenever a user's consecutive-day listen happened to land on a Sunday, the `!= 6` check evaluated to `False`, the whole `and` condition became `False`, and execution fell through to the `else` branch, which resets the streak to `1` instead of incrementing it. Every other day of the week satisfied `weekday() != 6` and incremented normally, so the bug was invisible except on the one day this condition specifically excluded.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause entirely, leaving `elif days_since_last == 1: user.listening_streak += 1`. This is the smallest change that removes the erroneous exclusion without touching the same-day no-op branch or the reset-on-skipped-day branch. Side-effect check: ran the full `tests/test_streaks.py` suite (5 tests, all passing after the fix, including the previously-failing `test_streak_increments_on_sunday`) — this covers new-user initialization, same-day no-op, consecutive-day increment, skipped-day reset, and the Sunday case together, confirming the fix didn't disturb the other three branches of the same function.

---

### Issue #2 — Friends Listening Now shows people from yesterday

**How I reproduced it:** Created two fresh, isolated test users (a viewer and one friend, unrelated to the seed data so no other events could contaminate the result) and gave the friend a single `ListeningEvent` timestamped `datetime.now(timezone.utc) - timedelta(hours=23, minutes=50)` — old enough to fall on the previous calendar date, but still inside a 24-hour window. Calling `GET /feed/<viewer's id>/listening-now` showed the friend in the feed despite the event being dated a full calendar day earlier. (My first attempt reused the seeded user `kenji`, who already had a genuine same-day event in the fixture data, which masked the bug — isolating the test to two throwaway users with no other events was necessary to get a clean signal.)

**How I found the root cause:** `routes/feed.py` delegates directly to `feed_service.get_friends_listening_now()`, the only function that builds this feed. It computes `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD` and filters for `ListeningEvent.listened_at >= cutoff`, where `RECENT_THRESHOLD = timedelta(hours=24)` is a module-level constant. There's no other filtering logic in the function — just this one cutoff computation feeding the query — so the constant's definition was the only place left to look.

**The root cause:** "Listening now" was implemented as a flat rolling 24-hour lookback rather than "today" in the calendar sense. Because the window is defined purely by elapsed hours from the current instant, an event from just under 24 hours ago always passes the filter even when it happened on the previous calendar date — e.g. at 9am today, an event from 10am yesterday (23 hours ago) still satisfies `listened_at >= now - 24h`, so a friend who was last active the previous morning is shown as if they were listening "now."

**My fix and side-effect check:** Changed the cutoff from a rolling 24-hour window to the start of the current UTC calendar day: `cutoff = now.replace(hour=0, minute=0, second=0, microsecond=0)`, and removed the now-unused `RECENT_THRESHOLD` constant. This makes "listening now" mean "listened at some point today," which by definition excludes anything dated on a previous day — directly resolving the reported symptom without picking an arbitrary shorter duration. Side-effect check: with the two isolated test users, an event from 23h50m ago (yesterday) is now correctly excluded, while an event from 5 minutes ago (today) is correctly included — confirming the boundary works on both sides, not just the side that was failing. Also confirmed `get_activity_feed()`, the other function in the same file, is untouched (it never referenced `RECENT_THRESHOLD` or any cutoff) and still returns the same non-recency-filtered results as before.

---

### Issue #4 — Missing notification when a friend rates your song

**How I reproduced it:** Picked a song shared by `nova` and, as a different user (`darius`), called `POST /songs/<song_id>/rate` with a valid score. The request succeeded (`201`), but `GET /users/<nova's id>/notifications` showed the same count before and after — no new notification was created. Repeating the equivalent flow for adding the same song to a playlist (`POST /playlists/<id>/songs` as a non-sharer) did produce a new notification for `nova`, confirming the two "friend interacts with my song" flows behave differently even though they're conceptually the same shape of event.

**How I found the root cause:** Both flows live in `services/notification_service.py`. Per the brief's hint that this issue is architectural rather than a typo, I compared `add_to_playlist()` against `rate_song()` line by line. `add_to_playlist()` ends with a conditional call to `create_notification()` (skipped only if the adder is the song's own sharer). `rate_song()` validates the score, upserts the `Rating` row, commits, and returns — there is no call to `create_notification()` anywhere in the function. That absence, contrasted directly against the working pattern in the sibling function, was the root cause; it wasn't a matter of the call being present but broken, it was simply never written.

**The root cause:** `rate_song()` never invokes `create_notification()`. Every other "friend interacted with my song" pathway in the file (`add_to_playlist()`) follows the pattern of committing the primary action and then explicitly notifying the sharer, but that pattern was never applied when the rating feature was implemented. There's no shared/central mechanism that notifications hook into automatically — each service function is responsible for remembering to call `create_notification()` itself — so a path that omits the explicit call simply never notifies anyone.

**My fix and side-effect check:** Added a conditional `create_notification()` call at the end of `rate_song()`, mirroring `add_to_playlist()`'s exact pattern: skip if `song.shared_by == user_id` (don't notify yourself), otherwise create a `"song_rated"` notification naming the rater, the song, and the score. Side-effect check, using the live endpoint: (1) a non-sharer rating the song now produces exactly one new notification for the sharer; (2) the sharer rating their own shared song produces no notification, matching the self-notify skip already used by `add_to_playlist()`; (3) re-rating the same song by the same non-sharer user (the upsert path, where `existing.score` is overwritten rather than a new `Rating` row inserted) still correctly fires a notification, since the notify call sits after the upsert regardless of which branch (`existing` vs. new `Rating`) was taken. Ran the full test suite afterward — no existing test touches `notification_service`, and none regressed.
