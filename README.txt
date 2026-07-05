ELEVATED — 75-Day Transformation Tracker
=========================================

WHAT'S IN HERE
--------------
index.html                     -> The entire app. This IS your site.
                                  (Renamed from the tracker file so hosts load it automatically.)
75-days-luxury-tracker.html    -> The exact same file under its working name, for your reference.
eli_FROM_FILE.png              -> Preview image of Eli's avatar (reference only).
nia_FROM_FILE.png              -> Preview image of Nia's avatar (reference only).

The whole app lives inside the single HTML file. All the styling, the code,
the avatars (drawn in code), and the Firebase/cloud-sync config are contained
inside it — there are no other files it depends on. The two PNGs are just
previews; the app does NOT need them to run.

HOW TO USE IT
-------------
Option A — Just open it:
  Double-click index.html and it opens in your browser. Works offline for
  everything except cloud sync (which needs internet).

Option B — Host it (recommended, what you've been doing):
  Drag this folder (or just index.html) onto Netlify at the SAME site as before
  to keep your existing URL and data. Because the main file is now named
  index.html, the host will load it automatically.

IMPORTANT BEFORE REPLACING YOUR LIVE SITE
-----------------------------------------
1. In the live app, open the ⚙ More menu and tap "⬇ Backup" first.
2. Then upload this version to the SAME Netlify URL.
3. Do it on BOTH phones and the computer so every device is on this version.

NOTES
-----
- Data is stored per-device in the browser AND synced to the cloud (Firebase).
- Photos stay on each device (they are not part of the cloud sync or backups).
- The combined ⬇ Backup file (made from inside the app) contains BOTH accounts'
  data and is the thing to keep safe.
