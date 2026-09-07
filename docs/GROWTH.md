# Muninn — Growth, Monetization & Pre-Launch Legal

Requested by Ian (HIGH IMPORTANCE). Written by Fable 2026-09-01. Honest,
opinionated, and sized for a solo founder funding this out of pocket.

## Part 1 — The path to monetization & growth

### The order of operations (don't skip steps)

1. **Retention before acquisition.** Do not spend a dollar on ads until the
   handful of friends already on Muninn open it *unprompted* in a normal week.
   Advertising leaky products burns money; the beta's job right now is to make
   5 people check it every trip. The signature moments (raven search, taste
   page) are retention features — ship those first.
2. **Positioning is your moat, say it everywhere:** *Google tells you what's
   popular. Muninn remembers what* you *love — and finds it anywhere.* Every
   screen, tweet, and onboarding line should carry this. You are not competing
   with Google Maps; you're the taste layer it doesn't have.
3. **Built-in viral loop = the shareable taste card.** The taste-page
   archetype card (ROADMAP batch 8) is the growth engine: people share "what
   kind of traveler am I" images; each share advertises Muninn for free.
   Same mechanic as Spotify Wrapped. Build the share-as-image button early.
4. **Organic channels before paid** (in order of fit):
   - Short-form video: "I flicked a globe and a raven found me the best bar in
     Nashville" — the app *demos beautifully*; screen recordings are the ad.
   - Reddit (r/travel, r/solotravel, city subs) — as a builder sharing a free
     tool, not as ads.
   - The niches from VISION: RV/overlanding, liveaboard/boating forums, road
     warriors — small, underserved, and exactly Muninn's use case.
5. **Waitlist + invite codes** when interest exceeds capacity — throttles your
   API costs AND creates scarcity. A simple landing page with the globe demo
   video and email capture.
6. **Paid ads** only after: retention is real, a price exists, and you know
   LTV > acquisition cost. That's months away; don't pre-spend.

### Pricing (freemium — the standard and correct shape here)

- **Free**: N searches/month (e.g. 15), standard ranking, full saving/import.
  Free users build taste graphs = future moat value.
- **Muninn Pro, ~$4–6/mo or ~$40/yr**: unlimited searches, Deep Context always
  on, search radius/lenses, taste card refreshes, (later) Destination mode.
- Trigger to launch pricing: the moment strangers (not friends) are active
  weekly.

### Unit economics (why this works at small scale)

Per active user per month, roughly: Claude ranking ~15 searches × ~$0.02 =
$0.30 (Haiku default; deep-context searches ~$0.05–0.10 each — cap free users);
Google Places within free tier until real scale *if* the resolution cache stays
on and imports stay approval-gated (already built — that $200 bill stays
unrepeatable); Supabase free → $25/mo Pro somewhere around thousands of users;
Render $7–25/mo. **~100 free users ≈ tens of dollars/month; one Pro subscriber
covers ~10 free users.** Keep the free tier capped and this never runs away.

### On selling taste data: don't.

Your instinct is right, and it's also strategically correct, not just ethical:
- Trust IS the product. "Muninn knows my taste" only works if users believe
  that knowledge serves them. One "sells your data" headline kills the app.
- Location + preference history is close to the most sensitive data class;
  selling it individually is a GDPR/CCPA minefield.
- The defensible version, YEARS away and opt-in only: aggregated, anonymized
  market insights ("what do high-taste travelers seek in secondary markets")
  licensed to tourism boards/hospitality — the B2B angle already in VISION.
  Never individual data, never without explicit opt-in, never before scale.

### Funding

Costs stay near-zero by design; revenue-fund from Pro subscriptions. Don't
raise money to buy users for an unproven retention loop. If the taste-graph
thesis proves out with organic growth, that traction raises on far better
terms later.

## Part 2 — Legal / pre-public-release checklist

Not legal advice; a map of what to handle. Use template services for the beta,
pay a lawyer ~1hr before real scale.

### ⚠️ #1 issue to resolve — Google Places content on a non-Google map

Google Maps Platform ToS **restricts displaying Places API content on maps
other than Google's** and restricts caching (place details generally ≤30 days;
place IDs cacheable indefinitely). Muninn currently pins Google-derived places
on a MapLibre/OSM map and stores enrichment. Options, in order of preference:
1. Re-read the current Google Maps Platform ToS §3.2.3 + Places policies and
   confirm what's actually restricted today (terms have shifted; do this
   before public launch, not after).
2. Mitigations if restricted: store only place_id + user's own notes and
   re-fetch details (allowed pattern); and/or swap the *candidate/display*
   source to an open provider (Foursquare/OSM Overture) while keeping Google
   only for resolution; and/or show Google-derived results in list cards
   rather than pinned.
This is a launch blocker to *investigate*, not necessarily to re-architect —
but go in with eyes open.

### Attribution obligations (mostly already met — verify on screen)

- **OpenStreetMap/OpenFreeMap** (ODbL): "© OpenStreetMap contributors" must be
  visible — MapLibre's attribution control shows it; don't hide it.
- **Google**: "powered by Google" attribution wherever Places data shows.
- **Terrain** (Mapzen/AWS terrarium): attribution string already in the style.
- **NASA GIBS** imagery: public domain; courtesy credit "NASA Worldview" in an
  about page is good form. **Natural Earth**: public domain. **MapLibre** BSD,
  **fonts** OFL — fine.

### Documents you need before strangers sign up

- **Privacy Policy** (legally required — CCPA/GDPR both reach you the moment a
  California/EU user signs up): what's collected (email, saved places, search
  locations, taste profile), why, who processes it (Supabase, Google, Anthropic,
  Render, NASA/Nominatim requests), retention, and user rights. Note that
  saved-places history is effectively location-pattern data — treat it as
  sensitive.
- **Terms of Service**: no warranties, recommendations are suggestions (a bad
  restaurant night isn't a lawsuit), account termination, age 13+ (COPPA),
  arbitration clause.
- **Data deletion path**: db/wipe_account.sql exists — productize it into a
  "delete my account" button before public launch (GDPR requires it).
- **Cookie/storage disclosure**: localStorage session + preferences — one line
  in the privacy policy is enough (no ad trackers = easy mode).
- **Entity**: form an LLC before public launch (~$100–300) so a ToS dispute
  isn't personal. Get an EIN, separate bank account, put the app under it.
- **Trademark**: quick knockout search on "Muninn" in software/travel class
  before spending on brand; the Norse name is used by other software.
- If/when app stores: Apple/Google add their own privacy-label requirements —
  the PWA route dodges this for now.

### Templates to start from (beta-grade)

Termly / GetTerms / iubenda generate serviceable Privacy Policy + ToS for
free-to-cheap. Link them in onboarding ("by continuing you agree…") and the
About page. Upgrade to lawyer-reviewed when money changes hands.
