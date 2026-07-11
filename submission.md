# Mixtape Bug Hunt — Submission

Branch: `bugfix/mixtape`
Bugs fixed: **#5 (last playlist song), #4 (rating notification), #1 (Sunday streak reset)**

---

## AI Usage

I used an AI assistant (Claude) as a navigation and verification partner, not as a bug oracle. Specifically:

- **Codebase orientation.** I had the AI summarize each `services/` module's responsibility and trace one full call chain (route → service → model) so I could write the codebase map below with real function names instead of guesses. I verified every claim by opening the file myself.
- **Understanding suspicious code.** For the streak bug I asked what Python's `datetime.weekday()` returns for each day, then confirmed it in a REPL (`Sunday == 6`) rather than trusting the answer.
- **Where AI was wrong / incomplete — Issue #3.** My original plan was to fix a duplicate-search bug (#3). The AI and I both expected `GET /songs/search?q=Anthem` to return a 3-tag song three times. When I actually ran it, it returned the song **once**. Investigating the raw SQL proved the `outerjoin` really does fan out to 3 rows at the database level, but SQLAlchemy's *legacy* `db.session.query(...).all()` auto-deduplicates entities by primary key, silently masking the symptom on this version. Because the reported behavior can't be reproduced through the endpoint on the installed stack, I swapped #3 for #1. This is the clearest example of why "reproduce before you fix" matters: the plausible explanation was structurally correct but did not match observable behavior.
- **What I verified myself.** Every fix was confirmed by (a) a before/after reproduction script that calls the real service functions and (b) the existing `pytest` suite (13 passed).

---

## Codebase Map

### Main files and their roles

| File | Role |
|------|------|
| `app.py` | Flask application factory (`create_app`). Configures SQLite, initializes `db`, registers the four blueprints under `/songs`, `/playlists`, `/users`, `/feed`. |
| `models.py` | SQLAlchemy models. **7 models** — `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification` — plus **3 association tables** defined as raw `db.Table`: `friendships`, `song_tags`, and `playlist_entries`. |
| `routes/*.py` | Thin HTTP layer. Each route parses `request`, calls exactly one service function, and wraps the result in `jsonify`. No business logic. |
| `services/*.py` | All business logic. One service per feature: `streak_service`, `feed_service`, `search_service`, `notification_service`, `playlist_service`. **Every bug in this project lives here.** |
| `seed_data.py` | Rebuilds the DB with 5 users, 13 songs (with 0 / 1 / 3+ tags), 3 playlists, listening events, and one example playlist-add notification. |
| `tests/` | `pytest` suites for streaks, search, and playlists. Two tests (`test_streak_increments_on_sunday`, `test_playlist_returns_all_songs`) are effectively regression tests for bugs #1 and #5. |

### Key data-model facts (these explain the bugs)

- **`playlist_entries` is not a plain join table.** Beyond the two foreign keys it carries `position` (integer), `added_by`, and `added_at`. A playlist is therefore an *ordered* list — song order is explicit, not insertion order.
- **A `Rating` is its own table**, not a column on `Song`. It has a `UniqueConstraint(user_id, song_id)` so a user can rate a song at most once.
- **A `Notification` is a plain text row** with no foreign key to whatever caused it. It is not derived by the database — some service function must *explicitly write it*. (This is the root of Issue #4.)
- **`song_tags` has no uniqueness beyond its composite PK**, so a song with 3 tags has 3 rows — relevant to the search fan-out.

### Data flow traced end-to-end: "a friend rates your song → you get a notification"

```
POST /songs/<song_id>/rate   {user_id, score}
  └─ routes/songs.py  rate()                          # parse body, cast score to int
       └─ notification_service.rate_song(user, song, score)
            ├─ validate score 1–5, load Song + rater
            ├─ upsert Rating (unique per user+song)
            ├─ db.session.commit()
            └─ if song.shared_by != user_id:          # <-- the fix for #4
                 create_notification(user_id=song.shared_by,
                                     type="song_rated", body=...)
                   └─ Notification row committed
GET /users/<id>/notifications
  └─ users.py notifications() → get_notifications()   # newest-first list
```

### Pattern noticed

Every route delegates immediately to a single service function; routes do input parsing and JSON formatting only. Services own all logic and all `db.session.commit()` calls. So to debug any endpoint you follow exactly one hop: route → the service function it names.

---

## Root Cause Analysis

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it.** Seeded the DB and called `get_playlist_songs()` on the "Friday Energy" playlist. The `playlist_entries` table has **7** rows for that playlist, but the function returned **6** song dicts. Re-checking against the join table confirmed the missing one was always the highest `position` (most recently added).

**How I found the root cause.** The route `GET /playlists/<id>/songs` → `playlist_service.get_playlist_songs()`. I read the function top to bottom. The query is correct — it joins `playlist_entries`, filters by playlist, and orders ascending by `position`. The bug is in the very last line, the return statement.

**The root cause.** The function ended with:
```python
return [song.to_dict() for song in songs[:-1]]
```
`songs[:-1]` is a Python slice that returns every element *except the last*. Because the query orders ascending by `position`, the last element is always the most recently added song, so exactly one song — the newest — is dropped on every call. The docstring even states "this function returns all songs," which the code contradicts.

**My fix and side-effect check.** Removed the slice: `return [song.to_dict() for song in songs]`. Re-ran the reproduction: 7 rows in, 7 songs out. The repo's existing `test_playlist_returns_all_songs` (which asserts 5 and whose comment reads "Bug causes this to return 4") now passes, as does `test_playlist_returns_songs_in_order` (ordering unaffected) and `test_empty_playlist_returns_empty_list` (an empty list `[]` is unchanged by removing `[:-1]`).

---

### Issue #4 — Rated songs never generate a notification

**How I reproduced it.** As `kenji`, called `rate_song()` on a song shared by `simone`, checking her notification count before and after. Before: 0. After: **still 0** — the `Rating` row was written (the score persisted) but no `Notification` appeared. By contrast, `add_to_playlist()` for the same sharer produced a notification immediately.

**How I found the root cause.** `POST /songs/<id>/rate` → `notification_service.rate_song()`. Since the *playlist-add* notification worked, I put the two functions side by side in the same file. `add_to_playlist()` ends with an `if song.shared_by != added_by_user_id: create_notification(...)` block. `rate_song()` ended right after `db.session.commit()` with `return rating` — there was no `create_notification` call anywhere in it. That side-by-side comparison was the moment it was certain: the bug is a missing step, not a broken one.

**The root cause.** Notifications are not derived by the database; they are explicit rows a service must write (see the model note above). `rate_song()` correctly saves the rating but never calls `create_notification()`, so no notification is ever created for the sharer. This is architectural — the code that should exist is simply absent.

**My fix and side-effect check.** After the commit in `rate_song()`, I added the same guarded notification the playlist path uses:
```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score} stars.",
    )
```
Re-ran the reproduction: sharer notifications went 0 → 1. Checked side effects: the `song.shared_by != user_id` guard means rating your own song still produces no notification (matching the playlist behavior); the rating upsert path (updating an existing rating) is unchanged; `get_notifications` ordering is unaffected. Full test suite still 13 passed.

---

### Issue #1 — Listening streak resets every Sunday

**How I reproduced it.** Called `update_listening_streak(user, now)` directly with controlled datetimes: `last_listened_at` = Saturday, `now` = the following Sunday, starting streak 12. Result: streak dropped to **1** instead of 13. The same setup with Monday→Tuesday correctly produced 6, proving the failure was specific to Sunday.

**How I found the root cause.** `GET /users/<id>/streak` reads the stored value; the value is written in `streak_service.update_listening_streak()`. Reading the branch that handles "listened yesterday":
```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```
I confirmed in a REPL that `datetime.weekday()` returns **6 for Sunday** (Mon=0 … Sun=6).

**The root cause.** The `elif` requires *both* `days_since_last == 1` **and** `today.weekday() != 6`. On a Sunday, `today.weekday()` is `6`, so `today.weekday() != 6` is `False`, the whole `elif` fails, and execution falls into the `else`, which resets the streak to 1 — even though the user listened on consecutive days. Consecutive days are consecutive regardless of weekday, so the `weekday()` guard had no legitimate purpose; it only ever fired on Sundays and always incorrectly.

**My fix and side-effect check.** Deleted the spurious clause so the branch is simply `elif days_since_last == 1:`. Re-ran the boundary matrix: Sat→Sun now increments 12 → 13; Mon→Tue still 6; two listens same day still no change; a 2-day gap still resets to 1. The repo's existing `test_streak_increments_on_sunday` (asserts 2 after Sat→Sun) now passes, along with all other streak tests — 13 passed total.

---

## Regression Tests (stretch)

The repository already contained tests that assert the *correct* behavior for two of these bugs and therefore failed on the buggy code and pass after my fixes:

- `tests/test_playlists.py::test_playlist_returns_all_songs` — guards Issue #5 (comment: "Bug causes this to return 4").
- `tests/test_streaks.py::test_streak_increments_on_sunday` — guards Issue #1.

Full suite: `pytest tests/` → **13 passed**.

---

## Commit History

_(Paste `git log --oneline` screenshot of the `bugfix/mixtape` branch here.)_
