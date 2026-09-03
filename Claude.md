# CLAUDE.md — Elevated (75-Day Couples Transformation Tracker)

Guidance for Claude Code (or any AI/dev) working on this project.

---

## 1. What this project is

A **single self-contained HTML file** that is a faith-centered daily habit tracker
over a 75-day journey. Originally built for one married couple (Mahdi & Lashawn),
now a multi-tenant app anyone can sign up for, with optional partner pairing.

- **Main file:** `index.html` (identical copy kept as `75-days-luxury-tracker.html`,
  which is now stale relative to `index.html` — update both together or drop it).
- **Size:** ~9,700 lines. All CSS, JavaScript, SVG avatars, and Firebase config
  live INSIDE this one file. There are no build steps, no bundler, no external
  source files it depends on.
- **The two `.png` files** are avatar previews for reference only — the app does
  not load them.
- **Hosting:** Netlify, connected to GitHub — pushes/merges to `main` auto-deploy.
- **Users:** originally just `mahdi` and `lashawn` (a married couple); the app now
  has **real multi-account sign-up via Firebase Auth** — anyone can create an
  account. Mahdi and Lashawn are just the first two accounts, distinguished
  internally by `profileType` (see §3/§6) rather than being hardcoded identities.

## 2. How to run / test locally

- Just open `index.html` in a browser. Everything works offline except cloud sync
  (which needs internet + Firebase).
- There is no test framework. Validation is done by static checks (see §7).

## 3. Data & persistence model

- One global `data` object, now keyed **by Firebase Auth UID** (`data[uid]`), not
  by a hardcoded name. `currentUser` holds the signed-in account's UID.
- Persisted to browser `localStorage` under the key **`75data`**.
- Each account object contains everything from the original model — `days`
  (dateKey -> {itemKey: bool, _water: n}), `goals`, `entries` (notes, keyed
  `dateKey|itemKey` -> [{text,time}]), `photos` (on-device only), `emotions`,
  `checkins`, `archives`, `weekFlags`, `wheel`, `bible` (dateKey ->
  {book,chapter,verse}), `locked`, `sectionLocks`, `layout` ({order:[], hidden:[]}),
  `startDate`, `themeMode`, `lastModified` (critical for sync, see §5) — **plus**
  keys added for multi-account support:
  - `profileType`: `'mahdi'` | `'lashawn'` | `'default'`, claimed once at sign-up
    (see §6). Drives which pillar set, companion avatar, and journey title an
    account gets — **never index `pillars`/`COMPANIONS` by a raw UID**, always go
    through `profileTypeFor(uid)` / `pillarsFor(uid)` / `companionFor(uid)`.
  - `colorTheme`: chosen at onboarding (non-Mahdi/Lashawn accounts), e.g. `'midnight'`.
  - `onboarded`: whether the post-sign-up name/theme flow has been completed.
  - `teamMode`, `lastTeamResetAt`, `lastSeenPartnerResetAt`: mutual opt-in linked
    resets between paired partners (see §6).
  - `partnerUid`, `inviteCode`: invite-code pairing state.
  - `jobApplications`: running list for the Job Application Tracker, persists
    across the whole journey (not tied to a single day).
  - `contentSeed`: randomizes which day's content (verse/quote/etc.) shows first;
    stable within a journey, rerolled only on Reset Journey.
  - All of the above are initialized in `ensureUserKeys(u)` — that function is now
    the single source of truth for "what keys does every account need," see §8.
- **Checkbox id / data key format:** `` `${pillarName}-${itemName}` `` e.g.
  `Faith-Prayer`, `Nourish-Water`. Water key is `Nourish-Water`.

## 4. Cloud sync & auth (Firebase / Firestore)

- Firebase project **`elevated-75-tracker`**; compat SDK v10.12.2 (app, auth,
  firestore) loaded in the head.
- **Auth**: real email/password sign-up and sign-in (`firebase.auth()`).
  `onAuthStateChanged` (registered inside `initFirebase()`) is the single source
  of truth for routing — signed in → straight into that account's own journey;
  signed out → the auth screen. Session persistence is explicitly set to LOCAL.
- **Firestore collections:**
  - `journeys/<uid>` — one doc per account: `{ payload: JSON(data[uid] WITHOUT
    photos), writeTs, fromDevice, updated }`. This replaced the old
    `journeys/mahdi` / `journeys/lashawn` name-keyed docs.
  - `users/<uid>` — `{ profileType, displayName }`, looked up on sign-in/session
    restore to know which pillar set etc. to load.
  - `slots/mahdi` and `slots/lashawn` — claimed exactly once, at sign-up, by
    whichever account first claims that `profileType` (see §6). Prevents a new
    public sign-up from accidentally taking the Mahdi/Lashawn identity.
  - `invites/<code>` — invite-code pairing docs (see §6).
- **`migrateLegacyIdentity(uid, profileType)`**: one-time copy of the old
  name-keyed `data.mahdi`/`data.lashawn` (local) and `journeys/mahdi`/`journeys/lashawn`
  (cloud) into the new UID-keyed record, so the original two accounts' history
  wasn't lost in the migration to real auth.
- `initFirebase()` runs on `window.onload` (must NOT run in <head> — load-order).
- **Photos are never cloud-synced** — they stay on each device.
- **No live/real-time listener** (`startCloudListener()` is intentionally a
  no-op; the old implementation is kept as `startCloudListener_DISABLED()` for
  reference — it caused disruptive re-renders mid-typing and raced with manual
  saves). Instead: `cloudSaveSilent()` auto-pushes on a debounce after local
  changes, and `pullOnReturnToApp()` auto-pulls (via `cloudSyncAll()`) whenever
  the tab/app regains focus or visibility — so a device picks up changes without
  a disruptive live subscription. Manual "☁️ Load from Cloud" still exists too.
  ⚠️ `startCloudListener_DISABLED()`'s inline key-migration block is a separate,
  older copy of what `ensureUserKeys()` does and has drifted out of sync (missing
  `teamMode`, `partnerUid`, `colorTheme`, etc.) — if it's ever re-enabled, replace
  that block with a call to `ensureUserKeys()` instead of hand-copying keys again.

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

Sections ("pillars") are keyed by **`profileType`** in the `pillars` object, not by
UID or literal name — resolve with `pillarsFor(uid)`:
- **`mahdi`:** Nourish [Water, Healthy Choices] · Faith [Prayer, Faith and Mind,
  Journal, Church] · Physical [Physical Activity] · Professional [Job Apps,
  Prof Dev] · Creativity [Music Work, Brand Work] · Love [Love and Connection]
- **`lashawn`:** Nourish · Faith · Physical · Professional [Career Growth, Prof Dev]
  · Content [Content, Engagement] · Wellness [Me Time, Self-Care, Boundaries,
  Inner Beauty] · Love [Love and Connection]
- **`default`** (every other sign-up): Nourish [Water, Healthy Choices] · Faith
  [Prayer, Faith and Mind, Journal] · Physical [Physical Activity] ·
  Professional [Prof Dev] · Love [Love and Connection] — a generic, shorter
  starter set. New accounts also pick a `colorTheme` (e.g. "Midnight Dawn") at
  onboarding, since the app no longer defaults everyone into Mahdi/Lashawn's look.

**Identity claiming**: `profileType` is set once at sign-up via
`claimProfileSlot(uid, profileType)`, which atomically claims `slots/mahdi` or
`slots/lashawn` in Firestore (first come, first served) — everyone else gets
`'default'`. This is how the app tells "the real Mahdi/Lashawn accounts" apart
from any other public sign-up while still using one shared codebase and auth
system.

**Partner pairing & Team Mode**: any two accounts can link via an invite code
(⚙ More → 🔗 Get Invite Code / Enter Partner's Code → `partnerUid`/`inviteCode`
on `data[uid]`, docs in the `invites` collection) — not just Mahdi/Lashawn.
Once paired, both can opt into **Team Mode**: mutual opt-in only, and once both
sides have it on, a reset (manual or from missing the 70% weekly goal twice) on
either account resets both — `lastTeamResetAt`/`lastSeenPartnerResetAt` dedupe
so a partner's device only honors each reset event once.

**Job Application Tracker**: a standalone running list (`data[user].jobApplications`,
opened via a button inside the Job Apps item) — separate from the daily
checkbox, persists across the whole journey.

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

`ensureUserKeys(u)` is now the single canonical place new per-account keys get
initialized — call it, don't hand-roll another `if (!data[u].x) data[u].x = ...`
elsewhere. Still check these other spots when adding a key, since they can run
before `ensureUserKeys()` does or bypass it: the main load migration, `cloudLoad()`,
`importData()`, and `pushBothToCloud()`. The disabled `startCloudListener_DISABLED()`
has its own OLD hardcoded key list (see §4 warning) — do not add new keys there,
it's out of date and only kept for reference. Grep for an existing key like
`data[u].bible` or `data[u].teamMode` to find every live spot.

## 9. Key functions (where to look)

- `render()` — builds the daily view; uses `getOrderedPillars(currentUser)`.
- `loadToday()` — hydrates checkboxes/notes, applies collapse + lock states.
- `initFirebase()` / `onAuthStateChanged` — auth bootstrap and screen routing.
- `profileTypeFor(uid)` / `pillarsFor(uid)` / `companionFor(uid)` — resolve an
  account's identity-dependent data; always go through these, never index
  `pillars`/`COMPANIONS` by a raw UID.
- `claimProfileSlot(uid, profileType)` — one-time `mahdi`/`lashawn` identity claim.
- `migrateLegacyIdentity(uid, profileType)` — copies old name-keyed local/cloud
  data into a newly-authenticated Mahdi/Lashawn UID.
- `ensureUserKeys(u)` — canonical per-account key initialization (§8).
- `cloudSyncAll(cb)` / `cloudSaveSilent()` / `pushBothToCloud(cb)` — sync.
- `pullOnReturnToApp()` — auto-pull on tab focus/visibility (§4).
- `save()` — writes local + stamps lastModified + schedules cloud save.
- `renderBibleStudy()` / `bibleCommentary()` / `blbUrl()` — Bible tool.
- `companionAvatarSVG(profileType, size)` — returns inline SVG for Eli/Nia.
- `getOrderedPillars(u)` / `getLayoutPref(u)` — custom layout.
- `saveSection/unlockSection/toggleSection/applySectionStates` — per-section save.
- `lockDayReview/unlockDay/applyDayLockReview` — whole-day lock.
- `toggleTeamMode()` / `applyTeamModeToggle()` — linked-partner reset opt-in.
- `openJobTracker()` and the `jobApplications` handlers — Job Application Tracker.
- `exportData/importData` — combined backup (currently exports every account
  present in local `data`, not hardcoded to two).

## 10. Known dead / legacy code (safe to clean up, currently harmless)

- `togglePillarCollapse()` + `collapsedPillars` — from the old collapsible-section
  design; superseded by `toggleSection`/section-collapse. No longer called.
- `cycleSpouseTip()` + `spouseTipText` — leftover from the pre-merge "Time with
  Lashawn" block; the merged "Love and Connection" block doesn't use it.
- `startCloudListener_DISABLED()` — kept intentionally as reference; not called.
- The old `if (item === 'Progress Photo')` render block — never fires now that
  Progress Photo is an optional camera button inside Physical Activity.

## 11. Release checklist (tell the user every time)

1. In the live app: ⚙ More → **⬇ Backup** first (exports every account present
   in local `data`, not just two).
2. Deploys now happen automatically via Netlify's GitHub connection — merging to
   `main` is the release; there's no manual drag-and-drop step anymore, but a
   push to `main` (direct or merged) is instantly live, so treat merging to
   `main` with the same care as a manual deploy used to require.
3. Photos are NOT in backups/cloud — they live per-device.
4. This is now a public-facing multi-tenant app — a change that touches auth,
   the `journeys`/`users`/`slots`/`invites` collections, or `ensureUserKeys()`
   can affect every signed-up account, not just Mahdi/Lashawn. Treat those
   changes with correspondingly more caution than a purely cosmetic edit.

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