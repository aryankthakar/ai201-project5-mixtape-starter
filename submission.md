# Project 5 — Mixtape Bug Hunt — Submission

## AI Usage

I used Claude (Claude Code) throughout for codebase navigation, code explanation, execution
tracing, and drafting, with verification happening at each step rather than at the end.

**Navigation and orientation (Milestone 1).** I had Claude read `README.md`, `models.py`, every
file in `routes/` and `services/`, and `seed_data.py`, then summarize each file's responsibility
and trace two call chains (rating a song, viewing a playlist). This produced the codebase map. I
did not independently re-read every file line by line before accepting the map, but I did read
the two traced call chains myself against the actual source to confirm the summary matched.

**Diagnosis, with a case where the AI's initial plan was wrong.** For Issue #3, Claude's original
plan (from reading the code) assumed a tag-join `outerjoin` would multiply rows and produce
duplicate songs. This looked plausible before running anything. Claude then wrote a script that
called `search_songs` directly with about ten queries, including songs with three tags, and every
result set came back with zero duplicates. Because the reproduction step came before any fix was
attempted, this wrong hypothesis was caught immediately rather than being "fixed" against a bug
that doesn't manifest in the code as written. Claude swapped to Issue #4 instead of forcing a fix
onto an unreproducible bug. I confirmed this decision was reasonable by checking that the existing
test suite (`test_search.py`) explicitly asserts no duplicates for multi-tag songs, which is
independent evidence the swap was correct, not just a coincidence of Claude's ten test queries.

**Investigation and fixes (Milestone 3).** For each of the three fixed bugs, Claude isolated the
relevant service function in a Python shell (per the brief's suggested workflow), reproduced the
bug with a script before touching source, then proposed a minimal fix. I reviewed the reasoning
behind each fix before it was applied:
- Issue #5: the root cause (`songs[:-1]` truncating an already-correct list) was identified from
  reading the function directly, no execution tracing needed.
- Issue #1: Claude connected the Sunday-specific failure to `datetime.weekday()` returning 6 for
  Sunday, then confirmed the fix by re-running the reproduction script across all seven weekdays
  plus the other three streak rules, not just the Sunday case.
- Issue #4: Claude compared `add_to_playlist` and `rate_song` line by line (the hint's suggested
  approach) and found the missing `create_notification` call. It also surfaced a related, unlisted
  bug in `add_to_playlist` (a `position`-column `IntegrityError`) while building the test control
  for this fix; I asked for that to be documented rather than silently fixed, since it's outside
  the five listed issues.

**Where I verified or overrode the AI.** I did not blindly accept fixes: I asked for a verification
script after every change (checking all playlists, all seven weekdays, and both the friend-rate and
self-rate cases, not just the originally reported symptom) before accepting a fix as done. I also
directed process decisions myself throughout, e.g. choosing to commit fixes one at a time with my
review in between rather than letting all three be applied at once, and requiring that git commands
be handed to me to run rather than executed automatically.

---

## Codebase Map

Mixtape is a Flask JSON API backed by SQLAlchemy over SQLite. The architecture is a strict
**routes → services → models** layering: routes parse input and format responses, services hold
all business logic (this is where every bug lives), and models define the data.

### Main files and their roles

**`app.py`** — Application factory (`create_app`). Configures the SQLite DB, initializes
SQLAlchemy (`db`), registers four blueprints under URL prefixes, and calls `db.create_all()`.
Start with `FLASK_APP=app:create_app flask run` — **not** `python app.py` (double-import error).
_Note: the `if __name__ == "__main__"` block sets `port=5002`, but that only applies to
`python app.py`; under `flask run` the port comes from the CLI (`--port`)._

**`models.py`** — Six SQLAlchemy models plus three association tables:
- `User` — has `listening_streak` + `last_listened_at` (drives streaks); self-referential
  many-to-many `friends` via the `friendships` table.
- `Song` — shared by a user (`shared_by` FK); many-to-many `tags` via `song_tags`.
- `ListeningEvent` — one row per listen (`user_id`, `song_id`, `listened_at`); feeds streaks + feed.
- `Rating` — a user's 1–5 score for a song, unique per (user, song).
- `Playlist` — many-to-many `songs` via `playlist_entries`, **which carries a `position` column**
  — songs have an explicit order, not just insertion order.
- `Notification` — a message for a `user_id` with a `notification_type` and `body`.

**`routes/`** — Thin Flask blueprints; each endpoint delegates immediately to a service:
- `songs.py` — `GET /songs/search`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`
- `playlists.py` — create playlist, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, add song
- `users.py` — user profile, `GET /users/<id>/streak`, notifications (list + mark read)
- `feed.py` — `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity`

**`services/`** — Business logic (all five seeded bugs live here):
- `streak_service.py` — records listening events and updates the consecutive-day streak.
- `feed_service.py` — "friends listening now" (24h window) and general activity feed.
- `search_service.py` — song search by title/artist, with tags.
- `notification_service.py` — creates/reads notifications; also `add_to_playlist` and `rate_song`.
- `playlist_service.py` — playlist creation and ordered song retrieval.

**`seed_data.py`** — Populates the DB (5 users, 13 songs, 3 playlists, 10 tags). User IDs are UUIDs.

**`tests/`** — pytest stubs: `test_streaks.py`, `test_search.py`, `test_playlists.py`. Run `pytest tests/`.

### Data flow trace — rating a song
`POST /songs/<song_id>/rate` (`routes/songs.py:rate`) parses `user_id` + `score`, then calls
`notification_service.rate_song(user_id, song_id, score)`. That function validates the score range,
looks up an existing `Rating` for the (user, song) pair, and either updates its score or inserts a
new `Rating`, then commits. **Observation for later:** unlike `add_to_playlist` in the same file —
which calls `create_notification()` to alert the song's original sharer — `rate_song` creates no
notification at all. (This is the shape of Issue #4.)

### Data flow trace — viewing a playlist's songs
`GET /playlists/<id>/songs` (`routes/playlists.py:get_songs`) calls
`playlist_service.get_playlist_songs(playlist_id)`, which joins `Song` to `playlist_entries`,
filters to the playlist, and orders by `playlist_entries.position`.

### Patterns noticed
- **Routes never touch the DB directly** (except a trivial `User` lookup in `users.py`); all logic is
  in services. To fix a broken endpoint, trace to its service.
- **Every "write" service commits its own transaction.** Notifications are a side effect triggered
  inside the service that performs the action (e.g. `add_to_playlist` notifies the sharer).
- **Ordering is explicit** via `position` on `playlist_entries`, not insertion order.

---

## Bugs I'm tackling (3 of 5)

Read all five issue descriptions. Original picks were #1, #3, #5. During Milestone 2, **#3
(duplicate search results) could not be reproduced**: the tag join it relies on is deduplicated
by SQLAlchemy's `query(Song).all()`, and no query produced a duplicate row (see Reproduction notes).
Following the plan's "revisit if repro fails" rule, I swapped #3 for **#4**. Final three:

1. **Issue #1, listening streak keeps resetting** (`streak_service.py`): a weekday boundary bug.
4. **Issue #4, no notification when a friend rates your song** (`notification_service.py`): a missing
   side effect, compared against the working playlist-add path.
5. **Issue #5, last song in a playlist never shows** (`playlist_service.py`): an off-by-one on the
   returned list.

---

## Milestone 2: Reproduction

All three were reproduced by calling the service functions directly under an app context against
freshly seeded data (`python seed_data.py`), the approach the brief recommends ("isolate the
function in a Python shell"). No application code was changed. The one in-memory mutation (the
streak test) was rolled back, so nothing was committed.

### Issue #5, last song in a playlist never shows up
- **Condition:** any playlist with N songs.
- **Steps:** query the `playlist_entries` rows for "Late Night Vibes" (7 songs, positions 1 to 7),
  then call `playlist_service.get_playlist_songs(playlist_id)`.
- **Result:** the DB has 7 entries, but the function returned **6**. The song at position 7 was
  missing and the last returned title was "Golden Hour" (position 6). Bug reproduced.
- **HTTP equivalent:** `GET /playlists/<id>/songs` returns `"count": 6` for a 7-song playlist.

### Issue #1, listening streak keeps resetting
- **Condition:** the streak update runs on a **Sunday**, after the user listened the day before.
- **Steps:** take darius (streak 3), set `last_listened_at` to the day before a controlled Sunday
  `now`, and call `streak_service.update_listening_streak(user, now)`. Repeat with a Monday `now`
  as a control.
- **Result:** on **Sunday** the streak went `3 -> 1` (reset) when the rules say it should increment
  to 4. On **Monday** the same setup went `3 -> 4` correctly. The reset is Sunday specific. Bug reproduced.

### Issue #4, notified when a song is added to a playlist but not when it is rated
- **Condition:** a friend interacts with a song the user shared.
- **Steps:** as darius, rate simone's song "Crown Heights Anthem" via
  `notification_service.rate_song(...)` and count simone's notifications before and after. As a
  control, have nova add one of darius's already-in-playlist songs via `add_to_playlist(...)` and
  count darius's notifications.
- **Result:** rating produced **no** notification for the sharer (0 to 0). The control add-to-playlist
  produced **one** (0 to 1). The two parallel actions behave differently. Bug reproduced.
- **Side note found while reproducing:** `add_to_playlist` crashes when the song is *not* already in
  the playlist, because appending to `playlist.songs` does not populate the required `position`
  column. That is a separate latent issue, not one of the five listed, so it is out of scope here.

### Issue #3, duplicate search results (could NOT reproduce, dropped)
- Searched ~10 queries (including songs with 3 tags such as "Crown Heights Anthem"). Every query
  returned unique rows only. `search_service.search_songs` uses `db.session.query(Song).outerjoin(
  song_tags)...all()`, and the ORM collapses the joined rows back to one entity per song, so no
  duplicate appears. Replaced with #4.

---

## Notes: Deviations and Extra Findings

Two things surfaced during investigation that are worth calling out explicitly.

### 1. Swapped Issue #3 for Issue #4
My original three were #1, #3, #5. Issue #3 (duplicate search results) **could not be reproduced**.
`search_service.search_songs` runs `db.session.query(Song).outerjoin(song_tags)...all()`; although
the tag join produces one row per tag, SQLAlchemy's ORM collapses those back to a single `Song`
entity per id, so no duplicate ever reaches the caller. I tried roughly ten queries, including songs
with three tags, and every result set was unique. The project hint says #3 hinges on a "second code
path," and no such branch exists in the current file, so there is nothing to trigger. I replaced it
with **Issue #4**, which reproduced cleanly. Final three fixed: **#1, #4, #5**.

### 2. Extra latent bug found in `add_to_playlist` (out of scope, not fixed)
While building the control for Issue #4, I found that `notification_service.add_to_playlist` crashes
with `IntegrityError: NOT NULL constraint failed: playlist_entries.position` whenever the song is
**not already** in the playlist. The cause is that `playlist.songs.append(song)` inserts a row into
`playlist_entries` without setting the required `position` column, because the ORM relationship has
no way to know the intended position. This is a real bug in the "add a song to a playlist" feature,
but it is **not one of the five listed issues**, so I am leaving it unfixed and only documenting it
here. (It is arguably related in spirit to #5, since both stem from how `position` is handled.)

---

## Root Cause Analyses

### Issue #5: The last song in a playlist never shows up

**How I reproduced it.** Seeded the DB, then compared the raw `playlist_entries` rows for "Late Night
Vibes" (7 rows, positions 1 to 7) against `playlist_service.get_playlist_songs(playlist_id)`. The DB
had 7 entries but the function returned 6, always dropping the highest-position song. Confirmed the
same one-short result on all three seeded playlists.

**How I found the root cause.** The symptom (exactly one missing song, always the last by position)
pointed at list handling rather than the query. I read `get_playlist_songs` in
`services/playlist_service.py`. The query itself was correct: it joins `Song` to `playlist_entries`,
filters to the playlist, and orders by `position` ascending. The final line was the giveaway:
`return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the last element of an
already-correct, fully-ordered list.

**The root cause.** `songs[:-1]` returns every element except the last. Because the list was already
sorted ascending by `position`, "the last element" is always the highest-position song, so that song
was silently excluded from every playlist. The docstring even states "this function returns all songs
in the playlist," which the slice directly contradicts. This was not a query or ordering error; the
data was retrieved correctly and then truncated on the way out.

**My fix and side-effect check.** Changed the return to `return [song.to_dict() for song in songs]`,
removing the `[:-1]` slice. Verified across all playlists that the returned count now equals the
`playlist_entries` count and the order stays ascending by `position`. Checked the empty-playlist edge
case: an empty list returns `[]` correctly (previously a 1-song playlist would have returned `[]`,
dropping its only song). No other function depends on `get_playlist_songs` returning a truncated list.

### Issue #1: My listening streak keeps resetting

**How I reproduced it.** Took darius (streak 3) and set `last_listened_at` to the day before a
controlled `now`, then called `streak_service.update_listening_streak(user, now)`. With `now` on a
**Sunday** the streak reset from 3 to 1; with `now` on a **Monday** (same "listened yesterday" setup)
it correctly incremented to 4. The reset only happened on Sundays.

**How I found the root cause.** The Sunday-only pattern told me a date comparison was treating Sunday
as special. I read `update_listening_streak` in `services/streak_service.py` and looked at the branch
that handles a consecutive day: `elif days_since_last == 1 and today.weekday() != 6:`. The
`days_since_last == 1` part is correct (listened yesterday). The extra `and today.weekday() != 6` is
the problem. I confirmed with Python's docs that `datetime.weekday()` returns 6 for Sunday, so this
condition is false on Sundays.

**The root cause.** `datetime.weekday()` numbers days Monday=0 through Sunday=6. The consecutive-day
branch was gated on `today.weekday() != 6`, so when the current day was Sunday the "listened
yesterday, increment the streak" branch was skipped entirely and execution fell through to the `else`,
which resets the streak to 1. There is no rule in the streak logic that justifies treating Sunday
differently; the guard was simply wrong. Any user whose streak update landed on a Sunday had their
streak wiped even though they had listened on consecutive days.

**My fix and side-effect check.** Removed the `and today.weekday() != 6` guard, leaving
`elif days_since_last == 1:`. Verified the consecutive-day case now increments 3 to 4 on **all seven
weekdays** including Sunday. Re-checked the other three rules (all on a Sunday, the former failure
day): a same-day repeat leaves the streak unchanged, a two-day gap resets to 1, and a first-ever
listen sets it to 1. All still behave correctly, so removing the guard fixed the boundary without
disturbing the other branches.

### Issue #4: Notified when a friend adds my song to a playlist, but not when they rate it

**How I reproduced it.** As darius, rated simone's shared song "Crown Heights Anthem" through
`notification_service.rate_song(...)` and counted simone's notifications before and after: it stayed
at 0. As a control, had nova add one of darius's songs (already in a playlist) via `add_to_playlist`,
and darius's notification count went from 0 to 1. Same shape of action, one notifies and the other
does not.

**How I found the root cause.** The complaint names two parallel actions, so I compared the two
functions in `services/notification_service.py` line by line, exactly as the project hint suggested.
`add_to_playlist` ends with a block: `if song.shared_by != added_by_user_id: create_notification(...)`.
`rate_song` has no equivalent block at all. It validates the score, upserts the `Rating`, commits,
and returns. That absence is the whole bug.

**The root cause.** The root cause is a missing side effect, not a wrong value. `rate_song` never
calls `create_notification`, so rating a song produces no notification for the song's sharer. This is
an architectural inconsistency: the codebase's convention (noted in the codebase map) is that an
action which affects another user's shared song notifies that user from inside the service that
performs the action. `add_to_playlist` follows the convention; `rate_song` was never wired up to it.

**My fix and side-effect check.** After the `db.session.commit()` in `rate_song`, added a block
mirroring `add_to_playlist`: `if song.shared_by != user_id:` create a `song_rated` notification for
`song.shared_by` with a body naming the rater, song title, and score. Verified: a friend rating now
creates exactly one notification with correct content; a user rating **their own** song creates none
(the `shared_by != user_id` guard); the re-rate/update path also notifies; and invalid scores are
still rejected before any notification is created. The re-rate path sends a notification on each
rating, which matches `add_to_playlist`'s existing behavior (neither function deduplicates), so this
is consistent with the established pattern rather than a new inconsistency.

## Screenshot of `git log --online`
![bugfix-mixtape](/ai201-project5-mixtape-starter/bugfix-mixtape.png)
