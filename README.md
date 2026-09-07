# Muninn

Personal AI places app — a persistent taste profile built from years of saved
places that gives strong, personal recommendations anywhere, especially in the
secondary/tertiary towns that business travel actually routes through.

Named for one of Odin's ravens, whose name means *Memory*: go out, gather what's
actually good, come back and report — built on accumulated memory of taste, not
just in-the-moment cleverness. See [`CLAUDE.md`](CLAUDE.md)
for the full brief.

## Status — live, multi-user, in daily use

The personal MVP shipped, deployed to **`muninn.ianquimby.com`** (Render,
auto-deploy from `main`), and then grew into a small multi-user app that a
handful of friends are now using and giving feedback on.

**Core pipeline** (the original four pieces — all built and validated):

| # | Piece | Command | State |
| - | ----- | ------- | ----- |
| 1 | **Enrichment** — names → structured place data via Places API | `npm run enrich` | ✅ |
| 2 | **Taste profile** — summarize enriched favorites into a profile | `npm run profile` | ✅ |
| 3 | **Recommendation** — rank a town's candidates against the profile | `npm run recommend` | ✅ |
| 4 | **Interface** — installable PWA over the same library | `npm run serve` | ✅ |

**Since then, shipped:**

- **Accounts (Phase A)** — Supabase Auth (Google + email/password), open signup,
  password reset, per-user data. Owner (`OWNER_EMAIL`) keeps the rich CSV-seeded
  profile; everyone else builds theirs. Backward-compatible: no auth env ⇒
  single-user mode. See [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).
- **Guided onboarding** — hometown, all-time favorites (calibration anchors),
  dietary needs (hard ranking rules).
- **Swipe triage** — recommendation cards swipe **right → want-to-go** (green),
  **left → pass** (red, with an optional reason). Clean, gesture-first.
- **Dislikes that learn** — a pass with a reason excludes that place *and* steers
  away from similar ones (e.g. "too smoky / hookah").
- **General preferences** — free-text Likes / Avoid ("I just like Cuban
  sandwiches"), soft-weighted in ranking, distinct from hard dietary rules.
- **Two lists** — favorites (been there) vs. want-to-go (aspirational), surfaced
  as separate layers, mirroring the two Google Maps lists this grew from.
- **Profile page** — live taste readout, editable preferences, and your added /
  disliked lists.
- **In-app feedback** — users submit ideas/bugs; the owner triages them from a
  Profile-page inbox.
- **CI enrichment** — a manual [GitHub Actions workflow](.github/workflows/enrich.yml)
  runs enrichment with a repo secret and commits the result (no local run needed).
- **Bulk import** — paste a list or drop in a Google Takeout CSV; each entry is
  resolved (exactly, via the saved place's feature ID/`place_id` in the CSV URL),
  scored, de-duped, and shown for **review before saving**. Robust line-based CSV
  parsing (survives quotes/commas/newlines in notes), URL-only rows, resolvable
  misses with skip, coordinate/dropped-pin handling, and a retryable **bulk** commit.
  See [Bulk import](#bulk-import).
- **Cost controls for imports** — a shared **resolution cache** (`place_resolutions`)
  so a re-import doesn't re-bill the Places API, and an **owner-approval queue** for
  imports over ~250 places (Profile → Settings → *Import approvals*). Needs the
  `place_resolutions` and `import_requests` tables (in `db/schema.sql`). The legacy
  Places API's Place-Details billing (feature-ID exact match) is the expensive SKU —
  set a **daily quota cap** in Google Cloud as the hard backstop.
- **Search / sort / lenses over your saved lists** — big lists are now navigable:
  search by name/city/note/type, sort Recent / A–Z / **Nearest me** (shows distance),
  and filter by the same Near-me lenses (Coffee / Drinks / …). Reads paginate past
  PostgREST's 1,000-row cap so >1,000-place accounts load fully.
- **Origin tags** — favorite cards show **anchor** (onboarding), **imported** (from a
  list), or the date. Onboarding favorites now resolve + save to the Favorites list.
- **Deep-context research cache** — the "Deep" web-search ranker's per-place research
  is cached (`place_research`, shared) so it compounds and even the cheap default rank
  reuses it.
- **Admin tools** — the owner can list all users and inspect any account's taste
  and places for diagnosis, from the Profile-page admin panel.
- **Rebuild-from-scratch flow** — `db/wipe_account.sql` + `OWNER_HIDE_SEED` env flag
  let the owner rebuild their account as a fresh user (no CSV seed). See
  [Rebuilding an account](#rebuilding-an-account-from-scratch).

**Current focus:** the **beta UI revamp** — a globe-as-interface home (see
[`docs/GLOBE.md`](docs/GLOBE.md)): globe.gl, a device-clock time-of-day sky, clay
continents, zoom-to-city + "search this area", a cobe Lite mode, all in a
Studio-Ghibli-mythical × modern-digital treatment. Shipping isolated at a `/beta`
path so the live app is untouched. Also queued: the graduation loop (recall
want-to-go's by recency/proximity to infer visits and promote them).

## How it fits together

```
Google Takeout CSVs
   └─▶ taste_profile_places.csv        (cleaned input, checked in)
         └─▶ enrich  ─▶ taste_profile_enriched.csv   (Places API: coords, types, price, rating)
               └─▶ profile ─▶ taste_profile.{json,md}  (taste card the model reasons from)
                     └─▶ recommend ─▶ ranked picks      (Places candidates + Claude ranking)
                           └─▶ serve ─▶ PWA             (same library, behind a web UI)
```

## Stack

- **TypeScript / Node ≥ 20** (native `fetch`), run with [`tsx`](https://github.com/privatenumber/tsx).
- **Google Places API (New)** for place data.
- **Claude API** for recommendation ranking — Haiku 4.5 by default (cheap),
  swappable to Sonnet if ranking needs more nuance.
- One language throughout, so the scripts, the recommendation library, and the
  web server all share the same types (`src/lib/`).

## Setup

```bash
npm install
cp .env.example .env   # then fill in GOOGLE_PLACES_API_KEY
```

To get a Places key: create a project in the
[Google Cloud Console](https://console.cloud.google.com), enable **Places API
(New)**, create an API key, and restrict it to that API. For **bulk import**'s
exact feature-ID matching, also enable the legacy **Places API** and add it to
the key's API restrictions (see [Bulk import](#bulk-import)).

## Enrichment

Reads `data/taste_profile_places.csv` (1,942 unique saved places) and, for each
title, runs a Places **Text Search**, picks the best candidate by name
similarity, and writes an enriched CSV.

```bash
# See what would run — no API calls, no key needed:
npm run enrich -- --dry-run

# Enrich just the favorites first (strongest taste signal, ~877 rows):
npm run enrich -- --favorites-only

# Enrich everything:
npm run enrich
```

The run is **resumable**: results are appended row-by-row, and on restart it
skips titles already present in the output. If you hit a quota or Ctrl-C, just
run the same command again to continue.

**Run it in CI instead of locally:** the
[`Enrich places`](.github/workflows/enrich.yml) GitHub Actions workflow
(Actions tab → *Run workflow*) does the same run using a `GOOGLE_PLACES_API_KEY`
repo secret and commits the updated CSV back to `main` — no local setup needed.

Handy flags:

| Flag | Default | Purpose |
| --- | --- | --- |
| `--favorites-only` | off | Only rows whose source includes `favorite`. |
| `--limit <n>` | all | Process only the first N pending rows (testing). |
| `--concurrency <n>` | 5 | Parallel requests. |
| `--qps <n>` | 10 | Max requests/second (rate cap). |
| `--min-confidence <0-1>` | 0.5 | Below this name similarity → flagged `low_confidence`. |
| `--overwrite` | off | Ignore existing output, start fresh. |
| `--dry-run` | off | Parse and report only; no API calls or writes. |

### Output

- `data/taste_profile_enriched.csv` — every input column plus: `ftid`,
  `place_id`, `resolved_name`, `lat`, `lng`, `primary_type`, `types`,
  `price_level` (0–4), `rating`, `user_rating_count`, `formatted_address`,
  `business_status`, `match_confidence` (0–1), `resolution_status`.
- `data/unresolved_places.csv` — the subset that was `low_confidence`,
  `not_found`, or `error`, for manual review. `ftid` is the Google feature ID
  parsed from the original Maps URL; it uniquely anchors the intended place when
  a name is ambiguous.

Both outputs are gitignored — they're regenerated from the API, not source of
truth. The cleaned input `data/taste_profile_places.csv` is checked in.

### Why a confidence score

Most favorites use distinctive branded names ("Aguardente", "There, There.",
"Catalyst"), which Text Search resolves cleanly. Generic names ("India
Restaurant") legitimately can't be pinned from the name alone and are flagged
low-confidence so they can be checked against the `ftid` link by hand rather than
silently mis-resolved.

## Taste profile

Once the enriched CSV exists, build the taste profile from the **favorites**
(the strongest signal — want-to-go is excluded by default):

```bash
npm run profile
```

Writes two artifacts (both gitignored — regenerated from the enriched data):

- `data/taste_profile.json` — structured stats consumed by the recommender:
  venue-category and cuisine distributions, price band, rating and popularity
  patterns, the branded-vs-descriptive naming signal, geographic footprint, and
  the resolved calibration anchors.
- `data/taste_profile.md` — a compact prose "card" summarizing all of the above,
  meant to be dropped straight into the Claude prompt as taste context in the
  recommendation step.

The **popularity spread** (review-count distribution) is recorded as *context,
not a ranking goal*: the job is to surface the right place for this person in a
town they don't know — famous or not — so the recommender neither rewards
obscurity nor penalizes a great well-known spot. The **anchors** (Odette,
Aguardente, River Bar, Turkey and the Wolf, There There) are pulled from the
enriched data as few-shot "exactly right" examples, matched by expected locality
so a generic name resolves to the right city.

## Recommendations

> **Taste engine v2** is the default as of September 2026: facet-conditional
> anchors, place dossiers, town briefs, a pre-ranked shortlist and Sonnet 5
> ranking. The original flow below is preserved as v1 (`TASTE_ENGINE=v1`, and
> the automatic fallback). The thesis and the design are in
> [`docs/TASTE_ENGINE.md`](docs/TASTE_ENGINE.md); the eval harness is
> `npx tsx scripts/eval.ts --engine v2 --limit 30`. Run the v2 block at the
> end of `db/schema.sql` once so the caches persist.
>
> **Place cards** read the shared dossier for every saved place, so no card
> is blank: `npx tsx scripts/backfill-dossiers.ts --dry-run` lists the saved
> places without one, and without `--dry-run` writes them (about 5¢ a place;
> `--limit N`, `--concurrency N`, `--favourites-only`, `--area <regex>` on
> the address). The same run happens on the server at boot when
> `BACKFILL_DOSSIERS` is set (`all`, `favourites`, a number, or
> `favourites:<n>`; `BACKFILL_AREA` narrows it to addresses matching a
> regex), for running where the keys live; unset both once the log says done. A viewed card with no dossier starts one on its own. Notes and corrections need the "Place cards" block at the end
> of `db/schema.sql` (tables `place_notes`, `place_corrections`).
>
> **Town briefs and search snippets.** A brief reads Reddit, Eater and the
> NYT as search-engine snippets from another index, because Anthropic's own
> search tool cannot see sites that block its crawler. Optional: set
> `BRAVE_SEARCH_API_KEY` from the [Brave Search API](https://brave.com/search/api/)
> ($5 per 1,000 queries, $5 a month free; three queries per brief, cached
> with it). Without it the brief runs on Anthropic's search alone.
> `DOSSIER_MODEL` overrides the dossier model (default `claude-haiku-4-5`). Google's
> Custom Search JSON API (`GOOGLE_SEARCH_CX` + `GOOGLE_SEARCH_API_KEY`) is
> still honoured for projects that already had it; Google closed it to new
> customers in 2026 and retires it on 2027-01-01.

Given a town, pull nearby candidate places and rank them against your taste
profile with Claude (Haiku 4.5 by default). Needs both `GOOGLE_PLACES_API_KEY`
and `ANTHROPIC_API_KEY` in `.env`, plus a built profile card
(`data/taste_profile.md`).

```bash
npm run recommend -- "Asheville, NC"
npm run recommend -- "St. Petersburg, FL" --request "dinner and a good bar" --top 6
npm run recommend -- "Grand Rapids, MI" --keywords "restaurant,coffee,brewery"
```

Two sections come back:

- **Your saved places here** — your own enriched favorites (and want-to-go) that
  fall in the queried area, surfaced directly (no LLM needed — you already love
  them). This is the "accumulated memory" advantage in a town you know.
- **New picks** — fresh Places candidates that aren't already saved and aren't
  on your avoid list, ranked by Claude against your taste. Your local favorites
  are fed to the model as calibration for the local vibe.

The ranking prompt encodes the taste profile and two guiding principles:
**popularity is context, not a target**, and a **lean toward independent,
locally-owned places** (your favorites are almost all independents) — not a
blanket anti-chain rule; specific chains or spots you dislike go on the avoid
list instead.

**Avoid list** (`data/avoid.txt`): one place or brand per line to never
recommend — chains you dislike, or specific spots you avoid. Matching is
case-insensitive and substring-based on the name.

Flags: `--request`, `--keywords`, `--top`, `--max-per-keyword`, `--model`,
`--card`, `--enriched`, `--avoid`, `--json`. The saved-places section needs the
enriched CSV present (`data/taste_profile_enriched.csv`).

## Web app (PWA)

A tiny Node server (no framework) serves an installable web app over the same
recommendation library. The API keys stay on the server — never in the browser.

```bash
npm run serve          # http://localhost:3000
```

Needs `GOOGLE_PLACES_API_KEY` and `ANTHROPIC_API_KEY` in `.env`, plus a taste
profile. The server finds the profile from (in order): `TASTE_PROFILE_MD` (the
card as a string), `TASTE_PROFILE_PATH`, or the default `data/taste_profile.md`.

- `server/server.ts` — static file serving + `POST /api/recommend` +
  `POST /api/nearby`.
- `public/` — the installable PWA (`index.html`, `app.js`, `styles.css`,
  `manifest.webmanifest`, `sw.js`, `icon.svg`).

The app has two tabs:

- **Search a town** — the taste-matched recommendations (saved section + new
  picks) described above.
- **Near me** — uses device geolocation to show your saved gems around you on a
  map (Leaflet + OpenStreetMap) plus a ranked "heavy hitters nearby" list. The
  killer travel move: open the app anywhere and see if any of your spots are
  close. Backed by `POST /api/nearby` (haversine over your enriched coordinates).

**Testing from a Codespace / phone:** run `npm run serve`, then forward the port
(Codespaces does this automatically — open the forwarded URL). On the phone, use
the browser's "Add to Home Screen" to install it.

### Display geometry — where a pin is drawn

Google Places content may not be shown on a non-Google map (Maps Platform ToS
§3.2.3), so the beta's MapLibre chart never draws a Google coordinate. Each
saved place gets an OpenStreetMap point from Nominatim, resolved once per
`place_id` and shared by everyone in `place_display` (`src/lib/display.ts`);
the hometown gets one at profile save (`profiles.home_display_*`). A place the
geocoder cannot match is kept as a miss: it stays in every list and has no pin.
Brief: `dev/briefs/places-on-non-google-map.md`.

```bash
npm run backfill-display -- --dry-run        # count what is unplaced
npm run backfill-display                     # ~1 request/s; ~2,000 places ≈ 40 min, resumable
npm run backfill-display -- --misses         # list the recorded misses
npm run backfill-display -- --retry-misses   # ask again for them
npm run backfill-display -- --min-confidence 0.6 --limit 100
npm run backfill-display -- --homes          # a display point for every profile's hometown
```

Env (all optional): `DISPLAY_GEOCODER` (only `nominatim`), `NOMINATIM_URL`
(a self-hosted or paid instance with the same API, e.g. LocationIQ),
`NOMINATIM_KEY` (a paid tier's key), `NOMINATIM_USER_AGENT` (the public instance
requires a real contact), `NOMINATIM_MIN_MS` (gap between requests, default
1100), `BACKFILL_DISPLAY=1` (run the backfill at server boot, where the
database key lives; unset when it reports done). `PIN_PICKS=0` lists search
picks in the sheet instead of pinning them from Google coordinates — the
default is still on, and that is the open half of CUTOVER C18.

### Bulk import

Profile → Places → **Import a list**. Paste a list (one place per line, ideally
`Name, City`) or choose a **Google Takeout CSV**. Each entry is looked up, then
shown for **review before anything is saved**:

- **Exact resolution from the CSV.** Google Takeout list URLs carry a **feature
  ID** (`…!1s0x…:0x…`); Muninn resolves it to the precise saved place via Google's
  legacy Place Details endpoint. A `place_id` (`ChIJ…`) in the URL is used
  directly; failing both, a name search pinned to any URL coordinates, else the
  hometown. Each card is tagged **exact match** or **by name**.
- **Review, don't auto-save.** Resolved places show as cards (with a Maps ↗ link
  and a "check this one" flag on low-confidence matches) — drop any wrong ones,
  fix misses with an inline look-up, then **Add N** or **Cancel**. Batched
  client-side with a cancelable progress bar; 500 places/run.
- Server: `POST /api/import` with `resolve` and `commit` modes.
- **Resolution cache.** Each resolved place is cached by its stable identity
  (`pid:<place_id>` / `ftid:<feature id>`) in `place_resolutions`, shared across
  users (180-day TTL). A re-import — or two people importing the same place —
  hits the cache instead of re-billing the Places API, which is what a repeated
  big import otherwise does.
- **Owner approval for big imports.** Each place resolved is a paid Places API
  call, so an import over ~250 places isn't resolved on submit — it's **queued**
  (raw names/URLs stored, no API calls) and shows in the owner's Profile →
  Settings → *Import approvals*. On **Approve** the server resolves + saves it to
  the requesting user's account; **Reject** drops it. A cost gate against a
  surprise bill from a huge list. (`import_requests` table; `/api/import` `queue`
  mode, `/api/import-requests`, `/api/import-approve`.)

> **Requires the legacy "Places API"** (separate from "Places API (New)") enabled
> on the key **and** allowed in the key's API restrictions — the feature-ID
> lookup uses it. Without it, distinctive names still resolve via the name-search
> fallback, but exact matching won't run.

**Exporting from Google Maps:** takeout.google.com → *Deselect all* → check
**Saved** → *Create export*; the per-list CSVs land under *Takeout/Saved/*.

### Rebuilding an account from scratch

To rebuild the **owner** account from your own lists (instead of the CSV seed):
clear the account's DB rows with [`db/wipe_account.sql`](db/wipe_account.sql)
(scoped by `user_id`), then set **`OWNER_HIDE_SEED=true`** in the host env. The
owner is then treated like a fresh user — onboarding + imported lists only, no
CSV seed — while keeping admin access. Unset the var to bring the seed back; the
`data/` seed files are never touched. (Non-owner accounts already work this way.)

### Deploying to a domain

The app needs a Node host (it holds the API keys), so it can't be served from a
static host like Framer. Recommended setup:

1. Deploy `muninn` to a free Node host (e.g. **Render**): build command
   `npm install`, start command `npm run serve`. Set env vars there:
   `GOOGLE_PLACES_API_KEY`, `ANTHROPIC_API_KEY`, and `TASTE_PROFILE_MD` (paste
   the contents of your `data/taste_profile.md`); optionally `BRAVE_SEARCH_API_KEY`
   for the town briefs (see Recommendations).
2. Point a subdomain at it — e.g. `muninn.ianquimby.com` — with a CNAME record
   in your DNS (Porkbun) to the host's URL. Leave the apex domain on Framer.

Auto-deploys on every push to `main` via [`render.yaml`](render.yaml). Full
click-by-click steps: [`docs/DEPLOY.md`](docs/DEPLOY.md). The same PWA can later
be wrapped for the app stores (PWABuilder / TWA).

## Accounts & multi-user (Phase A)

Muninn supports logins so friends can build their own taste profiles. It stays
**backward-compatible**: with no auth env set the app runs exactly as the
single-user build did. Auth turns on only when `SUPABASE_URL` +
`SUPABASE_ANON_KEY` are present.

- **Sign-in:** Google + email/password (Supabase Auth), open signup, password
  reset. The browser talks to Supabase directly with the public anon key; the
  server verifies each token via `GET /auth/v1/user`.
- **Admin:** the `OWNER_EMAIL` account is the owner — it keeps the rich
  CSV-seeded taste profile; everyone else builds theirs from onboarding + adds.
- **Onboarding:** hometown, all-time favorites (calibration anchors), and
  dietary needs (treated as **hard** ranking constraints — e.g. gluten-free
  won't be sent to a brewery unless it's known to accommodate).
- **Per-user data:** `added_places` is scoped by `user_id`; `profiles` holds
  onboarding. See [`db/schema.sql`](db/schema.sql).

Design and the Phase-B plan (shared place catalog, scenario anchors) live in
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

**One-time setup (owner):**
1. Supabase → Authentication → Providers: enable Email and Google (paste a
   Google OAuth client id/secret).
2. Google Cloud → Credentials: OAuth Web client; authorized redirect URI
   `https://<project>.supabase.co/auth/v1/callback`. In Supabase → URL
   Configuration, add the app origin (`https://muninn.ianquimby.com`) to Site
   URL / redirect allow-list.
3. Supabase → SQL editor: run `db/schema.sql`.
4. Render env: add `SUPABASE_ANON_KEY` and `OWNER_EMAIL`, redeploy.
5. Sign in once as the owner, then run the backfill in `db/schema.sql` to stamp
   existing `added_places` rows with your user id.

## Project structure

```
CLAUDE.md                 project brief (auto-loaded context)
docs/ARCHITECTURE.md      multi-user data model (Phase A/B), two-list plan
db/schema.sql             Supabase tables (profiles, added_places, dislikes, feedback)
db/wipe_account.sql       scoped one-off reset of a single account's data
data/
  taste_profile_places.csv   cleaned input (checked in)
  taste_profile_enriched.csv enriched places (checked in, private repo)
  taste_profile.{json,md}    taste card the model reasons from
src/lib/
  places.ts               Google Places API (New) client
  similarity.ts           name-matching for enrichment & anchors
  profile.ts              taste-profile builder + card renderer
  recommend.ts            candidate discovery + Claude ranking (shared core)
  saved.ts                CSV + added places, favorites vs want-to-go split
  display.ts              open-provider display points for the map (Nominatim), never Google's
  add.ts                  freeform "add a place" parsing
  plan.ts                 request → domains + "X near Y" pairing
  store.ts                Supabase: added/disliked places, profiles, feedback
  auth.ts                 verify Supabase Auth tokens (server side)
  usercard.ts             per-user taste card + dietary/preference blocks
  csv.ts, pool.ts, geo.ts, avoid.ts, types.ts
scripts/                  CLIs: enrich.ts, build-profile.ts, recommend.ts,
                          backfill-dossiers.ts, backfill-display.ts
server/server.ts          web server (static + JSON API, auth-aware)
public/                   the installable PWA (index.html, app.js, auth.js, …)
.github/workflows/        enrich.yml — run enrichment in CI
```

The recommendation logic lives in `src/lib/recommend.ts` and is called
identically by the CLI (`scripts/recommend.ts`) and the web server — so the app
is a thin UI over the library.

## Roadmap / next

Beyond the MVP, in rough priority order:

1. **Near-me map** ✅ — geolocation + map of your saved gems nearby.
2. **Google deep-link** — from a place, open it in Google Maps so you can save it
   there in one tap. (Google has *no* public API to read or write your Maps saved
   places, so true two-way sync isn't possible — Muninn is the system of record;
   Google is optional convenience.)
3. **Dynamic taste memory (datastore)** ✅ — Supabase-backed store for places
   added at runtime (`src/lib/store.ts`), merged into your saved list. See the
   Supabase setup in [`docs/DEPLOY.md`](docs/DEPLOY.md).
4. **Add a place with context** ✅ — the **Add** tab: type a freeform note
   ("Little Salumi in MI today — great to-go counter, a few outdoor seats");
   Claude parses the place + vibe (`src/lib/add.ts`), Places resolves it, and it
   joins your taste memory with the note attached.
5. **Single-sentence "reviews"** ✅ (via the same flow) — a note on a place you
   already have attaches to it; these one-liners feed the ranker as first-person
   taste context ("counter service, no reservations, go early"). A dedicated
   "review an existing saved place" UI is a nice follow-on.
6. **Web-search-augmented ranking + research cache** ✅ — the **Deep context**
   toggle researches candidates (Sonnet + `web_search`) for what locals and
   critics actually say. Research is now a **separate, structured, per-`place_id`
   step whose notes are cached** in `place_research` (shared across all users, 45-
   day TTL) — so a deep query populates the cache and **every** later rank,
   including the cheap default Haiku one, injects that real-world context for
   free. Bounded (`max_uses`) and best-effort (falls back to the fast rank on
   error). Deep is off by default to keep the first research pass cheap.
7. **Beyond food — domain-aware & combined search** 🚧 in progress. A written
   request is parsed into a structured plan (`src/lib/plan.ts`): which domains to
   search (food, outdoors, shops, coffee, lodging…) and whether the targets
   should be *near each other*. This unlocks non-food queries ("cool shops in
   Providence", "a good hike") and combined natural-language ones ("a good hike
   with a brewery I'd like nearby" — ranks hikes that have a great, close
   brewery). Saved places now surface across all domains, not just food. Next:
   per-domain taste profiles and anchors.
8. **Accounts + per-user profiles** ✅ — Supabase Auth, onboarding, per-user
   data. See the [Accounts](#accounts--multi-user-phase-a) section and
   [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).
9. **Swipe triage + learning dislikes + preferences** ✅ — swipe to build your
   lists; a pass with a reason teaches an aversion; free-text Likes/Avoid steer
   ranking.
10. **Two lists (favorites vs want-to-go)** ✅ surfaced; **graduation loop** 🔜 —
    recall want-to-go's by recency/proximity to infer visits and promote them.
11. **CI enrichment** ✅ — [`.github/workflows/enrich.yml`](.github/workflows/enrich.yml)
    runs enrichment from a repo secret and commits the result.
12. **Bulk import** ✅ — Profile → Places → *Import a list*: paste a list or drop
    in a Google Takeout CSV. Each entry is **resolved but not saved** until you
    confirm — Takeout URLs resolve to the **exact** saved place via their feature
    ID (`place_id` if present; else a name search pinned to URL coordinates, then
    the hometown). Matches are scored, de-duped, and shown as a **review** (drop
    wrong ones, fix misses inline, then **Add N** or **Cancel**). `POST /api/import`
    has `resolve`/`commit` modes. See [Bulk import](#bulk-import). A cold account
    gets a real profile in one paste. (A Yelp/other-app importer and owner-side
    enrichment queue can build on the same path.)
13. **App store wrap** — package the PWA for iOS/Android (PWABuilder / TWA) later.

**Longer horizon** (multi-source data, a collaborative "taste graph", friends,
integrations) is captured in [`docs/VISION.md`](docs/VISION.md).

## Cost

The one-time enrichment of ~1,942 places plus ongoing personal queries sits
comfortably inside the Places API's free monthly allowance — effectively
$0/month at single-user scale. The field mask is kept lean (only location,
types, price, rating) to stay on the cheapest applicable SKU.
