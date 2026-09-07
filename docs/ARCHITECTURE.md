# Muninn — Multi-user Architecture

How logins, per-user taste profiles, the shared place catalog, and scenario
anchors fit together. Written before building so the data model is settled
first. Two phases: **A** ships auth + per-user taste and migrates my data; **B**
adds the global catalog and scenario anchors.

## Principles

- **Graceful degradation, same as the rest of the app.** With no Supabase Auth
  configured the server runs exactly as it does today (single-user, my CSV
  profile). Auth switches on only when `SUPABASE_URL` + `SUPABASE_ANON_KEY` are
  present and a valid token arrives. Nothing breaks mid-rollout.
- **Keys stay server-side except the ones meant to be public.** The Supabase
  *anon* key and URL are public by design (they're shipped to every browser);
  the *service_role* key never leaves the server. The server hands the browser
  its public config via `GET /api/config`.
- **The committed CSV stays as my seed/backup.** It is not deleted. It is my
  account's starting taste set. Phase B lifts its places into the shared catalog.

## Auth (Phase A)

Supabase Auth (GoTrue), two methods, open signup:

- **Email + password** — signup, login, password recovery (`/auth/v1/recover`).
- **Google** — OAuth redirect (`/auth/v1/authorize?provider=google`).

The browser talks to GoTrue's REST endpoints directly using the anon key (no
SDK). It keeps the returned `access_token` / `refresh_token` in `localStorage`
and sends `Authorization: Bearer <access_token>` on every `/api/*` call.

The server verifies each token by calling `GET {SUPABASE_URL}/auth/v1/user`
with the bearer token (`src/lib/auth.ts`). That returns the authenticated
`{ id, email }` — the source of truth for scoping. A request with no/invalid
token is anonymous: in single-user (no-auth) mode it's allowed as today; once
auth is required it's rejected from write/profile routes.

**Admin** is a single env var, `OWNER_EMAIL`. If the verified email equals it,
`is_owner` is true — that account gets the rich CSV-seeded profile and (later)
admin views. No admin table, no roles system at this scale.

## Data model

### `profiles` (Phase A)

One row per user, keyed to the auth user id.

| column          | type        | notes                                            |
|-----------------|-------------|--------------------------------------------------|
| id              | uuid PK     | = `auth.users.id`                                |
| email           | text        | denormalized for convenience                     |
| display_name    | text        | optional                                         |
| hometown_name   | text        | e.g. "St. Petersburg, FL"                        |
| hometown_lat    | double      | for Near-me default + locality-matched anchors   |
| hometown_lng    | double      |                                                  |
| favorites       | jsonb       | onboarding all-time favorites: `[{name, note}]`  |
| dietary         | text[]      | e.g. `{gluten_free, vegetarian}`                 |
| dietary_notes   | text        | free text, e.g. "dairy is fine in small amounts" |
| onboarded       | bool        | false until guided setup completes               |
| created_at      | timestamptz |                                                  |

### `added_places` (Phase A change)

Add `user_id uuid` and scope every read/write by it. Existing rows (mine) get
backfilled to my user id once I've signed in and my id is known.

### `places` — shared catalog (Phase B)

The "mega list", independent of Google. When any user adds a place it upserts
here (dedup by `google_place_id`) **and** links to that user. A second user
adding the same spot recalls the existing catalog row instead of re-resolving.

| column          | type    | notes                          |
|-----------------|---------|--------------------------------|
| id              | uuid PK |                                |
| google_place_id | text UQ | dedup key (nullable for manual)|
| name, address   | text    |                                |
| lat, lng        | double  |                                |
| primary_type    | text    |                                |
| types           | text[]  |                                |
| price_level     | int     |                                |
| rating          | real    | last-seen Google rating        |
| rating_count    | int     |                                |
| first_added_by  | uuid    | provenance                     |
| created_at      | ts      |                                |

### `saved_places` — join (Phase B)

Replaces per-user `added_places` with a join to the catalog, carrying the
personal layer: `user_id`, `place_id → places.id`, `note`, `rating_personal`,
`anchor_scenarios text[]`, `added_at`. This is where a place becomes part of
*your* taste while the factual record lives once in `places`.

## Per-user taste profile

The recommender is fed a markdown "taste card" + hard constraints. Per user:

- **Owner (me):** the existing rich CSV-derived card (`data/taste_profile.md`),
  plus my added places, plus my dietary line if set. Unchanged quality.
- **New users:** a compact card generated from onboarding — hometown, stated
  all-time favorites (used as anchors), dietary constraints — plus any places
  they've added. Thin at first, thickening as they use the app. This is why
  onboarding is *guided and rich*: the first recommendations have to be strong
  enough to beat Google Maps on day one, and stated favorites are the only
  calibration signal a brand-new account has.

## Dietary restrictions/preferences

Captured in onboarding, stored on the profile, and injected into ranking as a
**hard constraint**, not a soft preference:

> Do not recommend a place that cannot accommodate these unless research shows
> it explicitly does. Gluten-free → avoid breweries/bakeries/pasta houses
> *unless* known GF-friendly. Vegetarian/vegan → avoid steak/BBQ-only spots
> *unless* they have a real veg menu. Say so in the caution line when a pick is
> a stretch.

In Deep mode (web search) the model can verify accommodation from real reviews
before including a borderline place.

## Scenario anchors (Phase B)

Anchors calibrate "exactly right." A single anchor set is too blunt — lunch,
midweek dinner, and high-end date night are different bars. So anchors get
tagged with scenarios (`anchor_scenarios text[]` on `saved_places`), and the
recommendation flow classifies the request into a scenario and pulls the
matching anchors.

Three-layer, **infer-then-confirm**:

1. **Infer** from the data — a cheap, highly-rated lunch spot is a lunch anchor;
   a pricey late-night favorite is a date-night anchor. The classifier proposes.
2. **Confirm** — surface the inferred tags to the user to accept/adjust. Cheap
   UI, high signal.
3. **Explicit** — the user can tag any saved place for a scenario directly.

The `plan` step (`src/lib/plan.ts`) already classifies a request into
domains/pairing; scenario classification slots in alongside it, and
`recommend()` selects anchors by scenario before building the card.

## Two lists: favorites vs. want-to-go

Muninn mirrors the two Google Maps lists it grew from: **favorites** (been there,
loved it — the strongest taste signal) and **want-to-go** (read about it, saw a
post — aspirational, weaker but real). The enriched CSV already carries a
`source` column (`favorite`, `want_to_go`, `favorite+want_to_go`), so both load
through `loadSaved`; `savedInArea` splits them.

**Surfacing (built).** Search and Near-me show favorites and want-to-go as
separate labeled layers (◇ want-to-go pill). Want-to-go places are excluded from
"new picks" (you already have them listed) but do **not** calibrate ranking as
strongly as favorites — favorites remain the anchor set; want-to-go is a
lower-weight signal.

**Capture (next — swipe triage).** On a town search, each recommendation card
can be swiped: **right → want-to-go** (green, save it for later), **left →
dislike** (with the reason box we already have). This turns browsing results
into building your lists, one gesture at a time.

**Graduation loop (next).** Want-to-go is a queue that should drain into
favorites:
- **Recall by recency** (most-recently-saved first) and **by proximity** (a
  want-to-go place near your current location) — proximity is the strong signal
  that you may have actually visited, so surface "you saved this nearby — been
  yet? rate it" prompts to graduate it to a favorite.
- **Pattern-mine want-to-go** for taste calibration (cuisines/venues you keep
  bookmarking), weighted below favorites — aspiration is a weaker but genuine
  signal of where taste is heading.

## Sequencing

- **Phase A (now):** auth (email+password, Google, recovery) · `profiles` ·
  per-user `added_places` · guided onboarding (hometown, all-time favorites,
  dietary) · per-user taste card · dietary as a hard ranking constraint ·
  `OWNER_EMAIL` admin · my CSV as owner seed. Backward-compatible: no auth
  configured ⇒ today's single-user behavior.
- **Phase B (next):** `places` catalog with `google_place_id` dedup + cross-user
  recall · `saved_places` join · scenario anchors (infer-then-confirm) ·
  scenario classification in the flow.

## One-time setup (owner, in the consoles — not code)

1. **Supabase → Authentication → Providers:** enable Email; enable Google
   (paste a Google OAuth client id/secret).
2. **Google Cloud → Credentials:** OAuth client (Web), authorized redirect URI
   `https://<project>.supabase.co/auth/v1/callback`.
3. **Supabase → SQL editor:** run `db/schema.sql`.
4. **Render env:** add `SUPABASE_ANON_KEY` and `OWNER_EMAIL`
   (`SUPABASE_URL` / `SUPABASE_SERVICE_KEY` already set). Redeploy.
5. First sign-in with the owner email, then run the backfill to stamp existing
   `added_places` rows with the owner id.
