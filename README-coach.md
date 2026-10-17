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
