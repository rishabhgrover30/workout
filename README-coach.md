# Workout: coach update guide

Live: https://rishabhgrover30.github.io/workout/ · Repo: rishabhgrover30/workout (Pages: main, root)

The app reads `plan.json` on every open, on "Refresh plan", and whenever it comes back to the foreground. It adds `?t=Date.now()` and `cache: no-store`, so a push shows up as soon as Pages redeploys (about 30–60 s). Nothing else needs to change. `index.html` is generic.

## Quick update (after each session)
```bash
cd /workspace/workout-site
git pull --rebase -q                      # in case the repo changed elsewhere
# edit plan.json (see below), then validate:
python3 -m json.tool plan.json > /dev/null && echo OK
git add plan.json
git commit -m "Log 2026-10-17 Day 1; next: day2"
git push
# optional: confirm the live copy (wait about a minute)
curl -s "https://rishabhgrover30.github.io/workout/plan.json?t=$(date +%s)" | python3 -c "import json,sys;d=json.load(sys.stdin);print(d['updated'],d['nextSessionId'])"
```
Always run the JSON check before pushing. An invalid file makes the app show an error, or the last cached plan if it has one.

## What to edit in plan.json
- `updated`: ISO timestamp, e.g. `2026-10-19T19:30:00+01:00` (shown in the Program tab).
- `nextSessionId`: `day1`–`day4`. This is the session the Today tab opens by default; a dot marks it in the picker.
- `sessions[].exercises[]`, per exercise:
  - `target`: next session's weight or instruction, e.g. `"67.5 kg"`.
  - `lastTime`: e.g. `"65 kg × 6, 6, 5"`.
  - `notes`: coaching cue.
  - `sets` (a number) and `reps` (a string like `"4–6"`). Changing `sets` changes the number of tick rows.
  - `short` is the label used in the copied summary (e.g. "Bench").
- `log[]`: add one object per session; the order doesn't matter (newest is shown first):
  ```json
  { "date": "2026-10-19", "sessionId": "day2",
    "entries": [ { "exercise": "Back squat", "sets": ["100×6", "100×5", "100×5", "100×4"] } ],
    "coachNote": "Solid. Squat stays at 100 until 4×6." }
  ```
  `"inProgress": true` shows an "in progress" tag; remove it when the session is done.
  A log entry dated today, with `entries[].exercise` matching an exercise `name` or `short`, pre-ticks those sets on the phone. This only applies if he hasn't already ticked anything on the phone today; local ticks win.
- `weekly`: `{ "goal": 4, "done": n, "weekOf": "YYYY-MM-DD" }`, where weekOf is that week's Monday. Reset `done` to 0 and bump `weekOf` each Monday.
- `notes[]`: the rules list in the Program tab.
- `wedding`: drives the "N days to the wedding" line in History.

## What the summary he pastes looks like
```
Day 1 · Upper Strength, Sat 17 Oct
Bench 65×6, 67.5×6, 67.5×5; Pull-ups +10×6, 5; Cable laterals ✓
Note: bench felt easy
```
`✓` means he ticked the set without typing numbers. Use the session's `target` as the assumed load.

## Notes
- Phone-side ticks are stored in localStorage per date and session (`workout.v1.YYYY-MM-DD.dayN`); the repo never sees them.
- Keep `shot-*.png` out of the repo (it's gitignored).

## Live sync (Supabase: Meal Tracker + Workout)
Both apps POST to a private Supabase project (URL and keys only in the secrets file) through two RPC functions:
`POST {SUPABASE_URL}/rest/v1/rpc/sync_meal` `{p_key, p_date, p_snapshot}` and
`POST {SUPABASE_URL}/rest/v1/rpc/sync_workout` `{p_key, p_date, p_session, p_snapshot}`, headers `apikey` + `Authorization: Bearer <anon key>`.
- Workout: every tick, untick, kg/reps edit or note change sends today's full session about 1.5 s later → table `workout_sessions` (one row per date + session; `sets_done`, `sets_total`, `started_at`, `updated_at`, full `snapshot` with every set).
- Meal: each change sends that whole day about 3 s later → table `meal_days` (one row per date; `totals`, full `snapshot` with items + targets).
- Failed sends stay queued on the phone and are retried on the next change, when the app returns to the foreground, when the phone comes back online, and on load. A 403 shows "Sync key rejected"; a 400 (malformed day) is dropped so it can't block the queue.

### Security model
- Tables have RLS on with **no policies** and no grants for `anon`/`authenticated`: the public anon key can't read or write them.
- The only thing anon can do is call `sync_meal` / `sync_workout` (SECURITY DEFINER). Each checks `p_key` against a sha256 hash in
  `private.sync_config` (schema not exposed by the API), rejects snapshots over 200 KB, odd dates and bad session ids, then upserts.
- Only the service key (box only) can read. Schema + checks: `/workspace/supabase-setup/schema.sql` (template), `setup.sql` (with the hash), `e2e.sh`.
- Rotate the sync key: new token → update `SYNC_TOKEN` in the secrets file → re-run `setup.sql` with the new hash → send new setup links.

### Secrets: never in the repos
Both repos are public. `SYNC_URL` stays `''` in code (`var SYNC_URL = '';` in workout `index.html` and fuel-tracker `src/app.js`).
The project URL, anon key and sync key exist only:
- on the phone, in localStorage (`workout.v1.sync` = {url, anon, key}; `mealtracker.v1.settings`.syncUrl/.syncAnon/.syncKey)
- on the box, in **/workspace/.supabase-secrets** (chmod 600: SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_KEY, SUPABASE_DB_PASSWORD, SUPABASE_REF, SYNC_TOKEN)
  and the ready-made setup links in **/workspace/.supabase-links** (chmod 600).

### Setting it on his phone (one link per app)
Format: `https://rishabhgrover30.github.io/<app>/#sync=<supabase url>&anon=<anon key>&key=<sync key>` (each value URL-encoded).
The hash is never sent to GitHub; the app saves it and clears it with history.replaceState.
Send these only in the private chat. A link opened from chat lands in Safari, and the home-screen apps have separate storage,
so tell him to **copy** each link and **paste it into the app's sync box** (Meal Tracker: Settings > Live sync, first box;
Workout: Program tab, bottom, Live sync, first box) and tap **Save**. The status next to the app name should read "Sync on", then "Synced HH:MM".

### Reading live progress
```bash
/workspace/fitness-read.sh workout            # today's sessions (Europe/London date), newest first
/workspace/fitness-read.sh workout 2026-10-17
/workspace/fitness-read.sh meal               # today's meals: totals + items + targets
/workspace/fitness-read.sh meal yesterday
```
Prints compact JSON `{app, date, rows:[…]}`. Workout row: `{date, session_id, sets_done, sets_total, started_at, updated_at, ago_min, snapshot:{sessionName, note, sets:[{exercise, set, kg, reps, done, time}]}}`.
Meal row: `{date, updated_at, ago_min, totals:{kcal,p,c,f,fib}, snapshot:{targets, items:[{time, meal, food, qty, unit, kcal, p, c, f, fib}]}}`.
An empty kg means bodyweight. `time` is the phone's local time; `updated_at` is UTC-offset ISO.
When he says he's started, poll every few minutes and use `ago_min` to see whether he's still moving.
Then log the session into `plan.json` as usual: the database is the live feed, and plan.json stays the source of truth.

`apps-script/Code.gs` is the retired Google Sheets version, kept for reference only; nothing calls it.
