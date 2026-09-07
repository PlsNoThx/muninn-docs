# Cutover — retiring the classic app, the beta at `/`

The task list for making the beta the app. Written 2026-09-07 from a full
audit of `public/app.js` + `public/index.html` against `public/beta/*`,
every `/api/` route's callers, the serving code, `public/sw.js`, and the
docs. Numbers are for reference; the order within a section is the order
to do them. Strike items here as they ship; ROADMAP keeps the story.

What the beta already has that the classic app has, and need not be
redone: sign-in and onboarding (shared `auth.js`), town search with the
saved-here groups and the pick cards (rank, fit, why, caution, Maps link,
swipe to save or pass), near-me with new picks, find-a-place to add with
the already-have state, bulk import with the target chips, CSV picker,
progress, the >250 queue and batched commits, the saved sheet with its
segments, search, three sorts, lenses, provenance tags, remove and undo,
the taste page with preferences and dietary, PWA manifest and the shared
service worker. Beta-only extras the classic never had: the globe, the
reach ring, the place page with journal, dossier, corrections and anchors,
the weight vial, the portrait, dark and lite modes, miles or kilometres.

## A. Blockers — the beta cannot be the only app without these

1. ~~**Owner tools.**~~ Shipped 9/7: `public/beta/owner.js`, a fourth nav
   tab (`#owner`) revealed when `/api/me` says owner, with the three panels
   the classic had behind a segmented row: import approvals (pending and
   processing first, then the last five handled; approve re-reads the
   queue twenty seconds later), the feedback inbox (new first, resolved
   dimmed, mark resolved), and the users list with the per-user drill-down
   (onboarding favorites, saved through the app, passed), read once per
   person. Every route and its owner gate were already in
   `server/server.ts`; nothing changed there.
2. **Feedback capture.** No way for a person to send feedback from the
   beta. Port the classic modal (kind: feature / bug / general, a message,
   `POST /api/feedback` with a context string such as the current town) as
   a line on the taste page or the "Who's Muninn?" panel.
3. ~~**Saved rows without coordinates vanish.**~~ Shipped 9/7 with the
   display-geometry pass (`dev/briefs/places-on-non-google-map.md`):
   `setSaved` keeps every row; the map builders draw only rows with a
   display point, the count and the taste stats count every row.
4. **Near-me with no location.** When geolocation is denied or fails, the
   classic app fell back to the profile hometown; the beta toasts and
   stops. Fall back to `HOME` (fly there, name it, show the ring).
5. **The Back button.** The classic app pushed history per tab so hardware
   Back returned to the previous view; the beta only pushes the place
   page, so Back from the saved or taste sheet leaves the app. Push a
   history entry for each sheet and reconcile on `popstate` the way
   `#place/<id>` does.
6. **The saved-sheet demote drops its reason.** Swiping a want-to-go left
   sends `reason: ""`; the pick path asks. Ask here too (`saved.js`,
   the dislike call), and offer the note on a promote (`/api/relist`), as
   the classic did. This is the dislikes-that-learn signal.
7. **Serving the beta at `/`.** In one commit, with a deploy window:
   - `server/server.ts` static mapping: `/` serves `/beta/index.html`;
     keep `/beta` and `/beta/` as aliases for installed PWAs whose
     `start_url` is `/beta/` (a later commit can 301 them).
   - Leave the assets under `/beta/…` (every module import, `world.geo.json`
     and `cities.json` are absolute `/beta/` paths); only the HTML is
     path-sensitive.
   - `public/beta/manifest.webmanifest`: `start_url` to `/`. Decide the
     classic manifest's fate (it also claims scope `/`); deleting it makes
     an installed classic PWA open `/`, which is the beta, which is the
     point.
   - Icons: shared `auth.js` loads `/icon.svg` for the sign-in and
     onboarding cards. Point it at the beta's icon or keep the file.
   - `public/sw.js`: the `SHELL` precache list is entirely classic
     (`/index.html`, `/styles.css`, `/app.js`); once those are gone
     `cache.addAll` rejects and the worker never installs. Replace the
     list with the beta shell, set the offline fallback to the beta HTML,
     bump `CACHE`, and fix the activate step that deletes every cache but
     the shell, which wipes the tile cache on every bump (whitelist
     `TILES`).
   - Supabase Auth: `redirect_to` becomes `origin + "/"`; confirm `/` and
     `/beta/` are both on the allow-list through the window.
   - Cloudflare: the repointed `/` HTML is `no-cache`, but verify on the
     phone after the deploy, and remember the `__V__` token is not
     stamped into `.webmanifest` or `.json`.
   - **Picks:** `PIN_PICKS` must be off, or picks resolved through the
     display resolver, before this ships. With the flag on, a search pins
     Google coordinates on the MapLibre map (C18). The flag is a staging
     device, not a resolution; the options and numbers are in PROGRESS.
8. **The town route's heartbeat.** `/api/recommend` still has the 100 s
   Cloudflare limit; the beta's fallback for an unplaceable name uses it.
   Once the classic client is gone the route can heartbeat like
   `/api/nearby-recommend`, with `fetchArea` reading `data.error`.

## B. Parity worth keeping — do before or soon after the switch

9. **Import review and misses.** The classic review showed each match
   with its meta, a Maps link, an `exact match` / `by name` tag and a
   `check this one` flag, let you drop one, and gave every miss its own
   lookup box with a "Use this" accept; the beta shows a checkbox list and
   a "not found: …" line. Port the miss rows at least: an import of a long
   list lives or dies on them.
10. **Import robustness.** A Cancel during lookup; commit batches with the
    classic's three retries and text-first parsing (a 502 HTML page
    becomes a retry, not a crash); a notice when the 2,000 cap truncates;
    the feature-ID diagnostics when the legacy Places API is missing from
    the key; the duplicate count in the commit summary; the Takeout
    how-to (five steps) rather than one line.
11. **Find-it refine.** "Not the right place? add details" with the query
    pre-filled, re-asking `/api/find`. The beta has a static hint.
12. **De-dup picks by `place_id`** in `tierRecs`; the classic had
    `dedupeRecs` as a guard.
13. **The saved sheet's 200-row cap.** Ian has ~2,000 places; page or
    virtualise rather than cap.
14. **Refresh `HOME` after a profile save** so a new hometown steers the
    next search without a reload.
15. **Remember the last town** across reloads (classic: localStorage;
    beta: sessionStorage, cleared) — or decide the session-only behaviour
    is intended and say so in ROADMAP.
16. **Lenses.** Resting by decision (ROADMAP open item 9). If they return,
    `NEAR_LENS` needs a `shops` mapping. If not, remove `NEAR_LENS`.
17. **Small things the classic had:** the account chip (avatar, name,
    email, sign-out in the header rather than under settings); the "How
    Muninn works" explainer on the taste page; the saved-here "See all N"
    fold; the "why nothing" line in plain words when a search finds
    nothing (the beta only has `?debug=1`); spin-the-globe's curated
    cities with blurbs and a "show picks here" button (decide: the flick
    lands on a random top-400 city today).

## C. Before strangers sign up (GROWTH.md Part 2, restated as tasks)

18. **Google Places content on a non-Google map.** Investigated 9/7
    (`dev/briefs/places-on-non-google-map.md`): ToS §3.2.3 rules out "as
    is", and attribution does not cure it. Built the same day: an open
    provider for display, Google for resolution only, for **saved places
    and the hometown** (`place_display`, `src/lib/display.ts`, the backfill
    script, `home_display_*` on the profile). **Still open: picks.** A
    search still pins live Places candidates from Google coordinates
    behind `PIN_PICKS` (default on, today's behaviour). A7 cannot ship with
    that flag on; the options with their latency and cost are in PROGRESS,
    and the decision is Ian's.
19. **Attribution on screen.** "© OpenStreetMap contributors" visible
    (MapLibre's control, never hidden); "powered by Google" wherever
    Places data shows — that is the cards, lists and sheets; a pin is
    now an OpenStreetMap point and carries the OSM credit instead;
    terrain and imagery credits in an about panel.
20. **Privacy policy and terms**, linked from onboarding ("by continuing
    you agree") and the about panel. Template services are fine for the
    beta. Saved-places history is location-pattern data; say so.
21. **Delete my account.** Productise `db/wipe_account.sql` as a button on
    the taste page (with a confirm), plus sign-out and local wipe.
22. **Entity and name.** LLC before public launch; a trademark knockout
    search on "Muninn" in software and travel.

## D. After the switch — cleanup

23. Delete `public/index.html`, `public/app.js`, `public/styles.css`,
    `public/manifest.webmanifest`, the Leaflet tags, and the root
    `icon.svg` once nothing loads it. Remove the `/api/nearby` and
    `/api/add` handlers (`/api/nearby` has only the classic caller;
    `/api/add` has none).
24. Docs: README's "Web app (PWA)" section (two tabs, Leaflet,
    `/api/nearby`), the project-structure list, CLAUDE.md's "Globe/beta UI
    work stays at `/beta`" rule and its `public/app.js` pointers, ROADMAP's
    "don't touch the classic app" note and "icon.svg stays for the classic
    app", PLACE_CARDS.md's scope wall. Retire docs/GLOBE.md's two-engine
    history to a line.
25. Move the beta out of `public/beta/` only if the path stops mattering;
    the `/beta/` asset paths are fine to keep forever and save a
    cache-busting round with Cloudflare.

## Not blockers, listed so they are not mistaken for ones

Coin-colour legend, share-as-image, the logo, the onboarding synopsis,
too-broad favourites, the eval baseline, the dossier model experiment, the
wider dossier backfill, the job-and-poll search (the durable answer to
long searches; the heartbeat holds for now). All in ROADMAP "Open work"
and PROGRESS.
