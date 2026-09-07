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

- `src/lib/` — all shared logic (places, recommend, profile, store, auth,
  usercard, plan, add, saved, similarity, csv/pool/geo/avoid/types). CLI
  scripts and `server/server.ts` are thin callers over this — same types
  throughout, one language.
- `server/server.ts` — static + JSON API, auth-aware via Supabase token verify.
- `src/engine/` — the taste engine switch (`index.ts`): v2 in `src/engine/v2/`
  (facets, anchors, dossiers, briefs, pre-rank, Sonnet rank, on-save dossier),
  v1 in `src/engine/v1/` over `src/lib/recommend.ts`.
- `db/schema.sql` — all tables (profiles, added_places, dislikes, feedback,
  place_resolutions, import_requests, place_research, and the v2 block:
  place_dossier, town_research, anchor_state, added_places.weight /
  anchor_facets / business_status; also in `db/v2.sql`). `db/wipe_account.sql`
  — scoped single-account reset.
- Multi-user data model, Phase A/B plan: docs/ARCHITECTURE.md
- Taste engine v2 design (facet anchors, dossiers, town briefs): docs/TASTE_ENGINE.md
- Beta (`/beta`) living tracker, shipped batches, open work, and the
  hard-won MapLibre / browser / rig facts: docs/ROADMAP.md. docs/GLOBE.md is
  history (the globe.gl and two-engine eras). Next artistic pass: docs/SHADERS.md.
- Next session's brief (place cards): docs/PLACE_CARDS.md. Per-session
  handoff: PROGRESS.md.
- Retiring the classic app and serving the beta at `/`: the audited task
  list is docs/CUTOVER.md (blockers first).
- Growth, pricing, and the pre-launch legal checklist: docs/GROWTH.md
- Longer-horizon ideas (taste graph, integrations): docs/VISION.md

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
- **Globe/beta UI work stays at the `/beta` path** — don't touch live app
  routes or components while it's in progress (`public/app.js`,
  `public/index.html`; `public/auth.js` and `public/sw.js` are shared).
  Asset URLs carry a `__V__` token the server stamps per boot; keep it on any
  new `/beta` file or Cloudflare will serve the old one for hours.
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
- Before ending a session: update `PROGRESS.md` (current TODO, what changed,
  what's next) and commit with a descriptive message. This — not conversation
  history — is how context survives into the next session.
- For open-ended investigation ("why is X slow", "how does Y work"), use a
  subagent rather than burning main-thread context.

## Compaction

- When compacting, preserve: files modified this session, the current TODO
  from `PROGRESS.md`, and any test/build commands run.
- Compact proactively at a natural task boundary, not right before hitting the
  limit.
