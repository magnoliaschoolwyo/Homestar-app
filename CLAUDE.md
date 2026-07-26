# CLAUDE.md — HomeStar

This file gives Claude Code the standing context for this project. Read it at the
start of every session. It describes what the app is, the rules for changing it,
the data model, and the current work backlog.

---

## What this project is

**HomeStar** is a family chore/habit/rewards PWA with three tabs: **Celia**,
**Kata**, and **Mama**. The girls check off tasks, earn points, and redeem points
for rewards. Mama has her own tab — a daily habit tracker (morning routine, daily
tasks, a rotating weekly focus, a 7-day streak grid, and a 5-week "Dopamine Reset"
habit program) that doesn't earn or spend points. There is **no time-based
schedule for the girls' tasks** — progress is task-completion only. The app is a
PWA: installable on iPad/Android, works fully offline, and stores all data locally
on the device. There is no backend and no user accounts.

The tone is colorful, playful, and touch-optimized for children.

---

## Hard constraints (do not violate)

- **Plain HTML/CSS/JS only.** No frameworks, no build tools, no bundlers, no npm
  dependencies in the app itself.
- **No backend.** All state lives in `localStorage` on the device.
- **Must keep working offline** and remain installable as a home-screen app.
- Do not introduce anything that requires a network connection at runtime.
  (A Google Calendar sync feature used to violate this — it was removed; see
  "Decisions already made" below. Don't re-add network calls without checking
  with the user first.)

---

## File structure

- `index.html` — **everything**: markup, all CSS (inline `<style>`), and all JS
  (inline `<script>`). There is no separate `app.js` or CSS file — don't assume
  one exists.
- `sw.js` — service worker (offline cache). Has an explicit `ASSETS` precache
  list plus opportunistic runtime caching of any same-origin GET response.
- `manifest.json` — PWA manifest (name, icons, theme, display mode).
- `celia.jpg`, `kata.jpg` — the girls' real photos, used as their avatar circles.
  These used to be embedded as base64 directly in `index.html` (~250KB of text);
  they were extracted to real files so the service worker can cache them
  independently of the HTML (previously, *any* edit to `index.html` forced a
  full re-download/re-cache of both photos). Keep them as separate files.
- `icon-192.png`, `icon-512.png` — PWA install icons, referenced by
  `manifest.json`.

**`index.html` is large (~110KB, ~2,500 lines).** Reading it in full at once will
hit tool size limits — read specific line ranges instead of the whole file.

**Whenever `index.html` changes, bump `CACHE_NAME` in `sw.js`** (e.g.
`homestar-v7` → `homestar-v8`) so devices actually pick up the new version — the
service worker is cache-first.

---

## Data model

### localStorage keys
- `hs_state` — the **only** key. Holds the entire app state as one JSON object
  (see below). There is no separate `hs_week` key — the week is
  `STATE.week` inside `hs_state`.

> Before editing state logic, read the *actual current shape* of `STATE` from
> `index.html`'s `applyStateDefaults()` function — do not assume. Preserve
> backward compatibility so existing saved data on the family's device is never
> wiped. `applyStateDefaults()` is the single source of truth for every key that
> must exist on `STATE`, filled in with `if (!STATE.x) STATE.x = ...` so old
> saved data upgrades safely. When adding a new STATE field, add its default
> there.

### Current STATE shape (as of this writing)
```
week                depositions "A" | "B" — current chore week
weekPeriodStart     dateString of the current Sun–Sat period, used only to
                    detect the automatic Sunday flip (see below). Deliberately
                    NOT given a default in applyStateDefaults() — its absence
                    means "this device hasn't run the auto-flip logic yet,"
                    which lets the first run anchor without an unwanted flip.
celiaPts / kataPts  number — spendable points
pointsRestored_v1   one-time-credit flag from a past incident; leave alone
celiaDone / kataDone  { taskId: true }  — today's checked-off tasks
celiaAvatar / kataAvatar / mamaAvatar   emoji string
bg                  current background theme id (see BG_OPTIONS)
history             array of { who, task, pts, time, date } — capped at 100,
                    newest first (see addHistory())
lastReset           dateString of the last daily reset (girls' + mama's tasks)
celiaSavings / kataSavings   { rewardId: pointsDeposited } — reward progress
celiaBonusTasks / kataBonusTasks   array of { id, name, pts } — parent/kid-added
                    one-off tasks
customRewards       array of { id, icon, name, pts } — user-added rewards
mamaDone            { taskId: true } — Mama's today-checked habit tasks
mamaHistory         array of { date, taskId, done } — capped at 90*20 entries
mamaResetWeek       1–5, which week of the Dopamine Reset program Mama is on
mamaIntention / mamaIntentionDate   today's journaled intention + its date
lastBackupAt        ISO timestamp of the last manual export, or null
```

### Task object formats — there are three separate systems, not one
1. **Base tasks** (`TASKS_CELIA`, `TASKS_KATA`): `{ id, name, pts, cat, week }`.
   `week` is `null` (every day) or `"A"`/`"B"` (only in that chore week).
2. **Day-of-week rotating chores** (`DAILY_CHORES`, keyed `1`–`5` = Mon–Fri):
   `{ id, name, pts, cat }`, shared by both girls, shown in a banner above their
   task list. No entry for a day/weekend means "no special chore today."
3. **Bonus tasks** (`STATE.celiaBonusTasks`/`kataBonusTasks`): free-form,
   parent/kid-typed one-offs: `{ id, name, pts }`. User-typed `name` is rendered
   with `escapeHtml()`/`jsAttr()` — see "XSS-safe rendering" below.

`cat` (base tasks + daily chores only) is one of the category keys below.

### Reward format
Built-in rewards live in `SHARED_REWARDS`, `CELIA_REWARDS`, `KATA_REWARDS`:
`{ id, icon, name, pts }`. User-added ones live in `STATE.customRewards`, same
shape. All reward cards use a **dual-progress deposit/redeem** mechanic: kids
"deposit" points from their spendable total into `celiaSavings`/`kataSavings`
for a specific reward, and can redeem once fully saved. This replaced a simpler
"redeem outright" model — don't reintroduce a single flat points check without
checking whether that mechanic is still wanted.

### Category colors
- **Chores** — amber / orange
- **Academic** — blue
- **Exercise** — green
- **Music** — purple

### XSS-safe rendering
Any user-typed string (bonus task names, custom reward names, Mama's daily
intention) **must** go through `escapeHtml()` when placed in `innerHTML`, and
`jsAttr()` when placed inside an `onclick="...('...')"` attribute. This isn't
optional stylistic advice — a name with an apostrophe (e.g. "Mom's Day Out")
previously broke its own button because it wasn't escaped. If you add a new
render path for user-typed text, use these helpers.

---

## Task list (current, as coded — not aspirational)

**Chores (both girls, every day):** Unload dishwasher (1), Hygiene (1),
15-min bedroom reset (1)

**Chores (Week A/B alternating, swapped between the girls):** Collect eggs (1),
Compost (1) / Dog poop (1), Living room pick-up (1)

**Day-of-week rotating chore banner (both girls, not week-based):**
Mon: Laundry (3) · Tue: Chickens (3) · Wed: Bathrooms (2) + Upstairs common area
(2) · Thu: Cat boxes (2) · Fri: Game together (3) · Weekends: none

**Academics (both girls):** Scripture & Quiet Time (5), Math (3), Read 20 min
(3), Grammar (1), History/Science w/ Mom (2), Book study w/ Mom (2) — **none of
these alternate by week**, including History/Science (explicitly decided
against — see below)

**Academics (girl-specific):** Always Ice Cream — *Celia only* (3), Monarch —
*Kata only* (3)

**Exercise (both girls):** 20 min exercise (4), 20 min outside play (1)

**Music (both girls):** Music practice (4)

**Bonus tasks:** free-form, added per-girl from the UI, not hardcoded.

### Rewards
**Shared:** Day without chores (50), Dessert at the park (75), JACZ Ice cream
(100), Hike and picnic (150), Playdate at home (200), Pool w/ friends (500),
Debit card (1000)

**Celia only:** Movie night pick (50), Trip to the Secret Fort (125), Special
outing (150)

**Kata only:** Movie night pick (50), Special outing (150)

**Custom rewards:** user-added from the Rewards tab, stored in
`STATE.customRewards`.

---

## Design principles

- Kid-friendly, colorful, touch-optimized
- Each girl (and Mama) has her own tab / view
- Checking a task off is instant — celebration animation + confetti fire immediately
- **Unchecking** an already-completed task (or bonus task) requires confirmation
  first, via `uncheck-modal` — it silently removes earned points, so an accidental
  tap shouldn't cost points with no warning. This does *not* apply to Mama's habit
  checkboxes (no points at stake there).
- Reward cards show a per-girl deposit progress bar, with a pulsing **Redeem**
  button once fully saved
- Redeeming a reward also requires confirmation (`redeem-modal`)
- Points persist to `localStorage` on the device

---

## How to work in this repo (working agreements)

- Because `index.html` is one large file, **don't paste the entire file into
  chat** — deliver updated files via the file-send mechanism and describe the
  change in prose/diff form instead.
- After each change, **state which file(s) need to be committed**, and give the
  exact `git add`/`commit`/`push` commands (there was no git repo in this folder
  as of this writing — confirm whether one exists before assuming `git` commands
  will work).
- If a change touches `index.html`, **always bump the `sw.js` cache version.**
  Note whether `manifest.json` needs updating too (it rarely does).
- Keep everything plain HTML/CSS/JS. No frameworks, no build step.
- New base tasks follow `{ id, name, pts, cat, week }`; new rewards follow
  `{ id, icon, name, pts }` in the correct girl's array, `SHARED_REWARDS`, or
  let the user add them as a custom reward via the UI instead.
- Preserve existing saved data (`hs_state`) — never ship a change that silently
  wipes progress. Add new fields via `applyStateDefaults()`, not by assuming
  they already exist.
- **Test in the browser before calling something done.** Serve the folder with
  a local static server (e.g. `python3 -m http.server`) and drive it with the
  browser tool — don't just eyeball the diff. This project has caught real bugs
  (duplicate function declarations, unescaped attribute injection, wrong avatar
  targets) that only showed up when actually clicked through.

---

## Deployment

- **GitHub repo:** <ADD REPO URL>
- **Netlify URL:** <ADD NETLIFY URL>
- **Flow:** commit to GitHub → Netlify auto-deploys within ~30 seconds.

After making changes, remind the user exactly what to commit and push.

---

## Decisions already made (don't re-litigate without asking)

- **History/Science and Book study do NOT alternate by week.** This was
  explicitly requested — leave them as flat daily tasks even though the rest of
  `CLAUDE.md`'s spirit might suggest otherwise.
- **The Google Calendar sync feature was removed entirely** (Settings section,
  Dashboard panel, all fetch/parse/proxy JS). It sent the family's private iCal
  URL — effectively a bearer credential — to a public third-party CORS proxy
  (`api.allorigins.win`) every 15 minutes. If calendar integration comes up
  again, the two safer paths discussed were: (a) point at a separate, genuinely
  public Google Calendar containing only what should be shown, or (b) self-host
  the proxy via a Netlify Function (a deliberate, scoped exception to "no
  backend" — flag it as such, don't silently reintroduce a backend).
- **Photos are real, separate JPEG files, not embedded base64.** Don't inline
  them back into `index.html`.
- Week A/B now **auto-flips every Sunday** (`checkWeekAutoFlip()`), with a
  manual "Override" button kept in Settings. The first run after this feature
  shipped anchors to the current week without flipping (see `weekPeriodStart`
  above) — don't remove that anchor-on-first-run behavior, it's what prevents
  existing devices from jumping to the wrong week on update.

---

## Known open items / minor rough edges

- The modal-backdrop-click-to-close handler (near the bottom of the script)
  lists `avatar-modal`, `deposit-modal`, `redeem-modal`, `reset-modal` but not
  `uncheck-modal` — tapping outside `uncheck-modal` won't dismiss it (its
  Cancel button still works fine). Low priority, hasn't been fixed.
- No automated tests exist. Manual browser verification (see working
  agreements above) is the only safety net.

### Verify before presenting
- Install-to-home-screen works on iPad (manifest icons correct).
- Offline behavior works (service worker cache is doing its job).
- Celebration animation and Redeem pulse both feel satisfying.

---

## Future features (not built yet — do not start unless asked)

- Custom avatars chosen by the girls (currently they have fixed real photos;
  only Mama has a pickable emoji avatar)
- Streak / history tracking for the girls (Mama already has a 7-day habit
  streak grid; the girls don't have an equivalent)
- Push notifications for incomplete tasks
