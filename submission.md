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

## Bug Reports (Milestone 2 — Reproduction)

Each bug below was triggered deliberately against the seeded database, using either the live Flask routes (via `app.test_client()`, which exercises the exact same code path as a real HTTP request) or direct calls to the service function with controlled inputs where the route can't supply the needed condition (e.g. a specific `now`). No fix code has been written — this section only confirms what is broken and how to see it break on demand.

---

### Bug #1 — Listening streak keeps resetting
**File:** `services/streak_service.py`
**Description:** Users report their listening streak dropping back to 1 even when they believe they listened on consecutive days.

**How to reproduce:**
1. Take a user with an established streak and set `last_listened_at` to a Saturday (e.g. `2026-06-27 20:00 UTC`, `weekday() == 5`) and `listening_streak = 5`.
2. Call `update_listening_streak(user, now)` with `now` set to the very next day, Sunday `2026-06-28 09:00 UTC` (`weekday() == 6`) — exactly one calendar day later, which should count as a consecutive-day listen.
3. Inspect `user.listening_streak` afterward.

**Result:** Streak went from `5` → `1` instead of `5` → `6`, even though exactly one day had passed since the last listen. The failure is specific to the day being a Sunday — repeating the same test with a Sunday→Monday gap (or any other pair of consecutive weekdays) increments normally.

---

### Bug #2 — Friends Listening Now shows people from yesterday
**File:** `services/feed_service.py`
**Description:** The "Friends Listening Now" feed is supposed to show friends who are currently/recently listening, but users report seeing friends whose last activity was the previous day.

**How to reproduce:**
1. Pick a user with at least one friend (e.g. `nova`, friends with `kenji`).
2. Insert a `ListeningEvent` for that friend with `listened_at = datetime.now(timezone.utc) - timedelta(hours=23, minutes=50)` — old enough that its calendar date differs from today's date (e.g. event dated `2026-07-03` while "now" is `2026-07-04`), but still inside a 24-hour window.
3. Call `GET /feed/<nova's id>/listening-now`.

**Result:** The friend's event showed up in the returned feed (`kenji` appeared in `nova`'s listening-now list) despite the event being dated a full calendar day earlier than "now." Confirmed the event's date (`2026-07-03`) differs from today's date (`2026-07-04`) before checking the feed response, so this isn't a same-day event that merely looks old.

---

### Bug #3 — Same song shows up twice in search
**File:** `services/search_service.py`
**Description:** Users report the same song appearing more than once in search results.

**How to reproduce (attempted):**
1. Confirmed at the raw-SQL level that a song with 3 tags produces 3 joined rows: `search_songs()`'s query does `Song.outerjoin(song_tags, ...)` with no `.distinct()`, and a direct SQL equivalent of that join does return one row per tag (3 rows for a 3-tagged song).
2. Called `search_songs("Crown Heights")` (a song with 3 tags) directly, searched every song in the seed data by title and artist substrings, and tried broad single-letter queries (`"a"`, `"e"`, `"o"`, `"the"`, `"night"`) across the whole catalog.
3. Ran the repo's existing test, `tests/test_search.py::test_search_no_duplicates_multi_tag_song`, whose own comment states "Should be 1, bug causes it to be 3."

**Result: could not reproduce.** Every search above returned each song exactly once, and the existing test **passes** (does not catch a bug) in this environment. The reason: SQLAlchemy 2.0's `Session.query(Song)` (this repo runs SQLAlchemy 2.0.51) automatically de-duplicates full-entity results by primary key before they reach `search_songs()` — even though the underlying join genuinely produces 3 raw rows at the SQL level, the ORM layer collapses them back to 1 Python object. I verified this isn't specific to the `song_tags` table by reproducing the same collapse with an unrelated join (`playlist_entries`) against a song that's in 3 different playlists — still 1 result, not 3.

This needs a decision before Milestone 3: either treat Issue #3 as not currently reproducible in this codebase/dependency combination and substitute a different issue for one of the required 3, or investigate further (e.g., whether the grading environment pins an older SQLAlchemy version where this auto-uniquing doesn't happen). Flagging rather than fixing something that isn't observably broken.

---

### Bug #4 — Missing notification when a friend rates your song
**File:** `services/notification_service.py`
**Description:** Users get notified when a friend adds their shared song to a playlist, but not when a friend rates it.

**How to reproduce:**
1. Pick a song and note its sharer (e.g. a song shared by `nova`).
2. As a different user (e.g. `darius`), call `POST /songs/<song_id>/rate` with `{"user_id": darius_id, "score": 5}`.
3. Before and after, call `GET /users/<nova_id>/notifications` and compare the count.

**Result:** Notification count for the sharer was `0` before and `0` after the rating (rate request itself succeeded with `201`). Repeating the equivalent flow for playlist-add (`POST /playlists/<id>/songs` as a non-sharer) does produce a new notification for the sharer, confirming the two "friend interacts with my song" flows behave differently today.

---

### Bug #5 — The last song in a playlist never shows up
**File:** `services/playlist_service.py`
**Description:** Users report that the last song they see in a playlist in the app is missing when the playlist is viewed via the API/list view.

**How to reproduce:**
1. Pick any seeded playlist (e.g. "Late Night Vibes," which has 7 song entries in the `playlist_entries` table — confirmed via a direct count query).
2. Call `GET /playlists/<playlist_id>/songs`.
3. Compare the returned `count` to the real number of entries.

**Result:** The playlist has 7 entries in the database; the endpoint returned only 6 songs. This reproduced identically on all 3 seeded playlists (each has 7 entries in the DB, each endpoint call returns 6) — the last song by `position` is consistently dropped.

---

**Summary:** 4 of 5 issues (#1, #2, #4, #5) reproduced cleanly and deterministically. #3 did not reproduce despite a genuine, multi-angle attempt (unit-level, endpoint-level, and the pre-existing test suite) — see the note above. No code has been modified; the database was re-seeded (`python seed_data.py`) after this investigation to return to a clean state before any fix work begins.
