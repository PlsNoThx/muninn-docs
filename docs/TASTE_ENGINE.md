# The Taste Engine

How Muninn finds and rates places today, an honest read on where that falls
short, and a thesis for the engine that should replace it. Written 2026-09-02
from the code as it runs on `main`, not from the brief.

**Status (2026-09-03):** the v2 engine's server side is built and is the
default (`src/engine/v2/`, switch in `src/engine/index.ts`, v1 preserved in
`src/lib` behind `TASTE_ENGINE=v1` and as the automatic fallback). Shipped:
facets, facet-conditional anchors with explicit/confirmed/rejected states,
dossiers from Google's review fields, town briefs, the pre-rank and shortlist,
the Sonnet ranker with a cached prefix and structured output, the eval
harness (`scripts/eval.ts`), the `/api/anchor` and `/api/weight` endpoints,
and the deep toggle's retirement. Not yet built: the anchor prompts in the UI
(§4.2.1), the weight dial in the UI, Foursquare/Overture as a second
candidate source, Tripadvisor. Run the v2 block at the end of `db/schema.sql`
in Supabase once; without it the engine still works, it just cannot cache.
Decided 2026-09-03: **closed places still count toward taste** (anchors and
the temperament) and are marked "remembered" on their cards; they are never
candidates. Every save builds the place's dossier for everyone, corroborated
on the web when the person wrote a note (`src/engine/v2/onsave.ts`). The
weight dial is a ruled slider at the foot of each saved card.

**Where it stands (end of 2026-09-06):** live on main and verified across
a dozen towns on the phone. Since the 9/3 ship: the schema and cache bugs
that had v2 silently falling back to v1 are fixed; town briefs run (Sonnet
with Anthropic's web search for the sites it can read, plus Brave Search
snippets for Reddit, Eater and the NYT, which block Anthropic's crawler;
Google's Custom Search API is closed to new customers); the ranker scores
every shortlisted place and the sheet tiers at 70 / 50 instead of showing
six; reasons may only name the person's own places; anchors are chosen per
facet, classified by type or by name, merged across favourite/want-to-go
twins, and a general search covers every kind on the shortlist. Speed is
15–25 s cold and 8–12 s warm, bounded by the models' own throughput at the
hour (ROADMAP, Claude API facts). Open: the anchor prompts in the UI, the
eval baseline, a second candidate source, the tasks in PROGRESS.md.

Companion docs: [`VISION.md`](VISION.md) (why taste is the moat),
[`ARCHITECTURE.md`](ARCHITECTURE.md) (multi-user data model, scenario
anchors), [`GROWTH.md`](GROWTH.md) (unit economics, the Google ToS question).

---

## 0. The thesis in one page

**Taste is a set of exemplars, not a distribution.** The current engine
summarises 863 favourites into percentages ("Mexican 3.8%, price 1.8/4") and
asks a model to rank Google's top twelve results against those percentages. It
works better than Google Maps because the prompt is good, but the model is
ranking on priors, not on evidence: it never sees what any place is actually
like, and it never sees which of your favourites is the right comparison for
this search.

The engine that replaces it rests on six ideas:

1. **Exemplars over statistics.** Represent taste as a weighted set of places,
   each described richly, and retrieve the *relevant* exemplars for every
   query. Ian's note is the core of this: the anchors for a coffee search are
   coffee places you love, not There There. Anchors become
   **facet-conditional** (coffee, casual food, dinner by cuisine family,
   drinks, outdoors, shops, lodging) and are chosen per search.
2. **Describe every place once, reuse forever.** A user-agnostic **place
   dossier** (format, signature, vibe tags, price, caveats, sources) is built
   once per place and cached globally. Favourites and candidates share the
   schema, so "does this candidate resemble your anchors" becomes a
   like-for-like comparison instead of a guess from a name and a type token.
3. **Retrieve, then reason.** A deterministic pre-rank (facet match,
   credibility-weighted rating, price-band distance, tag overlap with the
   facet anchors, town-brief mentions) cuts thirty-six candidates to twelve.
   The model ranks twelve with real context. Smaller call, better answer.
4. **Research the town, not the query.** One deep, web-grounded call per town
   ("what locals and critics say, the twenty places they name") is cached for
   months and read by every later search there for a fraction of a cent. Deep
   quality at shallow cost is a caching problem, and the right cache key is
   the place and the town, never the query. **Every search is deep**; the
   toggle goes away (§5.8).
5. **Learn from behaviour.** Saves, passes with reasons, relists, "been
   there" graduations and a single optional weight dial feed exemplar weights.
   Today none of that changes the owner's card, which is a file from August.
6. **Measure it.** Hold out favourites and ask whether the engine would have
   found them. Without a recall number every prompt change is a vibe.

Cost follows from the shape: cached prefixes, shortlists, batch pre-warming and
first-party review fields make a Sonnet-quality search cheaper than today's
Haiku call, and a town's deep research is paid once.

---

## 1. What taste is made of (the signals we hold)

| Signal | Where it lives | Strength | Used today? |
|---|---|---|---|
| Favourites (visited, liked) | `taste_profile_enriched.csv` (863 resolved) + `added_places` | Strongest | Yes: profile card statistics, local calibration list, exclusion from picks |
| Want-to-go (aspirational) | same CSV (1,053 resolved) + `added_places` | Weak but real | Excluded from picks only. Never a signal |
| Personal notes on a place | `note` column / `added_places.note` | High per place, rare (1 of 863 favourites has one) | Shown in local calibration list |
| Dislikes with a reason | `disliked_places` | High, teaches a pattern | Yes: excluded by id, reasons rendered as a steer |
| Stated likes / avoids | `profiles.likes/avoids` | Medium, soft | Yes: soft prompt block; likes also seed search keywords |
| Dietary rules | `profiles.dietary` | Hard constraint | Yes: hard prompt block |
| Home base | `profiles.hometown_*` | Bias for resolution | Onboarding resolution only |
| Anchors ("exactly right") | `ANCHOR_SPECS` in code (5 names) / onboarding favourites | Highest | Yes, but global, fixed, and two of five never matched |
| Behaviour (saves, passes, relists, views) | `added_places`, `disliked_places` | Medium, free to collect | Stored, never fed back into the card |
| Geography of saves | derived | Medium (where you go, what you like *there*) | One line of state percentages |

The last column is the gap. Most of what we know about Ian's taste is stored
and not used, and the part that is used has been flattened into percentages.

---

## 2. How it works today, end to end

```
 title (CSV)            ┌───────────────┐
 ─── enrich.ts ───────▶ │ enriched CSV  │──▶ build-profile.ts ──▶ taste_profile.md (card)
 Text Search per title  │ 1,942 places  │                          taste_profile.json (stats)
                        └───────────────┘

 search ─▶ composeRequest ─▶ /api/recommend ─▶ planSearch (Haiku) ─▶ findCandidates (Text Search × keywords)
                                                                     ─▶ filter saved / disliked / avoid
                                                                     ─▶ gatherResearch (cache; deep: Sonnet + web_search)
                                                                     ─▶ rankCandidates (Haiku, or Sonnet when deep)
                                                                     ─▶ attachPlaces ─▶ cards
 coin / lens ─▶ /api/nearby-recommend ─▶ Text Search × lens.query (biased to reach) ─▶ same filters ─▶ rank (Haiku)
```

### 2.1 Enrichment: from a name to a place

`scripts/enrich.ts` runs one Google Text Search per saved title, picks the best
candidate by name similarity (Jaro-Winkler style, `similarity.ts`), and writes
coordinates, `primary_type`, `types`, `price_level`, `rating`,
`user_rating_count`, `formatted_address`, `business_status`, a
`match_confidence` and a `resolution_status`. Resumable, rate-limited.

What the data actually looks like after enrichment:

| Measure | Value |
|---|---|
| Rows | 1,942 (868 favourite, 1,065 want-to-go, 9 both) |
| Resolved / low confidence / not found | 1,804 / 112 / 26 |
| Favourites with a price level | 434 of 863 (50%) |
| Favourites with a personal note | 1 |
| Favourites permanently closed | 29 (plus 8 temporarily) |
| Favourite review-count quartiles | 226 / 690 / 1,849 |
| Countries | USA 740, Japan 66, Mexico 14, Guatemala 7 |

Two things worth knowing. Half the favourites have no price level, so the
"price band 1.8" in the card is measured on the half that do, which skews to
restaurants. And nothing reads `business_status`: closed favourites still
shape the profile and closed candidates can still be recommended.

### 2.2 The profile builder and the card

`src/lib/profile.ts` (`buildProfile`) rolls each favourite into one of seven
venue categories by Google type token, pulls a cuisine label from
`*_restaurant` types, and computes: venue mix, top cuisines, price mean and
distribution, rating mean/median/buckets, popularity spread (median reviews,
% under 200, % over 1,000), the **naming signal** (share of favourites whose
name shares no word with their cuisine/type: 58.7%), top regions, and the
five anchor matches.

`renderProfileMarkdown` turns that into the card the ranker reads. The card as
of 2026-08-30 says, in full: venue mix (Restaurant 31%, Market/shop 21%,
Cafe 14.5%, Outdoors 13.7%, Bar 6.5%), ten cuisines each under 4%, price
1.8/4, ratings mean 4.5, popularity spread, naming 58.7% branded, eight
regions, the anchors (Aguardente, Turkey and the Wolf, There There matched;
Odette and River Bar not found), and three paragraphs of instructions.

Read it as the model does and the problem is visible. "Mexican 3.8%" is the
top cuisine because 70% of favourites are not cuisine-typed restaurants at all
(coffee shops, bookstores, museums, clothing stores, hotels). The card can say
"this person likes coffee shops" (79 of them, the single largest primary type)
only as a 14.5% venue share, and says nothing about *which kind* of coffee
shop. The one genuinely distinctive signal it carries is the naming bias.

For non-owner accounts `buildUserCard` (`usercard.ts`) writes a thinner card
from onboarding favourites (with cuisine tags) and up to forty saved places.
It is, ironically, closer to the right shape: a list of named exemplars with a
short note each.

### 2.3 Understanding the request

- The client composes the request as `lens.request` + `"Specifically: " +
  ask` (`composeRequest`). Lens requests are prose ("restaurants worth going
  out of your way for", "coffee shops, cafés and good bakeries").
- Town search: `planSearch` (`plan.ts`, Haiku, ~300 tokens) turns the text
  into 1–3 `targets` with a concrete Google keyword each, plus a `pairing`
  flag for "X near Y". No request text means keywords come from
  `tasteKeywords`: the owner's top five cuisines as "<cuisine> restaurant",
  plus three comma-split likes, plus "restaurant" and "cafe".
- Near-me: the lens maps to `NEARBY_LENSES` with fixed intent queries
  ("coffee shops", "cafes", "espresso bar") and a category block-list.
- `requestCategories` (`saved.ts`) is a keyword regex that narrows which
  *saved* places are shown for the area. It is separate from the plan, so the
  two can disagree.

### 2.4 Candidate sourcing

- **Town search:** one Text Search per keyword, query string `"<keyword> in
  <area>"`, twelve results each, de-duplicated by `place_id`. Three keywords
  is the norm, so the pool is roughly thirty-six places chosen by Google's
  prominence ranking for that phrase.
- **Near-me:** one Text Search per lens query with `locationBias` to the reach
  circle, then a hard post-filter at 1.15× the reach because bias is only a
  bias. Twelve per query, two or three queries.
- **Paired ("hike with a brewery nearby"):** fifteen of each, every primary
  matched to its nearest companion by haversine, the pair rendered into the
  prompt with the distance.

The field mask is `id, displayName, formattedAddress, location, types,
primaryType, priceLevel, rating, userRatingCount, businessStatus`. On the
Places API (New) that mask bills at **Text Search Enterprise**: `priceLevel`,
`rating`, `userRatingCount` and `businessStatus` are Enterprise fields. That
SKU has a free allowance of **1,000 requests a month**, then **$35 per 1,000**.
The CLAUDE.md assumption of a 10,000-call free tier applies to Essentials
SKUs, which we do not use. A town search is three requests and a lens search
two or three, so the free tier covers about 350 searches a month across all
users, then each search costs about a dime in Places fees. This is the
mechanism behind the surprise bill GROWTH.md mentions.

### 2.5 Filtering

Candidates are dropped if their `place_id` or normalised name matches a saved
place, if the id is in the user's dislikes, or if `avoid.txt` contains a
substring of the name. Near-me also drops candidates whose venue category is
on the lens block-list. Nothing drops closed places. Nothing pages Google for
more when a filter removes most of the pool.

### 2.6 The research layer

`place_research` (Supabase) holds one summary per `place_id`, user-agnostic,
read only if fresher than **45 days**. In deep mode `researchPlaces` sends the
ten uncached candidates with the most reviews to **Sonnet 5 with
`web_search` (max 8 searches)** and asks for one or two grounded sentences
each (signature features, atmosphere, real caveats). Notes are injected as a
`context` field on the candidate in every later rank call, deep or not. The
client also re-runs a weak round (fewer than three picks or best fit under
70) once with deep on.

This is the best idea in the current engine: research keyed on the place and
shared across users. It is just too shallow (one or two sentences), too
narrow (ten places), too short-lived (45 days for facts that mostly change
yearly), and only ever triggered by a live deep search.

### 2.7 Ranking

`rankCandidates` builds a system prompt from: the principles preamble (taste
is defined by this person; cuisine is a lean not a filter; aim for range;
popularity is context; do not recommend calibration places; be honest; never
name a place they have not saved), an optional web-search instruction, the
dietary block, the preferences block (likes, avoids, dislikes with reasons),
and the card. The user message carries area, request, the local favourites
list (up to fifteen, with notes), and the candidates as pretty-printed JSON
(id, name, type, six types, price, rating, review count, address, optional
context). The model returns a JSON array of up to six picks with `why`,
`fit_score` and an optional `caution`.

Model: **Haiku 4.5** by default, **Sonnet 5** when deep (research happens
first; the rank call itself no longer web-searches). Output is parsed by
slicing between the first `[` and last `]`, de-duplicated, then
`attachPlaces` re-binds every pick to a real candidate by id or name and drops
anything unmatched (the fix for a scrambled name/id pair once saving the
wrong place). Near-me falls back to the six best-rated candidates if the
ranker returns nothing.

### 2.8 What comes back and what we learn from it

Save writes `added_places` (favourite or want-to-go, with a note); pass writes
`disliked_places` with a reason; relist moves between lists. The owner's card
is a static file rebuilt by hand from the CSV, so none of this changes what
the ranker believes about Ian. For other users, saved places do enter their
card (as a list of names), which is the closest thing to learning the system
has.

The portrait (`/api/portrait`, Haiku, three sentences) is cached by a hash of
the evidence, which is the right pattern for every derived artefact.

### 2.9 What a search costs today

Estimates from the prompt shapes above (Haiku 4.5 $1/$5 per MTok, Sonnet 5
$2/$10, web search $10 per 1,000 searches, Text Search Enterprise $35 per
1,000 past the free 1,000).

| Path | Model calls | Tokens (in / out) | Claude | Places (past free tier) |
|---|---|---|---|---|
| Town search, default | plan + rank on Haiku | ~4,800 / ~600 | ~$0.008 | ~$0.105 (3 requests) |
| Near-me lens | rank on Haiku | ~4,000 / ~500 | ~$0.007 | ~$0.07–0.105 |
| Deep search | plan + research (Sonnet, 8 searches) + rank (Sonnet) | ~25,000 / ~2,000 | ~$0.15–0.20 | ~$0.105 |

Two observations. Past the free tier, **Places fees dwarf Claude fees** on a
default search by more than ten to one. And the Haiku call never benefits
from prompt caching, because Haiku 4.5's minimum cacheable prefix is 4,096
tokens and our stable prefix (preamble + card + blocks) is roughly 2,000.

---

## 3. Honest assessment

What is strong: the principles in the prompt (taste is defined by the
person, popularity is context, never invent a place), the reconciliation step
that makes a pick and the place it acts on the same record, the shared
place-research cache, the dislikes-with-reasons pattern, dietary as a hard
constraint, and the near-me reach post-filter. Keep all of it.

What is weak, with the evidence:

1. **The card is statistics, not taste.** Percentages of coarse buckets carry
   almost no information about *which* coffee shop or *which* Mexican place.
   The model's "why" lines are generic because the card gives it nothing
   specific to cite except three anchor names.
2. **Anchors are global, fixed in code, and half missing.** Five names in
   `ANCHOR_SPECS`, two unmatched in the export. And, as Ian noted, a burger
   anchor is the wrong yardstick for a coffee search. There is no
   facet-conditional selection at all.
3. **Every favourite weighs the same.** A place saved once at an airport in
   2019 counts exactly as much as Aguardente. ROADMAP item 10 already names
   the fix (a single weight dial plus passive signals).
4. **The ranker sees names and type tokens, not places.** Without research it
   knows a candidate as `cocktail_bar, price 2, 4.6★, 812 reviews`. It ranks
   on its priors about that name. The naming-signal bias is literally the
   only taste signal it can apply to a candidate.
5. **The pool is Google's prominence list.** Twelve per keyword, no paging,
   no second source. The places Muninn exists to find (no big sign, 180
   reviews) are exactly the ones prominence ranking buries under the
   500-review brunch place.
6. **Deep is a per-query expense.** Ten places, one or two sentences, 45-day
   TTL, only when someone toggles deep. The knowledge does not compound
   across a town.
7. **Want-to-go is wasted.** 1,053 places that say where taste is heading are
   used only as an exclusion list.
8. **Behaviour never reaches the card.** Saves, passes, relists and notes are
   stored and ignored for the owner.
9. **No notion of place-type-of-town.** Ian's list is 29% Florida and 8%
   Japan; what he likes in a beach town differs from what he likes in
   Portland. Nothing models that.
10. **Closed places leak** in both directions because `business_status` is
    fetched and never read.
11. **No evaluation.** There is no way to know whether the last prompt edit
    made recommendations better. Every change so far has been judged by
    feel on a phone.
12. **The cost model is wrong in the docs.** Enterprise SKU, 1,000 free
    requests, $35 per 1,000 after. Fine for one user, a real line item at a
    hundred.
13. **Parsing is fragile.** Bracket slicing instead of structured output;
    the model can and occasionally does return prose or a scrambled pair.

---

## 4. The thesis, in detail

### 4.1 Exemplars over statistics

Keep the statistics as a short "temperament" paragraph (price band, quality
bar, naming bias, geography), because they are cheap and true. Move the
weight of the profile onto **named places with descriptions**, chosen per
search. The model reasons well from "this person's coffee anchors are Kafe
Kubana (Cuban window, cortaditos, standing counter), Bandit (third-wave,
quiet, pastry-forward) and Ludlow (roastery, industrial, no laptops)"; it
reasons poorly from "Cafe / coffee 14.5%".

### 4.2 Facet-conditional anchors (Ian's note, made concrete)

Define a small fixed set of **facets**. They are coarser than cuisine and
finer than the four lenses:

| Facet | Covers |
|---|---|
| coffee | coffee shops, cafés, tea |
| bakery-sweet | bakeries, dessert, ice cream |
| breakfast-brunch | breakfast and brunch restaurants |
| casual-food | counters, tacos, burgers, pizza, sandwiches, BBQ |
| dinner:latin / italian / japanese / american / seafood / asian-other / other | sit-down restaurants by cuisine family |
| drinks:cocktail / beer / wine / pub | bars by kind |
| outdoors | parks, trails, beaches, water |
| sights | museums, galleries, landmarks |
| shops:books / records / clothing / home / market | retail |
| lodging | hotels, inns |

Every saved place gets a facet at enrichment time from `primary_type` and
`types` (deterministic, the existing `venueCategory` extended one level).
Every query gets a facet from the lens, the plan targets and the ask text
(add a `facet` field to `planSearch`'s JSON; it already classifies).

**Anchor selection for a query, in order.** The rule is that the engine
always has anchors: explicit ones when the user has given them, inferred ones
when not, and a graceful blend in between. Nothing here is required of the
user.

1. **Explicit anchors** the user has tagged for that facet (§4.2.1 covers how
   we ask). Up to eight per facet.
2. **Confirmed anchors:** inferred ones the user has tapped to agree with.
   They rank just below explicit.
3. **Inferred anchors:** favourites in the facet, scored by
   `weight × credibilityRating × recency × behaviour` where `weight` is the
   1–5 dial (default 3; onboarding favourites 5), `credibilityRating` is the
   Bayesian rating already in `saved.ts`, recency decays over about three
   years, and `behaviour` lifts places that were relisted, noted, or opened
   repeatedly. Fill the facet to eight, forcing geographic spread (at most
   two from one metro) unless the query is in that metro. Anything the user
   has marked "not quite" is excluded and never re-suggested.
3. **Local anchors:** favourites in the facet within a radius of the query
   (today's `localFavorites`, switched from address-string matching to a
   distance), listed separately as "what they already love nearby".
4. **Adjacent-facet fallback** when a facet has fewer than three anchors
   (coffee falls back to bakery-sweet and breakfast-brunch; dinner:seafood to
   dinner:american and casual-food), flagged as adjacent so the model does
   not over-read them.

Of the eight, the five or six with the best score go to the prompt with their
dossier lines; more than that dilutes the comparison. Each anchor carries its
provenance (explicit, confirmed, inferred, adjacent) so the model can weigh a
guess below a statement.

#### 4.2.1 Asking for anchors without requiring them

The taste store must work at zero answers, so every prompt below is one tap,
skippable, and rate-limited. The design principle is to ask at the moment the
answer is already in the user's head, never as a form.

| Moment | The ask | Cost to the user | Rule |
|---|---|---|---|
| Onboarding | After the three all-time favourites, one optional screen: "One for coffee, one for dinner, one for a drink?" Three chips, each a name field with a suggestion list; pre-filled from an import when there is one | Three names, or skip | Skippable; never blocks finishing onboarding |
| After a save | On the saved card, a single chip: "One of your coffee anchors?" with **yes** / **not quite** | One tap | Shown only when that facet has fewer than five explicit anchors; at most one per session |
| At search time | Under the results, one line: "Guessing your coffee taste from Bandit and Kafe Kubana. Tap to adjust." Tapping opens a chip sheet of the top eight inferred favourites in that facet, toggle to confirm or reject | Zero taps to ignore, one per chip to confirm | Only when the facet has no explicit anchors; never on the first search of a session; never while the keyboard is up |
| After an import | "We found 19 places in Providence. Which would you send a friend to?" Chips, multi-select, skip | Optional | One question per import, capped at twelve chips |
| Taste page | A per-facet row of anchors: explicit in solid ink, inferred in a lighter hand; tap to promote or demote | Optional, for the curious | Always available, never prompted |
| Passive promotion | None. A favourite that is relisted, noted, weighted 5, or opened three times becomes **inferred-strong** and behaves like a confirmed anchor | Nothing | Automatic |

Rejections matter as much as confirmations: a "not quite" is stored so the
place is never suggested again for that facet and its inference score is
halved elsewhere. Stop asking about a facet once it holds five explicit or
confirmed anchors.

Data: `added_places.anchor_facets text[]` for explicit tags, and an
`anchor_state` table keyed on `(user_id, facet, place_id)` with `state` in
`explicit | confirmed | rejected` and `asked_at`, so the prompts above can
be rate-limited from one place. Inferred and inferred-strong are computed,
not stored.

The card then has two parts: a **durable card** (temperament paragraph,
dietary, likes/avoids, disliked patterns) that is byte-stable and cached, and
a **facet card** ("exactly right for this kind of place", five anchors with
dossier lines, plus local anchors) that changes per query and rides in the
user message. This is also the ARCHITECTURE.md "scenario anchors" idea with
the scenario made concrete and inferable.

### 4.3 The place dossier

One record per `place_id`, global, user-agnostic, built once and refreshed
slowly. Favourites, want-to-go, dislikes and candidates all get the same
schema, so every comparison is like-for-like.

```json
{
  "place_id": "ChIJ…",
  "name": "Aguardente",
  "facet": "drinks:wine",
  "cuisine_family": "portuguese",
  "format": ["sit-down", "bar seating", "small plates"],
  "price_band": 2,
  "signature": ["natural wine list", "tinned fish", "grilled sardines"],
  "vibe": ["dim", "neighbourhood", "date-night", "no reservations", "quiet-ish"],
  "best_for": ["late dinner", "solo at the bar"],
  "chain": false,
  "indie_signal": "owner-run, opaque name",
  "caveats": ["cash-forward", "closes early Sunday"],
  "status": "open",
  "sources": {"google_reviews": 5, "google_summary": true, "web": 2},
  "confidence": 0.8,
  "updated_at": "2026-09-02"
}
```

Where the dossier's facts come from, cheapest first:

1. **Google Atmosphere fields on the same Text Search request.** Adding
   `reviews` (five most relevant), `reviewSummary`, `generativeSummary`,
   `editorialSummary` and `priceRange` to the field mask moves the request
   from Enterprise ($35 per 1,000) to Enterprise + Atmosphere ($40 per
   1,000). For fourteen percent more per request, every one of the twelve
   candidates arrives with five reviews and a Google-written summary. That is
   the cheapest grounded context available anywhere, and it is first-party.
2. **Tripadvisor Content API** (5,000 free calls a month, five reviews per
   location) as a second opinion for restaurants and sights.
3. **Web search on Sonnet 5**, steered by the instruction toward the
   sources that carry taste (The Infatuation, Michelin, Time Out, Punch,
   local papers and alt-weeklies) with a short `blocked_domains` list for
   aggregators, only for shortlist places that survive pre-rank and have no
   web-sourced dossier yet. An `allowed_domains` list is not an option:
   the tool rejects any list naming a site that blocks Anthropic's crawler,
   and Eater, Reddit and the NYT all do (ROADMAP, Claude API facts). Those
   sites reach the brief as Google search snippets instead: the writer is
   handed the titles and excerpts from three Google queries before it
   searches, the same route the chat app takes through Brave's index.

Extraction runs on **Haiku 4.5** with a structured-output schema (about
3,000 input tokens per place with five reviews and the summaries, so about
$0.004 each, half that via the Batch API). Dossiering the whole 1,942-place
memory costs under $10 in Claude fees and, at 1,000 free Atmosphere requests
a month, nothing in Places fees if spread over two months.

Legal note (GROWTH.md): Google allows caching place IDs indefinitely and other
place data for up to thirty days. The dossier stores a *derived* summary, not
the reviews, and re-reads Google for status. Keep review text out of the
table and re-read the ToS before public launch.

### 4.4 Retrieve, then reason

Pre-rank every candidate deterministically before any model sees it:

```
score = 1.0 × facetMatch            (0/1, from types)
      + 0.8 × credibilityRating     (Bayesian, normalised 0–1)
      + 0.5 × priceCloseness        (1 − |price − user band| / 4)
      + 0.7 × tagOverlap            (Jaccard of dossier vibe/format tags vs the facet anchors' tags)
      + 0.6 × townBriefMention      (named in the town brief, §5.2)
      + 0.3 × indieSignal           (only if the user's naming bias is high)
      − 0.4 × distancePenalty       (near-me only, beyond half the reach)
      − ∞   × closedOrDisliked
```

Take the top twelve to the model with their dossier lines and the facet card.
Twelve candidates with real descriptions beat thirty-six with type tokens,
and the call is smaller. Weights start as above and get tuned against the
eval in §4.6; the point is not the exact numbers, it is that the model spends
its attention on discrimination between plausible places, not on discarding
the implausible.

### 4.5 Learning from behaviour

- **Weight dial** (ROADMAP 10): one 1–5 control on the saved card, shown
  only after a save, default 3. Stored as `added_places.weight`. It is the
  single most valuable input Ian can add and takes a second per place.
- **Passive weights:** relist to favourite +1, "been there" graduation from
  want-to-go +1, a written note +1, repeat card opens +0.5 (capped), pass
  −(exclude) with the reason feeding the disliked-pattern block.
- **Want-to-go as a signal:** pattern-mine it at half weight for the
  temperament paragraph and as a *lower-tier* anchor source when a facet is
  thin. Aspiration is information.
- **Rebuild the card on write**, not by hand: any save, pass, weight or note
  invalidates the durable card for that user (portrait-style hash), and the
  next search rebuilds it. This is what turns a file into a profile.

### 4.6 Evaluation: would Muninn have found it?

An offline harness, `scripts/eval.ts`, that runs entirely on cached data plus
cheap calls:

- **Leave-one-out recall.** For each favourite with at least three other
  favourites in the same metro and facet, remove it from `saved`, run the
  pipeline for that metro and facet, and record whether it appears in the top
  six (recall@6) and where (mean reciprocal rank). Two hundred cases on Haiku
  cost about $2; on Sonnet about $4. Run it on every prompt or weight change.
- **Pairwise judgements.** In the app, after a search, occasionally show two
  picks and ask "which first?". Each answer is a label; a hundred of them tune
  the pre-rank weights better than any amount of prompt wording.
- **Held-out towns.** Keep a few metros out of any tuning and report them
  separately so the weights do not overfit to St. Petersburg.

Without this the engine cannot be "polished"; it can only be changed.

---

## 5. Deep results without the deep cost

Ian's explicit ask. The answer is that depth and cost are only linked when the
expensive work is keyed on the *query*. Key it on durable things and pay once.

### 5.1 Cache at the right granularity

| What | Key | Lifetime | Who pays | Who benefits |
|---|---|---|---|---|
| Place dossier | `place_id` | 90 days; status re-checked from Google on read | the first user to surface it, in batch | everyone, forever |
| Town brief | `canonicalTown(metro) + facet-group` | 90–120 days | the first search in a town | every later search there |
| Durable card | `user + hash(evidence)` | until evidence changes | the user's next search | the user |
| Rank result | `user + query + candidate set hash` | 24 hours | – | repeated taps, relaunches |

Today only the place note is cached, at 45 days and one sentence. Extending
the TTL and the depth is nearly free; the town brief is the new piece.

The brief's metro is canonicalised before it is keyed (`canonicalTown` in
`src/lib/geo.ts`: lowercase, no punctuation or accents, saint/fort/mount
abbreviated, the country dropped, a trailing state code written out), so
"St pete FL", "Saint Petersburg, Florida" and "St. Petersburg, FL, USA" read
and write one row. The beta client resolves a typed name to a settlement and
sends "City, Region"; a nickname the geocoder does not know is expanded there,
not here (ROADMAP, "One town, one brief").

### 5.2 Town briefs: research the town, not the query

One Sonnet 5 call with `web_search` per metro and facet group (food and
drink; coffee and bakeries; outdoors and sights; shops), asked for: the
neighbourhoods that matter, the fifteen to twenty-five places locals and
critics actually name, one line each on why, and any "was great, has
declined" warnings. About eight searches and 20k tokens: roughly **$0.15 to
$0.25 once**. Stored in `town_research` with the source URLs.

Then the brief's names are resolved to `place_id`s with **Text Search
Essentials (IDs Only)**, which has an unlimited free allowance. Those ids join
the candidate pool for every later search in the town, whether or not Google's
prominence ranking would have shown them, and the brief text rides in the
prompt as cached context. A default search in a briefed town gets deep-search
quality for the cost of a cache read. This is the largest single lever in the
document.

Pre-warm briefs for the metros with the most saved places (Ian's own list
names them) and for any town searched twice.

### 5.3 First-party review fields before web search

Order the grounding sources by cost per place:

| Source | Cost per place (marginal) | Depth | Notes |
|---|---|---|---|
| Google Atmosphere fields on the search request | ~$0.0004 (12 per $0.005 uplift) | 5 reviews + AI summary + editorial line | first-party, structured, same call we already make |
| Tripadvisor Content API | $0 within 5,000/month | 5 reviews, rating | restaurants and sights |
| Web search, Sonnet 5, domain-restricted | ~$0.01–0.03 (a search is $0.01 plus tokens) | critics, local press, Reddit threads | reserve for shortlist places and town briefs |
| Yelp Fusion | $0.008–0.015 per call, reviews only on Plus and above, three reviews | US-centric | paid since 2024; marginal value over Google for our purpose |

Web search stays in the engine, but it stops being the first resort.

### 5.4 Prompt caching, done right

The stable prefix should be, in order: tools (none, or a fixed list), the
principles preamble, the durable card, the dietary block, the preferences
block, with `cache_control` on the last of those. Everything that varies per
search (facet card, town brief excerpt, candidates) goes in the user message.

The catch measured above: Haiku 4.5 will not cache a prefix under 4,096
tokens, and ours is about 2,000. Two ways out:

- **Rank on Sonnet 5.** Its minimum is 1,024 tokens and a cache read costs
  $0.20 per MTok. With a cached prefix and a twelve-place shortlist the
  Sonnet call is roughly: 2,000 cached × $0.20/M + 1,000 uncached × $2/M +
  500 out × $10/M ≈ **$0.007**, about what the Haiku call costs today, for
  a markedly stronger ranker. Use the one-hour TTL when a user is in a
  session of searches.
- **Or grow the stable prefix past 4,096** by keeping the facet anchors'
  dossiers in the system prompt per facet (one cache entry per user per
  facet). Workable, but it multiplies cache writes.

Recommendation: Sonnet 5 for ranking, Haiku 4.5 for planning, facet
classification and dossier extraction. Confirm with `cache_read_input_tokens`
in the logs; a zero there means a silent invalidator (a timestamp, unsorted
JSON keys, a varying tool list).

### 5.5 Make the model call small

- Twelve candidates, not thirty-six (§4.4).
- Compact candidate rows: one venue label instead of six type tokens, no
  address (the model never needs it), dossier line instead of raw context.
- **Structured outputs** (`output_config.format` with the pick schema)
  instead of bracket slicing: no parse retries, no scrambled fields.
- `max_tokens` sized to six picks (about 600), and `effort: "low"` on models
  that support it; ranking twelve described places is not a hard reasoning
  task once the retrieval is right.
- Never re-rank on the client for a "weak round": weak rounds should trigger
  a *wider candidate pool* (page two, second source, town brief), not the
  same pool at higher cost (§5.8).

### 5.6 Batch everything nobody is waiting for

Dossier extraction for saved places, town briefs for the top metros, portrait
regeneration, weekly status re-checks: all through the **Batch API at 50%
off**, which stacks with cache reads. A nightly job that dossiers every
place saved that day and briefs every town searched that day keeps the live
path on cache hits.

### 5.7 Embeddings, or not

Anthropic does not offer an embeddings endpoint (Voyage AI is the partner it
points to). The pre-rank in §4.4 uses tag overlap instead, which is
explainable and needs no extra vendor. Revisit embeddings only when the
taste graph (VISION Phase 3) needs user-to-user similarity at scale; dossier
tags will serve as the vector basis then too.

### 5.8 Every search is deep: retiring the toggle

Users will expect every search to be deep; that expectation is the moat. So
the deep toggle, the `deep` flag on `/api/recommend` and the client's
"digging deeper" retry all go. What replaces them:

- **Dossiers and briefs are the default path.** A search in a briefed town
  with dossiered candidates is deep by construction and costs a cache read.
- **First search in an unbriefed town** runs a bounded synchronous fill:
  Atmosphere fields on the candidate request (always), plus at most three web
  searches on Sonnet 5 for shortlist places with no dossier. About $0.05 and
  five to eight extra seconds, once per town. The UI says so: "First time
  here. Muninn is reading up on Asheville." The same request fires the town
  brief job in the background, so the second search in that town is fast and
  fully briefed.
- **A weak round widens the pool instead of re-ranking it:** page two of the
  Google search, the second candidate source, the brief's names. Never the
  same thirty-six places at a higher price.
- **"Refresh"** survives as a quiet action on the results sheet that re-runs
  the town brief when a user thinks it is stale. That is the only manual
  control left, and most people will never touch it.

The research code (`researchPlaces`) stays; it becomes the shortlist gap
filler and the batch dossier builder rather than a mode.

### 5.9 The budget, before and after

Per default town search, past the free tiers, order-of-magnitude:

| | Today | Proposed |
|---|---|---|
| Places | 3 × Enterprise ≈ $0.105 | 2 × Enterprise + Atmosphere ≈ $0.08, plus free ID-only resolves |
| Planning / facet | Haiku ≈ $0.001 | Haiku ≈ $0.001 |
| Grounding | none (or $0.15 deep, per query) | dossiers from the same Places call, Haiku batch ≈ $0.02 for new places; town brief amortised ≈ $0.001 |
| Ranking | Haiku, uncached ≈ $0.007 | Sonnet 5, cached prefix, 12 places ≈ $0.007 |
| **Total** | **≈ $0.11 default, ≈ $0.26 deep** | **≈ $0.11 in a briefed town, ≈ $0.16 the first time in a new one; always deep** |

The cost does not move; the quality does. And the Places line, which
dominates both columns, is the one to attack next with open candidate sources
(§6) once a second source can carry the first pass.

---

## 6. The other sources

| Source | Access (Sep 2026) | What it adds | Verdict for Muninn |
|---|---|---|---|
| **Google Places, Atmosphere fields** | Same API, Enterprise + Atmosphere SKU, 1,000 free then $40/1k | 5 reviews, AI review summary, editorial summary, price range | **Adopt first.** Cheapest grounded context, already in the call path |
| **Tripadvisor Content API** | Self-serve, 5,000 free calls/month, then pay-as-you-go; 5 reviews and 5 photos per location | Independent rating and reviews, strong for sights and hotels | Adopt for dossiers of sights, lodging, and restaurant second opinions |
| **Foursquare OS Places** | Free, Apache 2.0, 100M+ POIs, monthly Parquet on S3 (no ratings or tips) | An open candidate pool with categories and coordinates, no per-request fee | Adopt as the **second candidate source** and the way off the Enterprise SKU for the first pass; resolve winners to Google ids with the free IDs-only search |
| **Overture Maps Places** | Free, CDLA-Permissive/Apache, ~74M records (Meta, Microsoft, Foursquare) | Same role as above, fresher in some regions | Either/or with Foursquare OS; pick one to start |
| **Reddit** | Free for non-commercial at 100 qpm with prior approval; commercial is $0.24 per 1,000 calls with a 2–4 week manual review and no guaranteed approval | The best "where locals actually go" text there is | Do **not** build on the Data API. Reddit blocks Anthropic's crawler, so `web_search` cannot read it. Reached instead as **search-engine snippets**: three `site:reddit.com` and critic queries per town brief through the Brave Search API (the index the Claude chat app itself uses), titles and excerpts only, fed to the brief writer as evidence (`src/lib/websearch.ts`). $5 per 1,000 queries, $5 a month free. Google's Custom Search JSON API was the first choice and is closed to new customers (retires 2027-01-01) |
| **Michelin Guide** | No public API; third-party scrapers exist | Stars, Bib Gourmand, Green Star; Bib Gourmand is the tier that matches a 1.8/4 price band with a 4.5 quality bar | Use `web_search` restricted to guide.michelin.com in town briefs; store the distinction as a dossier tag |
| **Eater, The Infatuation, local press** | Web | Critic voice, "38 essential" lists, neighbourhood context | `web_search` in town briefs, steered by the instruction; these are what make a brief good. Eater and the NYT block the crawler and arrive as Google snippets instead; The Infatuation, Michelin, Time Out and local papers are read directly |
| **Yelp Fusion** | Paid since 2024: Starter $7.99, Plus $9.99, Enterprise $14.99 per 1,000 calls; 30-day trial; reviews only on Plus and above (3 per business) | Categories, price, review counts, some review text | Skip for now; Google Atmosphere and Tripadvisor cover it cheaper |
| **Facebook / Instagram** | Graph API places and check-ins heavily restricted since 2018 | Social proof | Not viable as a data source; ignore |
| **Beli, Resy, OpenTable** | No usable public APIs | Reservation demand | Ignore |

Design rule from VISION.md still holds: every source normalises to one place
identity (Google `place_id` where it exists, else coordinates + name), and
conflicts are reconciled in the dossier, not in the ranking prompt.

---

## 7. Proposed architecture (v2)

```
                ┌──────────────── taste store (per user) ────────────────┐
                │ saved places: list, weight, note, facet, anchor tags   │
                │ dislikes + reasons · likes/avoids · dietary · home     │
                │ durable card (cached by evidence hash)                 │
                └────────────────────────────────────────────────────────┘
                              ▲ writes invalidate      │ read per search
 save / pass / weight ────────┘                        ▼
                                                 ┌──────────────┐
 query ─▶ plan (Haiku: targets, pairing, FACET) ─▶ facet anchors │ (explicit → inferred → local → adjacent)
                                                 └──────┬───────┘
                                                        ▼
        candidates ◀── Google Text Search (+Atmosphere) ─┬─ Foursquare OS / Overture (free pool)
                                                         └─ town brief names → IDs-only resolve (free)
                                                        ▼
        ┌──────── place dossiers (global, 90d) ────────┐   ┌──── town briefs (global, 90–120d) ────┐
        │ Haiku extraction from reviews/summaries;      │   │ Sonnet + domain-restricted web search  │
        │ Sonnet + web for shortlist gaps; Batch nightly │   │ once per metro × facet group           │
        └──────────────────────┬───────────────────────┘   └───────────────────┬───────────────────┘
                               ▼                                               ▼
                    deterministic pre-rank (facet, credibility, price, tag overlap, brief mention) → top 12
                               ▼
                    Sonnet 5 rank: cached prefix (principles + durable card + blocks)
                                  + user turn (facet card, brief excerpt, 12 dossiered candidates)
                                  → structured output: picks with why (citing anchors by name)
                               ▼
                    attachPlaces → cards → save / pass / weight ─▶ back to the taste store
                               ▼
                    eval harness (leave-one-out recall@6, pairwise labels, held-out towns)
```

New tables: `place_dossier` (replaces `place_research`), `town_research`,
`anchor_state`, `added_places.weight`, `added_places.anchor_facets text[]`,
`added_places.facet`. Removed: the `deep` flag and the client toggle. New scripts: `dossier.ts` (batch), `brief.ts` (batch),
`eval.ts`. Changed modules: `plan.ts` (facet), `profile.ts` (durable card +
facet card), `recommend.ts` (pre-rank, shortlist, structured output,
caching), `places.ts` (field mask option, IDs-only search).

---

## 8. Roadmap

Ordered so each step is measurable before the next.

**Phase 1: measure and re-anchor (about two weeks of sessions)**
1. `eval.ts` with leave-one-out recall@6 on the current engine. Establish the
   baseline number before touching anything.
2. Facets on every saved place and every query; facet-conditional anchors
   with the explicit-then-inferred rule; the durable/facet card split; the
   after-save and at-search anchor prompts (§4.2.1), rate-limited.
3. Weight dial on saved cards, passive weights, card rebuilt on write.
4. Prompt caching with Sonnet 5 as the ranker; structured output; closed
   places filtered. Re-run the eval. Expect the biggest jump here.

**Phase 2: dossiers and briefs**
5. Atmosphere fields on the search request; `place_dossier` with Haiku
   extraction; nightly batch for saved places.
6. Town briefs with domain-restricted web search; IDs-only resolution;
   brief names joined to the candidate pool; pre-warm the top metros.
   Retire the deep toggle and the weak-round retry; first-search fill and
   background brief in their place (§5.8).
7. Deterministic pre-rank and the twelve-place shortlist. Re-run the eval.

**Phase 3: widen the pool, close the loop**
8. Foursquare OS Places or Overture as the second candidate source, with
   Google demoted to resolution and Atmosphere for the shortlist. Watch the
   Places bill fall.
9. Tripadvisor into dossiers for sights and lodging.
10. Pairwise judgements in the app; tune pre-rank weights on them.
11. Want-to-go pattern mining and adjacent-facet anchors from it.

Deferred, on purpose: embeddings, the taste graph, weather and destination
mode. They all get easier once dossiers and facets exist.

---

## 9. Decisions and open questions

Decided 2026-09-02:

- **More anchors per facet, never required.** Up to eight per facet, elicited
  with one-tap prompts at natural moments and falling back to inference
  whenever the user has said nothing (§4.2, §4.2.1).
- **Deep is the default, not a mode.** The toggle goes; every search is deep
  through dossiers and town briefs, with a bounded first-search fill in new
  towns (§5.8). Users will expect this, and it is the moat.

Still open:

1. **Facet list.** Does the table in §4.2 match how you think about your own
   list? Anything missing that you search for (bookstores and record shops
   are split out deliberately; boat and marina stops are not).
2. **Weight dial wording.** "How good is it, really?" 1–5, or a three-way
   (fine / love / all-time)? Three is faster on a phone.
3. **How much of Muninn should run on Google?** Two things push the same
   way. The bill: every candidate search bills at the Enterprise SKU. And
   Google's terms: they restrict showing Places data on a map that is not
   Google's and caching most place data beyond thirty days, while Muninn pins
   Google-derived places on a MapLibre chart and keeps them. Moving the
   candidate pass to an open source (Foursquare OS Places or Overture) and
   using Google only to resolve IDs and pull reviews for the shortlist cuts
   both the cost and the exposure. Is that the direction, or do we keep
   Google as the sole source and take the terms question up separately with
   a lawyer before launch (GROWTH.md, Part 2)?

---

## Appendix A: where things live today

| Concern | File |
|---|---|
| Enrichment | `scripts/enrich.ts`, `src/lib/places.ts` (field mask at line 11) |
| Profile statistics and card | `scripts/build-profile.ts`, `src/lib/profile.ts` (`buildProfile`, `renderProfileMarkdown`, `ANCHOR_SPECS`) |
| Non-owner card, dietary and preference blocks | `src/lib/usercard.ts` |
| Request planning | `src/lib/plan.ts` |
| Saved-place helpers, Bayesian rating, area matching | `src/lib/saved.ts` |
| Candidates, research, ranking, pairing | `src/lib/recommend.ts` |
| Context assembly, nearby lenses, taste keywords, reconciliation | `server/server.ts` (`loadContext`, `NEARBY_LENSES`, `tasteKeywords`, `attachPlaces`, `handleRecommend`, `handleNearbyRecommend`) |
| Research cache, dislikes, feedback | `src/lib/store.ts`, `db/schema.sql` (`place_research`, `disliked_places`) |
| Portrait | `server/server.ts` (`handlePortrait`, `PORTRAIT_INSTR`) |
| Client request composition, lenses, deep retry | `public/beta/beta.js` (`composeRequest`, `LENSES`, `fetchArea`) |

## Appendix B: prices used (verified 2026-09-02)

- Claude Haiku 4.5: $1 / $5 per MTok; cache read $0.10; batch $0.50 / $2.50.
  Minimum cacheable prefix 4,096 tokens.
- Claude Sonnet 5: $2 / $10 per MTok; cache read $0.20; 5-minute cache write
  $2.50, 1-hour $4; batch $1 / $5. Minimum cacheable prefix 1,024 tokens.
- Claude Opus 5: $5 / $25 per MTok; cache read $0.50.
- Web search: $10 per 1,000 searches plus tokens. Web fetch: tokens only.
- Google Places API (New), per 1,000 requests, past the free monthly
  allowance: Text Search Essentials (IDs only) free and unlimited; Text
  Search Pro $32 (5,000 free); Text Search Enterprise $35 (1,000 free);
  Text Search Enterprise + Atmosphere $40 (1,000 free); Place Details
  Enterprise $20, Enterprise + Atmosphere $25 (1,000 free each). Billing is
  at the highest SKU any requested field triggers.
- Tripadvisor Content API: 5,000 free calls a month, then tiered.
- Yelp Fusion: $7.99 / $9.99 / $14.99 per 1,000 calls (Starter / Plus /
  Enterprise), 30-day trial, reviews from Plus.
- Reddit Data API: free non-commercial at 100 queries a minute with
  approval; commercial $0.24 per 1,000 calls after a manual review.
- Foursquare OS Places and Overture Maps Places: free, permissive licences,
  commercial use allowed.
