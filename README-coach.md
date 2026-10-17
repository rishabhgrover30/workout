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

## Live sync to Google Sheets (Meal Tracker + Workout, one Apps Script)
Both apps POST to one Apps Script web app bound to the sheet "Meal Tracker Log"
(1D0KjPb91cLB2mimfgy3Vg28UJx-kYKaRzYb4AjUheGU). Code: `apps-script/Code.gs`, identical in both repos.
The user's steps are in `apps-script/DEPLOY.md`.
- Workout: every tick, untick, kg/reps edit or note change sends today's full session about 1.5 s later.
  It goes to the **Workout** tab (one row per set) and **WorkoutSessions** (one row per session: done/total, started, last update, note).
- Meal: each change sends that whole day about 3 s later, to **Items** and **Daily**.
- Failed sends stay queued on the phone, and are retried on the next change, when the app returns to the foreground, when the phone comes back online, and on load.

### Secrets: never in the repos
Both repos are public. `SYNC_URL` stays `''` in code (`var SYNC_URL = '';` in workout `index.html`
and fuel-tracker `src/app.js`). The real URL and key exist only:
- on the phone, in localStorage (`workout.v1.sync` = {url, key}; `mealtracker.v1.settings`.syncUrl/.syncKey)
- on the box, in **/workspace/.sync-secrets** (chmod 600, outside both repos, also gitignored):
  ```
  SYNC_URL='https://script.google.com/macros/s/…/exec'
  SYNC_KEY='…'
  ```
Every request needs the key (POST body `key`, GET `?key=`). Without it, the response is just `{"ok":false,"error":"unauthorized"}`.

### Setting it on his phone (one link per app)
The hash is never sent to the server; the app saves it and clears it with history.replaceState.
```bash
source /workspace/.sync-secrets
python3 - <<PY
import urllib.parse as u; s=u.quote("$SYNC_URL", safe=""); k=u.quote("$SYNC_KEY", safe="")
print("Meal Tracker: https://rishabhgrover30.github.io/meal-tracker/#sync=%s&key=%s" % (s, k))
print("Workout:      https://rishabhgrover30.github.io/workout/#sync=%s&key=%s" % (s, k))
PY
```
Send these only in the private chat. A link opened from chat lands in Safari. The home-screen apps
have separate storage, so tell him to copy each link and paste it into the app's sync box
(Meal Tracker: Settings > Google Sheet sync; Workout: Program tab, bottom) and tap Save.

### Reading live progress (plain curl, no Google auth)
Apps Script answers with a 302 redirect, so `-L` is required. Use the UK date (he's in Europe/London).
```bash
source /workspace/.sync-secrets; D=$(TZ=Europe/London date +%F)
curl -sL "$SYNC_URL?key=$SYNC_KEY"                                 # health: {"ok":true}
curl -sL "$SYNC_URL?key=$SYNC_KEY&app=workout&date=$D" | python3 -m json.tool   # today's sessions + every set
curl -sL "$SYNC_URL?key=$SYNC_KEY&app=workout&date=$D&session=day1"             # one session
curl -sL "$SYNC_URL?key=$SYNC_KEY&date=$D" | python3 -m json.tool               # meals: items[] + daily totals/targets
```
Workout response: `{ok, app:"workout", date, sessions:[{date, session, "session name", "sets done", "sets total", started, "last update", note, sets:[{exercise, set, kg, reps, done, time}]}]}`.
An empty kg means bodyweight. Times are the phone's local time.
When he says he's started, poll every few minutes and compare `last update` to the clock to see whether he's still moving.
Then log the session into `plan.json` as usual: the sheet is the live feed, and plan.json stays the source of truth.

If Code.gs changes, he must redeploy (Manage deployments > New version), or the old code keeps running.
