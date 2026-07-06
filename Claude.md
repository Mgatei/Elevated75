# CLAUDE.md — Elevated (75-Day Couples Transformation Tracker)

Guidance for Claude Code (or any AI/dev) working on this project.

---

## 1. What this project is

A **single self-contained HTML file** that is a faith-centered daily habit tracker
for a married couple over a 75-day journey.

- **Main file:** `index.html` (identical copy kept as `75-days-luxury-tracker.html`)
- **Size:** ~8,050 lines / ~421 KB. All CSS, JavaScript, SVG avatars, and Firebase
  config live INSIDE this one file. There are no build steps, no bundler, no
  external source files it depends on.
- **The two `.png` files** are avatar previews for reference only — the app does
  not load them.
- **Hosting:** Netlify (drag-and-drop to the same URL preserves the live URL).
- **Users:** two accounts — `mahdi` and `lashawn` (a married couple).

## 2. How to run / test locally

- Just open `index.html` in a browser. Everything works offline except cloud sync
  (which needs internet + Firebase).
- There is no test framework. Validation is done by static checks (see §7).

## 3. Data & persistence model

- One global `data` object holds BOTH users: `data.mahdi` and `data.lashawn`.
- Persisted to browser `localStorage` under the key **`75data`**.
- Each user object contains: `days` (dateKey -> {itemKey: bool, _water: n}),
  `goals`, `entries` (notes, keyed `dateKey|itemKey` -> [{text,time}]), `photos`
  (on-device only), `emotions`, `checkins`, `archives`, `weekFlags`, `wheel`,
  `bible` (dateKey -> {book,chapter,verse}), `locked` (dateKey -> true, whole-day
  lock), `sectionLocks` (`dateKey|Section` -> true, per-section lock), `layout`
  ({order:[], hidden:[]}, custom section order + hide), `startDate`, `themeMode`,
  and `lastModified` (timestamp — critical for sync, see §5).
- **Checkbox id / data key format:** `` `${pillarName}-${itemName}` `` e.g.
  `Faith-Prayer`, `Nourish-Water`. Water key is `Nourish-Water`.

## 4. Cloud sync (Firebase / Firestore)

- Firebase project **`elevated-75-tracker`**; compat SDK v10.12.2 loaded in the head.
- Collection **`journeys`**, two docs: `journeys/mahdi` and `journeys/lashawn`.
  Each doc = `{ payload: JSON(data[user] WITHOUT photos), writeTs, fromDevice, updated }`.
- `initFirebase()` runs on `window.onload` (must NOT run in <head> — load-order).
- **Photos are never cloud-synced** — they stay on each device.
- **Live auto-pull listener is intentionally DISABLED** (`startCloudListener()` is a
  no-op; body kept as `startCloudListener_DISABLED`) — it caused disruptive
  re-renders. Sync is done via explicit pulls instead (`cloudSyncAll`).

## 5. The sync "golden rule" (do NOT break this)

Data loss was caused by cloud pulls overwriting fresher local data on reload.
The fix, which MUST be preserved:

- `save()` stamps `data[currentUser].lastModified = Date.now()`.
- `cloudSaveSilent()` sends that as the doc's `writeTs`.
- `cloudSyncAll()` only replaces a user's local data if
  **`cloudTs > localTs`** (cloud genuinely newer). If local is newer or equal,
  KEEP LOCAL and (for current user) push local up.
- Any new cloud-load path must apply this same rule, or data loss returns.

## 6. Daily screen structure

Render order (top to bottom): verse/wisdom of day (static) → **Nourish** (compact
pinned widget) → **Spiritual Spin** wheel card → **Faith** → rest of sections.

Sections ("pillars") per user, in `pillars` object:
- **Mahdi:** Nourish [Water, Healthy Choices] · Faith [Prayer, Faith and Mind,
  Journal, Church] · Physical [Physical Activity] · Professional [Job Apps,
  Prof Dev] · Creativity [Music Work, Brand Work] · Love [Love and Connection]
- **Lashawn:** Nourish · Faith · Physical · Professional [Career Growth, Prof Dev]
  · Content [Content, Engagement] · Wellness [Me Time, Self-Care, Boundaries,
  Inner Beauty] · Love [Love and Connection]

Behaviors:
- Sections are slim, always-open boxes; **auto-collapse when all items complete**;
  tap header to reopen; each has a **per-section Save** (locks just that section).
- Items collapse to one line when checked; notes are OPTIONAL; tap a collapsed
  item to reopen and add a note.
- **Big Save** at the bottom locks the WHOLE day into read-only review (force-expands
  everything) with an Unlock button.
- **Customize Layout** (⚙ More menu) lets each user reorder + hide sections; the
  daily view uses `getOrderedPillars(user)`, never the raw `pillars` array. The raw
  array still drives stats so hidden sections' data is never lost.

## 7. REQUIRED validation after EVERY edit

Because it's one giant file, always run these before considering an edit done:

```
cd <folder>
node -e "
const fs=require('fs');const html=fs.readFileSync('index.html','utf8');
const js=[...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].pop()[1];
try{new Function(js);console.log('JS OK');}catch(e){console.log('JS ERROR:',e.message);}
const b=html.indexOf('<body>');const s=html.indexOf('<script>',b);
const body=html.slice(b,s);
const o=(body.match(/<div/g)||[]).length,c=(body.match(/<\/div>/g)||[]).length;
console.log('divs',o===c?'BALANCED':'MISMATCH '+o+'/'+c);
"
```

- Div balance is checked on the STATIC body only (before the first `<script>`),
  because `<div` inside JS strings causes false mismatches.
- For avatar edits: render the SVG to PNG (cairosvg) and visually inspect BEFORE
  shipping — this caught many bad avatars.
- **Do NOT use `&` inside JS string literals in inline `<script>`** — the HTML parser
  turns it into `&amp;` and breaks parsing. Use "and" (e.g. "Faith and Mind").

## 8. Adding a new data key (migration gotcha)

New per-user keys must be initialized in ALL of these spots or they'll be undefined
somewhere: the main load migration, the cloud-listener migration, `cloudLoad()`,
`importData()`, and `ensureUserKeys()`. Grep for an existing key like
`data[u].bible` to find every spot (there are ~5–6).

## 9. Key functions (where to look)

- `render()` — builds the daily view; uses `getOrderedPillars(currentUser)`.
- `loadToday()` — hydrates checkboxes/notes, applies collapse + lock states.
- `cloudSyncAll(cb)` / `cloudSaveSilent()` / `pushBothToCloud(cb)` — sync.
- `save()` — writes local + stamps lastModified + schedules cloud save.
- `renderBibleStudy()` / `bibleCommentary()` / `blbUrl()` — Bible tool.
- `companionAvatarSVG(user,size)` — returns inline SVG for Eli/Nia.
- `getOrderedPillars(u)` / `getLayoutPref(u)` — custom layout.
- `saveSection/unlockSection/toggleSection/applySectionStates` — per-section save.
- `lockDayReview/unlockDay/applyDayLockReview` — whole-day lock.
- `exportData/importData` — combined backup (both accounts in one file).

## 10. Known dead / legacy code (safe to clean up, currently harmless)

- `togglePillarCollapse()` + `collapsedPillars` — from the old collapsible-section
  design; superseded by `toggleSection`/section-collapse. No longer called.
- `cycleSpouseTip()` + `spouseTipText` — leftover from the pre-merge "Time with
  Lashawn" block; the merged "Love and Connection" block doesn't use it.
- `startCloudListener_DISABLED()` — kept intentionally as reference; not called.
- The old `if (item === 'Progress Photo')` render block — never fires now that
  Progress Photo is an optional camera button inside Physical Activity.

## 11. Release checklist (tell the user every time)

1. In the live app: ⚙ More → **⬇ Backup** first (combined file, both accounts).
2. Re-upload to the **same Netlify URL**.
3. Update **all devices** (both phones + computer) to the same version.
4. Photos are NOT in backups/cloud — they live per-device.

## 12. Git / version-control discipline (do NOT skip)

Because this is a one-file app with no tests, git history is the safety net. Follow
these rules on every session:

- **Commit before any risky or large edit** (rewriting a big function, restructuring
  sections, bulk find/replace across the file). A commit right before is a free
  undo button — make one even if the user didn't ask.
- **Small, frequent commits** with clear messages, not one giant commit at the end.
  Each commit should be a working state (passes the §7 validation checks).
- **Never use destructive git commands** (`git reset --hard`, `git checkout -- .`,
  `git clean -f`, `git branch -D`, force-push) unless the user explicitly asks for
  that specific action in that moment. Prior approval does not carry forward to
  future destructive commands.
- **Never delete data-bearing files** (avatars, backups, exported JSON) without
  explicit confirmation — move them aside instead if unsure whether they're needed.
- **Always work on a branch, never commit straight to `main`** unless the user says
  otherwise. Push with `git push -u origin <branch-name>`.
- **Before replacing `index.html` wholesale** (e.g. pasting in a fully regenerated
  file), commit the current version first so the previous state is recoverable.
- If something looks like in-progress or uncommitted work when a session starts,
  investigate before touching it — don't assume it's safe to overwrite.
- User is new to git. explain git concepts and terminology where possible


## 13. Git concepts glossary (for anyone new to GitHub)

Plain-language explanations of the terms that come up when working on this repo,
written for someone who hasn't used git/GitHub before.

- **Repository ("repo")**: the project's folder, plus its full history of changes.
  This repo is `Mgatei/Elevated75` on GitHub.
- **Commit**: a saved snapshot of changes, with a message describing what changed.
  Commits are the "undo points" — you can always go back to any previous commit.
- **Branch**: a parallel line of work. `main` is the primary branch — **it is also
  the branch Netlify deploys to production from**, so anything merged into `main`
  goes live. Feature branches (e.g. `claude/site-overview-aozxzh`) let changes be
  made and reviewed without touching the live site until they're merged.
- **`origin`**: the nickname for "the copy of this repo on GitHub" (the **remote**),
  as opposed to the copy on a laptop or in this session (**local**).
- **`git push`**: upload local commits to the remote (GitHub). This is what makes
  local changes visible to others / deployable.
- **`git pull` / `git fetch`**: download changes from the remote. `fetch` just
  downloads and lets you inspect them first; `pull` downloads AND immediately
  merges them into the current branch.
- **Pull request (PR)**: a request to merge one branch's changes into another
  (usually into `main`), with a space for review/comments before it happens.
  Nothing in a PR affects the live site until the PR is **merged**.
- **Merge**: combining the changes from one branch into another.
- **`HEAD`**: shorthand for "the commit you currently have checked out" — i.e.
  where you are right now in the history.
- **Staging area**: where `git add` puts changes before `git commit` locks them
  in. (Not something you usually need to think about — commits in this workflow
  typically add and commit together.)
- **Editing directly on GitHub's website** (the pencil icon on a file) creates a
  commit straight onto whatever branch you're viewing — if that's `main`, it
  goes to production the moment Netlify's next deploy picks it up, with **no PR,
  no review step, and no local testing**. That's higher-risk than the normal
  branch → PR → merge flow, and is how the "DayS" typo shipped straight to
  `main` earlier. Prefer making changes on a branch (even a quick one) and
  opening a PR, so there's a review step and a diff to check before it's live.
- **Deploy**: Netlify's process of taking the latest code on the connected
  branch (`main`) and publishing it to the live URL. A push to `main` triggers
  one automatically; check Netlify's **Deploys** tab to see its status/history.

## 14. Tone / product context (for copy inside the app)

Users are a Christian married couple (Mahdi: Liberian heritage, R&B artist,
pivoting to QA; Lashawn: content creator). Copy should be warm, faith-centered,
practical, and non-hyperbolic. Do NOT introduce diet numbers, calorie/macro
targets, body/appearance commentary, or daily food self-scoring — the "Healthy
Choices" section is intentionally gentle and number-free.