# Muninn — Project Rules

Persistent AI taste-profile recommendation app. Multi-user, live at
**muninn.ianquimby.com** (Render, auto-deploy from `main`), ~25 real users.
Current state, full feature list, and setup steps: see @README. This file is
for rules that change how Claude should work in this repo — not a feature log.

## Commands

- `npm install`
- `npm run enrich` — Places-enrich `data/taste_profile_places.csv` (flags:
  `--dry-run`, `--favorites-only`, `--limit`, `--concurrency`, `--qps`,
  `--min-confidence`, `--overwrite`). Prefer CI: `.github/workflows/enrich.yml`.
- `npm run profile` — rebuild `data/taste_profile.{json,md}` from enriched favorites
- `npm run recommend -- "Town, ST"` — CLI recommender (flags: `--request`,
  `--keywords`, `--top`, `--model`, `--card`, `--json`, see @README)
- `npm run serve` — local web server, http://localhost:3000
- `npx tsx scripts/eval.ts --engine v2 --limit 30` — taste-engine v2 eval harness
- No test suite yet — if you add one, put the command here.
- Deploy: push to `main` (Render auto-deploys). Full steps: docs/DEPLOY.md.

## Architecture (pointers, not prose — don't duplicate these docs here)

- `src/lib/` — shared logic (places, profile, store, auth, usercard, plan, add,
  saved, similarity, csv/pool/geo/avoid/types). CLI scripts and
  `server/server.ts` are thin callers over this.
- `src/engine/` — the recommender behind an engine switch (`index.ts`).
  `TASTE_ENGINE=v1` selects the old flow, which is also the automatic fallback.
- `public/beta/` — the app: `beta.js`, `saved.js`, `taste.js`, `place.js`,
  `cards.js`, `owner.js`, `beta.css`, MapLibre. **This is the product.**
- `public/app.js` + `public/index.html` — the classic app. A separate, working
  product. Don't touch until CUTOVER section D. `auth.js` and `sw.js` are
  shared with it; a change there must keep it working.
- `db/schema.sql` — all tables. `db/wipe_account.sql` — scoped account reset.

**Read the right doc for the work:**

- `PROGRESS.md` (root) — the current page. What changed, what's open, what's next.
- `docs/ROADMAP.md` — living tracker. Shipped work and open work. **The
  "Hard-won facts" section at the bottom is required reading before touching
  the map.**
- `docs/CUTOVER.md` — retiring the classic app, serving the beta at `/`.
  The audited task list. Section A in order.
- `docs/TASTE_ENGINE.md` — taste engine v2: facets, anchors, dossiers, briefs.
- `docs/GROWTH.md` — monetisation and the pre-launch legal checklist.
- `docs/PLACE_CARDS.md` — the place card and page brief.
- `docs/SHADERS.md` — the next artistic pass.
- `docs/ARCHITECTURE.md` — multi-user data model, Phase A/B.
- `docs/VISION.md` — long horizon.
- `docs/DEPLOY.md` — Render + Supabase.
- `docs/GLOBE.md` — **history only.** The globe.gl and two-engine eras,
  superseded by the single MapLibre engine. Don't build from it.

## Conventions

- TypeScript strict, Node ≥ 20, native `fetch`, run via `tsx`. Named exports.
- Recommendation logic lives only in `src/engine/` (v2) and
  `src/lib/recommend.ts` (v1) — CLI and server both call it through
  `src/engine/index.ts`; never fork the logic into the server route.
- New CLI flags/env vars: document in @README, not here.

## Hard rules (gotchas that cause real mistakes if missed)

- **Google Places ToS**: cache only `place_id` (indefinite) and lat/lng
  (<30 days). Never persist other Places fields (name, rating, etc.) beyond a
  session cache.
- **Field-mask discipline**: never request `rating`/reviews/photos unless
  actually rendered — it silently jumps the SKU from Essentials ($5/1,000) to
  Enterprise or Enterprise+Atmosphere ($35–40/1,000). (v2 deliberately asks
  for the Atmosphere review fields on candidate searches to build dossiers,
  TASTE_ENGINE.md §4.3; photos are not requested anywhere — keep it so.)
- **All place resolution goes through the shared resolution cache**
  (`place_resolutions`, 180-day TTL) — never call Places directly for a lookup
  that could hit cache first.
- **Imports over ~250 places go to the owner-approval queue**
  (`import_requests`) — never auto-resolve a large paste/CSV without that gate.
- **Bulk import's exact feature-ID match needs the legacy "Places API"**
  (separate product from "Places API (New)") enabled and in the key's
  restrictions, alongside Places API (New).
- **No `SUPABASE_URL`/`SUPABASE_ANON_KEY` set = single-user mode.** Don't break
  this fallback when touching auth code.
- **Deep-context research cache** (`place_research`, 45-day TTL) is shared
  across all users and both rank types (fast + deep) — populate once, reuse
  everywhere; don't re-research a cached `place_id`. That is the v1 cache;
  v2 uses `place_dossier` (90-day, one dossier per `place_id` for everyone)
  and `town_research` (per metro × facet group). Same rule: never re-derive
  a fresh cached row.
- **`TASTE_ENGINE` defaults to v2**; v1 exists only as `TASTE_ENGINE=v1` /
  automatic fallback. Confirm which engine a change targets.
- **`public/app.js` and `public/index.html` are the classic app — don't touch
  them until CUTOVER section D.** `public/auth.js` and `public/sw.js` are
  shared; a change there must keep the classic app working until A7.
- **No Google-derived coordinate goes on the MapLibre map.** Maps Platform ToS
  §3.2.3 prohibits displaying Places content with or near a non-Google map, and
  attribution does not cure it. Places stays for resolution, candidates,
  dossiers and ranking. See `dev/briefs/places-on-non-google-map.md`.
- **Every new beta file carries the `__V__` token** — Cloudflare edge-caches by
  extension and will otherwise serve stale assets for hours.
- **API keys never reach the browser.** Server-side only, no secrets in logs
  or commits.
- **Every push to `main` restarts the server and kills any town brief in
  flight** (a brief is minutes of background searching). Batch docs with
  code; don't push a docs-only commit while Ian is testing a new town.
- **Search speed is the models' own throughput at the hour**, measured and
  documented (ROADMAP, Claude API facts; `?debug=1` shows every stage).
  Don't add more latency instrumentation; measure on the second search
  after a deploy, never the first (a changed schema compiles once).
- **`?debug=1` on any search is the engine's full account** (memory,
  anchors, timings, API counts, rank usage, shortlist). Ask for it before
  theorising about a bad result.

## Session workflow

- One feature or bug per session. `/clear` between unrelated tasks rather than
  continuing in the same long conversation.
- Start with `/session-start <brief>` — it reads `PROGRESS.md`, the brief in
  `dev/briefs/`, and recent git log, then states a plan before touching code.
- End with `/session-end` — it updates `PROGRESS.md` and commits. This file,
  not conversation history, is how context reaches the next session.
- For open-ended investigation ("why is X slow", "how does Y work"), use a
  subagent rather than burning main-thread context.
- **Commit subjects are read as a project log.** State what changed and why —
  never `wip`, `fixes`, or `update`.
- **`PROGRESS.md`, `CLAUDE.md`, `README.md` and `docs/` are mirrored to a public
  repo** by `.github/workflows/mirror-docs.yml`. Never write keys, `.env`
  values, user emails, `OWNER_EMAIL`, user data, or Supabase project refs into
  any of them.

## Compaction

- When compacting, preserve: files modified this session, the current TODO
  from `PROGRESS.md`, and any test/build commands run.
- Compact proactively at a natural task boundary, not right before hitting the
  limit.
