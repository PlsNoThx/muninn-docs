# Muninn — Long-term Vision

The MVP is a **personal** taste engine: learn one person's taste from their saved
places, recommend anywhere. This doc captures the longer horizon — where Muninn
could go if the personal version proves out — and, importantly, an honest read on
what each step actually takes.

The through-line: **taste is the moat.** Anyone can list places; Muninn knows
*whose* taste it's serving and *why* a place fits it. Every idea below compounds
that.

## Phases

1. **Personal (now).** One user, taste profile from Google saved places, Places
   API for data, Claude for ranking. Prove it gives recommendations you'd never
   have found. ✅ shipped.
2. **Richer data (next moat-builder, single-user-friendly).** Stop depending on
   Google Places alone for both candidates *and* judgment. Blend in more sources
   so Muninn's picks reflect what locals and critics actually say — not just a
   prominence score. 🚧 *started:* web-search-augmented ranking (the "Deep
   context" toggle) already pulls real-world critic/local sentiment into scoring;
   structured second sources (Yelp, Reddit) are the next additions.
3. **Platform / social (the real business).** Multiple users → a collaborative
   "taste graph" that recommends based on people whose taste resembles yours, then
   friends. This is the piece Google/Yelp can't easily copy, because it's built on
   personal taste vectors, not aggregate stars. 🚧 *foundation shipped:*
   multi-user accounts, per-user taste profiles, and per-account data now exist
   (Phase A) — the graph itself waits on user density.

## Richer data sources (Phase 2)

The clean architecture: a **candidate-source interface** so any provider feeds a
common pool that the taste-ranker scores. Sources to add:

- **Reddit** — city subreddits and r/food threads are gold for "where locals
  actually go." Usable via the official API, but note: since 2023 it's rate-
  limited and commercial use needs an agreement (not free-for-all scraping). Best
  used to *surface candidates and sentiment*, feeding the ranker — not as a live
  firehose.
- **Yelp Fusion API** — structured places, categories, review counts, price; a
  solid, free-tier second opinion to Google, especially in the US.
- **Editorial / critics** — Eater, Michelin, local blogs. Reachable today via
  Claude with web search: "what do critics and locals say about this place?" as a
  ranking input. ✅ **shipped** as the "Deep context" toggle (Sonnet + `web_search`
  ranks the whole pool with real-world context) — a big quality jump — and the
  research is now **cached per place** (`place_research`, shared across users), so
  it compounds: one deep query enriches everyone's later picks, and even the cheap
  default rank reuses that context. Next: pre-warm the cache for saved places.
- **Design note:** normalize every source to the same place identity (coords +
  name) and let Claude reconcile conflicting signals in the ranking step. Bulk
  import already resolves places to a stable Google identity (`place_id`/feature
  ID) — the same identity key a multi-source candidate pool would normalize to.

The near-term unlock here was **web-search-augmented reasoning** — now live. The
next one is a **candidate-source interface**: a common shape that Yelp, Reddit,
or an editorial feed can each populate, so the taste-ranker scores a blended pool
instead of Google's alone.

## The taste graph (Phase 3 — the moat)

Once there are enough users, build an internal model that rates places by how
people *like you* felt about them — classic collaborative filtering, but keyed on
**taste similarity** rather than raw popularity:

- Each user's profile already is a **taste vector** (cuisine/venue/price/quality
  signals + their saved places). Compute similarity between users → "taste
  neighbors."
- Predict your rating of an unseen place from what your neighbors thought — a
  "ghost rating" that's personal, not an average star count. A 4.9 loved by people
  who share your taste beats a 4.9 loved by everyone.
- **Friend recommendations** layer on top: opt-in social graph, "Alex (84% taste
  overlap) loved this," trips and lists shared between friends.

Honest constraints:
- **Cold-start / density.** Collaborative filtering needs critical mass; it's
  weak until many users have rated many overlapping places. The single-user taste
  profile is the bridge that makes Muninn useful *before* the graph exists — and
  bulk import shortens cold-start further, turning a new account into a real
  profile from a Google/Takeout list in one paste.
- **Multi-user shipped (Phase A).** Accounts/auth, per-user data, and onboarding
  are live and a handful of friends are using it — so the architecture shift is
  largely done. What the graph still needs before it's worth building: **density**
  (many users, overlapping places) and the **privacy/consent + moderation** layer
  (taste + location are sensitive; sharing must be explicit opt-in). Until then,
  the per-user taste vectors quietly accumulate — the raw material the graph runs
  on.

## Integrations & exit

- **Yelp** — data-in via Fusion API (near-term, easy).
- **Facebook** — its Graph API for places/check-ins is heavily restricted
  post-2018; realistically limited. Don't design around it as a core input.
- **Google** — already the data spine; there's no API to write back to a user's
  saved places (see the deep-link note in the README). A Google acquisition is a
  plausible *outcome* of owning the taste layer they lack — but it's an exit
  thesis, not a build task. Build the moat; the outcomes follow.

## The interface (near-term)

The recommender is proven; the next leap is making it *feel* like Muninn. The home
surface becomes an **interactive globe** — your accumulated taste made visible as
glowing places on a world you can flick, zoom into, and "search this area" from,
with a raven that flies out on every lookup. Aesthetic: **Studio-Ghibli mythical ×
modern digital** — a painterly, storybook globe/atmosphere/raven under crisp modern
UI. Built isolated at a `/beta` path. The globe.gl and two-engine prototypes were
superseded by a single **MapLibre globe** — one engine from planet to street,
which is what made the transitions feel like Google Earth. The aesthetic
settled on an **18th-century chart**: cream paper everywhere, everything else
drawn on top in ink. Current state and remaining work live in
[`ROADMAP.md`](ROADMAP.md); the next artistic pass is briefed in
[`SHADERS.md`](SHADERS.md). `GLOBE.md` is kept as history of the earlier
eras. This is the
piece that turns a useful engine into something people *want* to open — and the
identity a taste-graph product would carry.

## Later-version ideas (queued from the beta build, 2026-09)

- **Weather-aware reasoning**: fold current/forecast weather into ranking —
  clear skies favor patios, rain favors cozy interiors. Needs a free weather
  API + a prompt extension; cheap once the ranker has a slot for context.
- **Destination mode**: Muninn searches the whole globe for 100%-match,
  "worth the trip" places (Dry Tortugas-tier), with a time-worth weighting
  (a week at Yellowstone vs. two hours at a bar), then strings smaller
  matched stops along the route — the bridge toward trip planning without
  becoming a trip planner.
- **First-class custom places**: add places Google doesn't know (the Tokyo
  alley bar) with photos + geotags; ownership and visibility controls
  (private / friends / public) so secret spots stay secret while still
  feeding the owner's taste profile and rankings.
- **Social layer / leaderboard**: share maps and places, compete on
  towns-explored or taste-match streaks — the engagement layer that rides on
  the Phase-3 taste graph once user density exists.

## What stays true across all of it

Muninn's job is the same at every scale: **go out, gather what's actually good for
this specific person, come back and report** — built on accumulated memory of
taste. Multi-source data makes the "what's good" sharper; the taste graph makes
"for this person" sharper. Everything serves those two.
