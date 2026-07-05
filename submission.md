# Mixtape — Codebase Map

## Main files and their responsibilities

**`app.py`** — Application factory (`create_app`). Configures the SQLAlchemy
DB URI (env `DATABASE_URL`, defaults to local sqlite file), initializes the
`db` extension, registers all four blueprints under their URL prefixes
(`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()` at
startup. `db = SQLAlchemy()` lives here (not in `models.py`) — `models.py`
imports `db` back from `app.py`, so `app.py` must be importable without
circular-importing the models. This is also why `python app.py` breaks
(double import of `app` under two different module names, `__main__` vs
`app`) and `flask run` with `FLASK_APP=app:create_app` is the required entry
point — it only imports the module once, as `app`.

**`models.py`** — All SQLAlchemy models and association tables:
- `User` — has `listening_streak` / `last_listened_at` columns directly on
  the row (not a separate streak table), plus a symmetric self-referential
  `friends` many-to-many via the `friendships` table.
- `Song` — `shared_by` (FK to User) + `shared_at` records who shared a song
  and when. Tags are many-to-many via `song_tags`.
- `ListeningEvent` — one row per (user, song, timestamp) "user listened to
  song X at time T" — this is what both the streak logic and the feeds
  are built from.
- `Rating` — one row per (user, song) with a `UniqueConstraint`, so a
  second rating from the same user updates rather than duplicates.
- `Playlist` — songs are attached via `playlist_entries`, a many-to-many
  table that (unlike `song_tags`) carries extra columns: `position`,
  `added_by`, `added_at`. Position is what gives playlists an explicit
  order instead of relying on insertion/PK order.
- `Notification` — generic `notification_type` + `body` string, no
  polymorphic subtype — every notification is just a row with a type tag
  and a pre-rendered message.

**`routes/`** — One blueprint per resource (`songs`, `playlists`, `users`,
`feed`). Every handler follows the same shape: parse `request.args` /
`request.get_json()`, call exactly one service function, catch `ValueError`
and turn it into a `400`/`404` JSON error, otherwise `jsonify(...)`. No
business logic lives here.

**`services/`** — All business logic, one module per feature area:
- `streak_service.py` — `record_listening_event` (create a `ListeningEvent`
  + update streak) and `update_listening_streak` (the day-diff math).
- `feed_service.py` — `get_friends_listening_now` (last 24h, deduped to one
  entry per friend) and `get_activity_feed` (last N events, no time filter).
- `search_service.py` — `search_songs` (title/artist `ILIKE` match) and
  `get_song`.
- `notification_service.py` — generic `create_notification`, plus
  `add_to_playlist` and `rate_song`, which are two_places business actions
  live (not just notification creation — `add_to_playlist` also mutates
  the playlist, and `rate_song` also mutates the `Rating` row).
- `playlist_service.py` — playlist CRUD-ish reads: `create_playlist`,
  `get_playlist`, `get_playlist_songs`, `get_user_playlists`.

**`seed_data.py`** — Drops and recreates all tables, then inserts 5 users
(with friendships), 25 songs (with deliberately varying tag counts — 0, 1,
3+ — the multi-tag songs are explicitly called out as the ones that
exercise Issue #3), 3 playlists, a mix of recent (~10–20 min old) and old
(hours-to-days old) `ListeningEvent`s, per-user `last_listened_at`
snapshots, and one known-good "song added to playlist" notification. It's
worth noting the seed comments directly telegraph which data exists to
expose which issue — e.g. "Older events... should NOT appear in listening
now after fix" implies the current behavior does show them.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`.
These aren't just regression tests; several assertions describe the
*intended* behavior that the current code doesn't satisfy yet (e.g.
`test_streak_increments_on_sunday` explicitly asserts a Saturday→Sunday
listen should increment the streak to 2). Running `pytest tests/` before
touching any service is effectively a pre-built list of which behaviors
are currently broken.

## Data flow — a user rates a song

1. Client sends `POST /songs/<song_id>/rate` with JSON body
   `{user_id, score}`.
2. `routes/songs.py:rate()` parses the body, validates presence of
   `user_id`/`score`, and calls `notification_service.rate_song(user_id,
   song_id, int(score))`.
3. `rate_song()` (in `notification_service.py`, not `search_service.py` —
   ratings live alongside notifications because rating is meant to also
   notify someone) validates the score is 1–5, loads the `Song` and rater
   `User`, and checks for an existing `Rating` row for that
   (user_id, song_id) pair — if found it updates the score in place
   (enforced uniquely by the DB's `UniqueConstraint`), otherwise inserts
   a new `Rating`. It commits and returns the `Rating`.
4. Contrast with `add_to_playlist()` in the same file: after mutating the
   playlist, it explicitly calls `create_notification(...)` to notify the
   song's original sharer ("X added your song to playlist Y"). `rate_song`
   has no equivalent call — nothing in the rate path ever creates a
   `Notification` row, so the sharer is never told their song was rated.
   This matches the way the route layer treats the two actions
   identically (both are just a `POST` that calls one service function and
   returns 201) — the asymmetry is entirely inside `notification_service.py`.
5. `routes/songs.py` serializes the `Rating` via `.to_dict()` and returns
   `201`.

So: rating a song *does* persist correctly and *does* return a valid
response — the missing piece is purely the notification side-effect that
the parallel `add_to_playlist` flow has and `rate_song` doesn't.

## Patterns noticed

- **Routes are pure adapters.** Every route: parse input → call one
  service function → catch `ValueError` → `jsonify`. There is no ORM usage,
  no business rule, and no direct `db.session` call in any route file
  (except `users.py`'s `get_user`, which reads a single row directly rather
  than going through a service — the one inconsistency in an otherwise
  strict layering).
- **Errors are `ValueError` + a message, uniformly.** Services never raise
  custom exception types; routes never do their own existence checks —
  they let the service raise and translate `ValueError` to 400 or 404
  depending on the endpoint's semantics (400 for "bad input to a write",
  404 for "resource doesn't exist").
- **`to_dict()` lives on the model, not the service or route.** Serialization
  shape is defined once per model and reused everywhere that model is
  returned.
- **Many-to-many tables are used for two different purposes.** `song_tags`
  and `friendships` are plain association tables (existence = relationship).
  `playlist_entries` is an association table *with data* (`position`,
  `added_by`, `added_at`) — it's really an entity in its own right, just
  modeled as a many-to-many secondary table instead of its own model class.
  Any query that joins through `playlist_entries` needs to be careful about
  row multiplication / ordering in a way that `song_tags` joins don't,
  since `song_tags` join fan-out only affects `tags`, but joining a Song
  query directly against `song_tags` can duplicate the base `Song` rows
  when a song has more than one tag.
- **Cross-imports between services are local, not top-level.** e.g.
  `notification_service.add_to_playlist` does `from services.playlist_service
  import get_playlist_songs` (unused after import, actually) and `from
  models import Playlist` inside the function body rather than at module
  top — likely to avoid circular imports, since `playlist_service.py`
  doesn't import `notification_service.py` but the reverse does.
- **Time handling is UTC-first but defensive about naive datetimes.**
  `streak_service.update_listening_streak` explicitly reattaches
  `timezone.utc` to `user.last_listened_at` if it comes back naive
  (e.g. from sqlite, which doesn't preserve tzinfo), suggesting this bit
  everyone once already.

## The five open issues (read, not yet fixed)

| # | Title | Service | Notes from reading |
|---|-------|---------|---------------------|
| 1 | Listening streak keeps resetting | `streak_service.py` | `update_listening_streak` has a `today.weekday() != 6` condition gating the increment branch — on a Sunday, a legitimate consecutive-day listen falls through to the reset branch instead. Confirmed by the existing (failing) test `test_streak_increments_on_sunday`. |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` | `get_friends_listening_now` filters by `RECENT_THRESHOLD = timedelta(hours=24)` against `now`. Looks structurally correct at a glance; needs closer tracing against how `listened_at` is stored/compared (naive vs aware) before concluding where the 24h boundary actually leaks. |
| 3 | Same song shows up twice in search | `search_service.py` | `search_songs` does an `outerjoin` against `song_tags` directly on the `Song` query, then `.all()`s the `Song` rows with no dedup — a song with N tags will join-fan-out into N duplicate rows. Seed data explicitly creates multi-tag songs to expose this. |
| 4 | Notified for playlist-add but not for rating | `notification_service.py` | `add_to_playlist` calls `create_notification(...)` after mutating the playlist; `rate_song` has no equivalent call anywhere in its body. Traced in the data flow above. |
| 5 | Last song in a playlist never shows up | `playlist_service.py` | `get_playlist_songs` builds the correctly-ordered `songs` list, then returns `songs[:-1]` — an off-by-one slice that drops the last element every time. |

## Rough plan

All five root causes are now understood well enough to fix, but per the
brief only three are required. Issues #1 (single boolean condition), #4
(single missing function call), and #5 (single slice bug) are the most
surgical, one-line-root-cause fixes with existing tests/seed data to verify
against, so they're the strongest first three. Issue #3 needs a `.distinct()`
or restructured query rather than a one-line change, and Issue #2 needs more
tracing before the actual defect is clear — good candidates for a second
pass.

## Root cause analysis

For each chosen bug, the reported behavior was reproduced against the
actual pre-fix code before any fix was written (`git checkout <pre-fix
commit> -- <file>`, run a script that triggers the exact condition, confirm
the bug, then restore the fix and re-run `pytest tests/` to confirm no
regressions).

### Issue #1 — Listening streak keeps resetting

- **Root cause:** `update_listening_streak()` in `services/streak_service.py`
  gated the consecutive-day increment on `days_since_last == 1 and
  today.weekday() != 6`. Since Sunday is `weekday() == 6`, a legitimate
  one-day gap that lands on a Sunday fails the `and` clause and falls
  through to the `else` branch, resetting the streak to 1 instead of
  incrementing it.
- **How I reproduced it:** In an isolated in-memory DB, created a fresh
  user and called `update_listening_streak(user, saturday)` (a Saturday,
  `weekday()==5`) followed by `update_listening_streak(user, sunday)` (the
  very next calendar day, `weekday()==6`). Expected streak after the Sunday
  call: 2 (consecutive day). Actual result on the pre-fix code: streak
  stayed at 1, reproducing the reported reset. This matches the existing
  (previously failing) test `test_streak_increments_on_sunday`.
- **Fix:** removed the `and today.weekday() != 6` clause so any one-day
  gap increments the streak regardless of day of week.

### Issue #4 — No notification when a friend rates my song

- **Root cause:** `rate_song()` in `services/notification_service.py`
  persists the `Rating` row and commits, but never calls
  `create_notification()` — unlike the parallel `add_to_playlist()`
  function in the same file, which explicitly notifies the song's sharer
  after mutating the playlist.
- **How I reproduced it:** In an isolated in-memory DB, created a sharer
  user and a rater user (different from the sharer), had the sharer own a
  song, checked the sharer's notification count (0), then called
  `rate_song(rater.id, song.id, 5)`. Expected: sharer's notification count
  becomes 1, with a `song_rated` entry. Actual result on the pre-fix code:
  notification count stayed at 0 — the sharer is never told their song was
  rated.
- **Fix:** added a `create_notification(...)` call (type `"song_rated"`)
  after the rating commit, skipped when the rater is the sharer themselves
  (mirroring the self-notification guard already present in
  `add_to_playlist`).

### Issue #5 — The last song in a playlist never shows up

- **Root cause:** `get_playlist_songs()` in `services/playlist_service.py`
  builds the correctly position-ordered `songs` list, then returns
  `[song.to_dict() for song in songs[:-1]]` — an off-by-one slice that
  drops the last element of every playlist, regardless of size.
- **How I reproduced it:** In an isolated in-memory DB, created a playlist
  with 4 songs at positions 1–4 and called `get_playlist_songs()`.
  Expected: 4 songs returned, including "Song 4" (highest position).
  Actual result on the pre-fix code: only 3 songs returned, with "Song 4"
  missing — reproducing the reported behavior exactly (not "some song
  missing at random," but specifically always the last one by position).
- **Fix:** return `[song.to_dict() for song in songs]` (drop the `[:-1]`
  slice).

### Issue #3 — not reproduced (deferred)

Attempted to reproduce the reported duplicate-search-result behavior by
seeding a song with 3 tags and calling `search_songs()` on it (the
`search_service.py` query does an `outerjoin` against `song_tags` directly
on the `Song` query with no `.distinct()`, which looked like a clear
row-fanout candidate). Result: the song came back exactly once, not
duplicated — the bug did not reproduce against the current SQLAlchemy
version in this environment. Since the reported behavior couldn't be
triggered, this issue was set aside in favor of the three above rather than
"fixed" speculatively.
