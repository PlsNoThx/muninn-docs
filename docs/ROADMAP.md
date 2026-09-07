# Muninn Beta — Execution Roadmap

Living tracker. Batches 1–5 and 8 are shipped, plus the September "beautify"
round (sight, cartouche, graticule, context box, Saved ledger, personal taste
page, night side, tile speed); what remains is listed under **Open work**. Written for whoever picks this up next — the **Hard-won facts**
section at the bottom is the part worth reading before touching the map.

Companion docs: [`SHADERS.md`](SHADERS.md) (the next artistic push),
[`GROWTH.md`](GROWTH.md) (monetisation + legal), [`VISION.md`](VISION.md)
(long horizon). `GLOBE.md` is historical — it records the globe.gl and
two-engine eras, superseded by the single MapLibre engine.

## How to work (usage discipline — this matters to Ian)

- All beta code is in `public/beta/` (`beta.js` ~2k lines, `taste.js`,
  `saved.js`, `beta.css`, `index.html`, `cities.json`, `world.geo.json`).
  `public/sw.js` is shared with the classic app (it now caches tiles). The classic app
  (`public/app.js`, `public/index.html`) is a separate, working product —
  don't touch it except where a change is explicitly shared (`auth.js`).
- Per batch: make all edits → `node --check` on a copy of beta.js (ES module,
  so copy to `.mjs` first) → one commit → push to the feature branch AND to
  `main` (Render deploys `main`, ~2 min).
- **Default to pushing without screenshots.** Ian tests on his phone. Spin up
  the headless rig only for high-visual-risk or gesture work.
- The rig lives in the session scratchpad: playwright + chromium at
  `/opt/pw-browsers/chromium` with `--enable-unsafe-swiftshader`, local copies
  of `maplibre5.js/css`, and a curl-backed tile cache (`tilecache/`) because
  CDN fetches fail through the sandbox proxy. Pre-warm tiles in parallel
  before any street-zoom test — the synchronous fetcher cannot keep up.
  Local server: `PORT=4600 npx tsx server/server.ts`.

## Shipped

**Batch 1 — ring + motion.** Ring bands rotate (outer clockwise, a second
sparse band counter-rotating), flick-to-spin continues the flick's direction
the long way round, flick gate tightened to z ≤ 2.8.

**Batch 2 — arcs, poles, relief.** Arcs are bold dashed lanes with a thin
offset twin; a comet runs out along one lane at a time, round-robin. Routes
group by REGION (cells within ~9° merge) and anchor on each region's densest
cell, so one lane per place you actually travel to and the origin lands on the
cluster coin. No lane crosses a pole — a path peaking past 68° is redrawn as a
longitude-following route with a gentle bow. Antarctica's polar-night band is
filled from the raw noise field, brightness-matched to the imagery it borders.
Hillshade exaggeration is zoom-interpolated.

**Batch 3 — dense pins.** The heatmap was built, and Ian rejected it as
splotchy. Superseded by **clustering**: engraved coins carrying the count,
coloured by their mix (slate = want-to-go, sienna = favourites, bronze
between) via `clusterProperties`. Tap a coin to open it up. Pins are tappable
with widened hit circles → a floating card popover.

**Batch 4 — cards.** Save / Pass / relist wired to `/api/save`,
`/api/dislike`, `/api/relist`, plus the classic app's swipe gesture ported
(right saves, left passes) and an "Open in Maps" link on every card and
popover. Weak rounds (fewer than 3 picks or best fit under 70) auto-retry once
with `deep:true`.

**Batch 5 — search UX.** The label shows "Town, Region". The raven flight was
built, then **removed at Ian's request** — the staggered pin drops and the
"picks ready" handle survive it. The lens selector went through four rounds
and landed as: a flat struck **doubloon** above the town label; tap it to
search everything here, hold it and it shrinks to a hub while four sibling
coins take station N/E/S/W (coffee, food, drinks, outdoors); drag to one and
release to fire; the chosen lens names itself in serif below the rose.

**Batch 8 — taste page.** Shipped on the new interface: an archetype earned
from the data ("The Exacting Harbor Forager"), engraved dials, and the
evidence underneath. Reads `/api/me` plus places already loaded, so it costs
no extra API calls.

**Not in the original plan, shipped anyway:**
- **Deploy visibility.** Cloudflare edge-caches `.js`/`.css` by extension and
  overrode our `no-cache` with a ~4h browser TTL, so pushes were invisible for
  hours. HTML passes through uncached, so asset URLs now carry a `__V__` token
  the server stamps per boot. This is why deploys show up now.
- **Near me.** Pulls the device position, marks it, flies there, names the
  label.
- **Place tokens.** Every place rides in a stretched name bubble
  (`icon-text-fit`) at street zoom, and the basemap's own generic POI labels
  are hidden, because our places were disappearing into the roads.
- **Touch isolation** on the coin (see Hard-won facts).

**The beautify round (Sep 2026, on Fable):**
- **The sight.** An engraved reticle at the true map centre — the point the
  area name is read from — fading in at region zoom, thinner at street zoom.
- **The cartouche.** The area name sits on a double-ruled title plate, legible
  over any street grid. Its grain follows the zoom: nearest known city at
  region view, county as you approach, town at street level
  (`nameTier`/`reverseTown`).
- **Graticule.** 30° rules at planet view, 10° as you approach, the equator,
  tropics and polar circles heavier and lettered along the line. Above the
  tints, under the coast ink, gone by street.
- **Cloud shadows** offset and darkened enough to read as height.
- **Top chrome paper fade** so tokens never sit on the clock. Classic tab gone.
- **Context box.** The bottom search opens into *where* + *after* with the
  deep-research toggle; the request is sticky (localStorage), shown under the
  cartouche with a clear ×, and composed with the lens on every search path
  (`composeRequest`). Note the hidden submit button — see Hard-won facts.
- **Saved ledger** (`saved.js`): favourites / want-to-go / passed, search,
  sort (recent, A–Z, nearest me), lens filters, add-by-name with a note,
  paste/CSV import with review-then-commit, swipe a want-to-go right after
  you've been. Every change redraws the globe through `setSaved`.
- **Taste page**: titled by first name, a three-sentence **portrait** from
  `/api/portrait` (Haiku, cached server-side by a hash of the evidence — one
  call per distinct profile), and the settings that steer every search (home
  base, dietary rules, likes, avoids, sign out). Nav is above the sheets now.
- **Night side.** The real sun from the clock (`subsolar`): ten twilight
  bands from the terminator to astronomical night as clipped latitude strips
  whose edges follow the terminator (`nightBand`), city lights from
  `cities.json` and your own saved places as lamps, opacity from a sun-altitude
  **expression** (`altExpr`) re-baked each minute. All map geometry: engine-
  locked, works in Lite. Fades out by z 7.5. Chrome stays cream.
- **Reach + near-me by the sight (Sep 2, morning).** Coin taps and lenses
  search around the sight's coordinates through `/api/nearby-recommend`
  with a hard radius (walk / bike / drive / far, `REACH`, persisted); a
  dashed ring on the chart shows the reach. The old path geocoded the
  county name to its centroid, hence a coffee shop 40 minutes away.
- **The lean.** The globe holds the Earth's tilt (`TILT`, negative leans the
  pole right) at planet zoom, easing to north-up by z6, applied on
  `zoomend` only. Set `TILT` to 0 to remove it.
- **Who's Muninn** on a tap of the wordmark; graticule drawn above relief so
  it shows over the sea; fourteen twilight bands.
- **Tile speed.** The service worker serves tiles, fonts and relief
  cache-first from a 900-entry store; `watchLink` measures the first tiles'
  resource timings and on a slow link drops relief, renders at 1x and skips
  the cloud snapshot; `prefetchHome` warms z5 around the densest saved
  clusters at idle.

**Taste engine v2 (Sep 3).** The recommender is rebuilt behind an engine
switch (`src/engine/index.ts`; `TASTE_ENGINE=v1` selects the old flow, which
also runs as the automatic fallback). Facets, facet-conditional anchors, place
dossiers from Google's Atmosphere fields, town briefs in the background, a
deterministic pre-rank to a twelve-place shortlist, Sonnet 5 ranking with a
cached prefix and structured output. Every search is deep; the deep chip is
gone. The results sheet says which of your places the ranking was measured
against. `scripts/eval.ts` measures leave-one-out recall. Thesis and detail in
[`TASTE_ENGINE.md`](TASTE_ENGINE.md). **Owner step:** run the v2 block at the
end of `db/schema.sql` in the Supabase SQL editor so dossiers, briefs and
anchor states persist. (Done 9/3; the closed-places backfill ran by id and
then by name, 90 of 90.)

**Search latency, first pass (Sep 4).** The v2 orchestrator no longer runs
its stages in a line. The three Places searches were already concurrent;
now the town brief (read, or started in the background) and the cached
dossiers for the pool and the anchors load at once, the brief's up-to-six
name resolves fire in parallel instead of one after another, and Haiku
describes only the pre-ranked top sixteen that still lack a dossier
(`EXTRACT_CAP` in `src/engine/v2/recommend.ts`) instead of up to
twenty-four. Pre-rank runs twice: once on cached dossiers to choose what is
worth describing, once with the new ones. `?debug=1` shows `ms` per stage
and `dossiers {cached, extracted, full, thin}`.

**Town briefs read Reddit through a search API (Sep 4, Brave from Sep 6).** Anthropic's search
index excludes every site that blocks its crawler, and those are the sites a
brief most wants (Reddit, Eater, the NYT). The brief writer now runs three
Google Custom Search queries per town first (`src/lib/websearch.ts`), hands
the titles and snippets to Sonnet as evidence, then searches the reachable
sources with Anthropic's tool. Off unless `GOOGLE_SEARCH_CX` is set; the
brief row records how many snippets it read and `?debug=1` shows "N places,
M snippets". Google's Custom Search JSON API was the first provider and
turned out to be closed to new customers ("This project does not have the
access to Custom Search JSON API", 9/6; it retires 2027-01-01), so the
provider is the Brave Search API, the index the Claude chat app itself
searches. Owner step: a Brave API key as `BRAVE_SEARCH_API_KEY` on Render.
Verified 9/6: Apopka's brief read 30 Brave snippets and its summary says
plainly that the town has no critic footprint. Open question: Reddit's
robots.txt has allowed only Google since 2024, so how fresh Brave's Reddit
coverage is will show in a few briefs' sources.

**Every place scored; the score decides (Sep 4, evening).** The ranker no
longer returns six: it scores every shortlisted row (sixteen now) with a
fit and a reason, and the sheet shows 70 and above as picks, however many,
50–69 collapsed under "worth a look", and a thin town's closest three with
honest scores instead of padding. Same Sonnet call, so no new cost. Also in
this batch: a general search's anchors now cover every kind on the
shortlist (a coffee shop was being judged against restaurants and bars);
the town brief streams with a twenty-minute limit at medium effort and
starts only after the search's own model calls; dossier batches run three
at a time; the durable card rounds its numbers so a save no longer throws
the cached prefix away. Thresholds are `PICK_MIN` / `MAYBE_MIN` in
`public/beta/beta.js`; the fit scale is the model's own, so read a few
towns before moving them.

**Ian's two bugs (Sep 6, evening).** A town search now leans toward home:
`homeLean` in `src/lib/places.ts` builds a rectangle about 900 km on a side
around the profile's hometown and the area path's three searches (and the
brief's name resolves) carry it as a `locationBias`. A rectangle, not a
circle: Google caps a circle at 50 km, which would drag every town search
home, while a rectangle has no cap and only breaks ties, so "Jupiter
Beach" from Naples finds the one up the coast and "Paris" still finds
Paris. `?debug=1` shows it as `lean`. On the client the fly-to geocoder
asks Nominatim for five hits and, among those of comparable importance,
takes the nearest to home (the hometown from `/api/me`, else the densest
saved cell). And a saved or passed pick now leaves the picks layer at once
(`dropPin`), so a save turns the pick pin into a want-to-go pin instead of
adding a second one beside it; the landing choreography reads the live
list so a fast save cannot bring the pin back. No hometown, no lean: the
single-user and fresh-account paths are unchanged. v1's area path (the
fallback engine) was left without the lean.

**The ring is the search (Sep 6, night).** Ian's three asks after testing:
the "after" box is always open (no quill, no standing-request chip); the
lens coin and the town cartouche are resting (the markup is gone from
`index.html`, the rose and coin code from `beta.js`; their CSS stays and
git has the rest, commit 56470f4 is the last with them); and the reach
ring around the sight is the search instrument. It is drawn bolder now (a
wash, a halo, a 2px rule, the distance written on its northern edge) and
it locks: a search with an empty box runs around the sight within the
ring, and the ring holds that point, solid, labelled "looking…", until the
search is done. A named town geocodes first (nearest home among
comparable hits), the ring moves there and locks, the planet flies out,
and the search runs within the reach around that point through the
near-me path, the town's name riding along for the brief. "coffee in
naples fl" splits on the client the way the server splits it. An
unplaceable name falls back to the plain town search. `lockReach` and
`reachRing` in `public/beta/beta.js`. Verified headless (Chromium through
a curl pass-through for tiles, since the sandbox's browser has no proxy
CA): the layers build, the ring sits at the sight, locks at Jupiter on a
town search, holds while the map moves, and releases dashed when the
search resolves. Three pre-existing MapLibre "expected number, found
null" warnings come from an older label expression, not the ring.

**Place cards (Sep 6, night).** Ian's brief: a tapped saved place pops its
card, the card opens a full page for the place, and the page carries every
edit and note so a spot can be dialled in; saved places never blank; the
rating slider needs a commit. Shipped as one renderer (`public/beta/cards.js`,
`placeCard`) for the pick, saved, passed and popover cards, replacing the
four copies. The pick card now shows the engine's caution, the dossier in one
line ("what it's like"), what the town brief said ("locals say") and the
distance from the sight; a saved card shows its dossier line (or the person's
own correction, marked "in your words") read from `GET /api/place?id=`, which
is cached per session and starts a dossier build for a place without one. The
weight dial is the **vial**: a glass filled green to the level with the word
beside it (fine / good / really good / great / all-time), inert to a drag; a
tap arms the ruled scale with a Set, and Set posts. On a want-to-go the glass
is empty and reads "been? rate it": Set makes it a favourite and rates it in
one motion (relist, then weight). Tapping a pin opens the real card in the
popover, with its actions. **The place page** (`public/beta/place.js`,
`#place/<id>`, back button and reload safe) has the list and the moves
between lists, the big vial, a dated journal of notes (`place_notes`; the
first entry also fills the row's note the engine reads, and every entry runs
the on-save corroboration), the dossier in full with "Not quite right?" (a
personal correction, `place_corrections`, shown on the card and handed to
the person's own searches in the facet card as "in their own words", never
written into the shared dossier), the anchor question posting to
`/api/anchor`, where it is and how far from home, and Maps. A passed place
gets the same page with its reason editable (`/api/pass-reason`). Backfill:
`scripts/backfill-dossiers.ts`. Owner steps: run the "Place cards" block of
`db/schema.sql` in Supabase; run the backfill once. Verified in the headless
rig against a fixture (popover, vial arm/drag/set, page sections, note,
correction, anchor, close, want-to-go promotion); the phone check is next.
Not done from the brief: a favourite-with-note path from a pick, the pass
reason box on the pick, the after-save anchor chip, receipt undo, a page for
an unsaved pick (deliberately not yet).

**The pick's teaching actions (Sep 6, late).** Save on a pick opens a
chooser (★ Been there / ◇ Want to go) with an optional note; Pass opens a
reason box; a swipe takes the quick path as before and the receipt then
offers the note (`/api/note`) or the reason (`/api/pass-reason`). After a
favourite, the anchor chip ("One of your coffee anchors? yes / not quite",
`/api/anchor`) when the search's facet is specific and has fewer than five
anchors, once a session. The word is "anchor" everywhere; "yardstick" is
retired. Fields inside a card no longer start a swipe. Briefs now record
`snippet_hosts`. Verified in the rig against a fixture search.

**Miles or kilometres (Sep 7).** One preference, kept on the device like
the theme and the reach (`mn-units` in localStorage; the default reads
`navigator.language`, en-US and en-GB to miles, everything else km). Set on
the taste page's settings form as a miles / kilometres segment, applied on
tap, no Save needed; in single-user mode the same segment stands alone.
Every distance label goes through `distLabel(km)` in `cards.js`: the reach
coin and the ring's northern edge, "within …" on the sheet subtitle, a pick's
"… from the sight", the ledger's "… away" under the nearest-me sort (was
miles by its own haversine; now `kmBetween` and the shared label), and the
place page's "… from home". The reach scale re-rules its ticks in the unit
(0.5 to 25 mi, or 0.5 to 60 km) and the reach snaps to round numbers in it
(a tenth of a mile to 2, halves to 10, whole miles after), so the coin reads
"5 mi", never "4.97 mi"; a first boot in miles lands on 15 mi. Verified in
the rig: en-US boots in miles, the toggle relabels the ring live and
survives a reload, de-DE boots at 25 km.

**One town, one brief (Sep 7).** Ian typed "st pete"; the search landed on
St. Pete Beach and the brief table held the town three ways ("St pete FL",
"Saint Petersburg, Florida", "St Pete") beside junk briefs for "Lunch" and
"Coffee" from before the request split. Two fixes. On the client,
`geocodeArea` in `beta.js` asks Nominatim for settlements first
(`featureType=settlement`, never the park or street that carries a name:
"st pete fl" was a county park in St. Pete Beach), expands the few
nicknames the geocoder does not know (`TOWN_ALIASES`: st pete, nola, vegas,
nyc, sf, dc, jax, atl; a typed region after a nickname is implied by it),
keeps the nearest-home rule for a name shared across regions ("Naples",
"Jupiter") but lets the bigger place win among neighbours in one metro (the
city over its beach), and hands the search the town's canonical name from
the geocoder's address, "City, Region", the same shape `reverseTown` gives
the cartouche. On the server, `briefKey` runs the metro through
`canonicalTown` (`src/lib/geo.ts`: lowercase, no punctuation or accents,
saint/fort/mount abbreviated, the country dropped, a trailing state code
written out), so "St pete FL", "Saint Petersburg, Florida" and "St.
Petersburg, FL, USA" share one row; `addressInArea` matches a region by code
or name so the canonical label still finds the person's own places.
`?debug=1` reports `briefKey`. The table was migrated by hand the same
night: every key rewritten to the canonical form (so no town re-briefs), the
St. Pete triplicate collapsed to the richest brief (25 places, 30 snippets),
and the three junk rows deleted. Nominatim facts worth keeping: it does not
know "st pete" at all (an Indonesian village), it does know "philly" and "ft
lauderdale", and without the settlement filter "jupiter beach" is a beach in
Croatia.

**The heartbeat (Sep 7, night).** Tampa, cold, at a slow hour, came back
"Load failed" on the phone while the server's log shows the rank finishing.
The answer was lost in transit: Cloudflare gives the origin 100 s to start
responding and then drops the request with a 524, and a search's wall
clock brushes that (St. Pete the same night: 70 s of engine time). Now
`/api/nearby-recommend` sends its 200 headers the moment the payload is
valid and writes a space every fifteen seconds until the JSON follows
(`heartbeat` in `server/server.ts`); leading whitespace is valid JSON, so
`res.json()` on the client is unchanged, and every timer on the path (edge,
Render, the phone's radio) is reset by each byte. The status is already
sent by the time the engine can fail, so an error travels in the body and
`fetchNear` in `beta.js` reads `data.error`. The classic app's town route
(`/api/recommend`) is untouched, so `public/app.js` is untouched; the
beta's plain-name fallback still goes through it and keeps the old limit.
Proved in the sandbox with a throwaway server: headers in a millisecond,
chunks before the body, the body parsed with its leading spaces, an error
body read as one. What it does not cover: iOS suspending Safari when the
screen locks mid-search cancels the fetch client-side, heartbeat or not; a
native shell with a background session would, and a job-and-poll search
would survive anything.

## Open work

Roughly in the order that serves the product. The current page (what is next,
what changed last session) is `PROGRESS.md` at the repo root.

0. ~~Place cards~~ — shipped 9/6 (above), teaching actions included. Left:
   receipt undo. Brief: [`PLACE_CARDS.md`](PLACE_CARDS.md).
0a. **Cutover** — retiring the classic app and serving the beta at `/`.
   The audited task list (owner tools, feedback, data-safety fixes, the
   root-path plumbing, parity, the legal checklist as tasks, cleanup) is
   [`CUTOVER.md`](CUTOVER.md). A1, the owner sheet (`owner.js`, the
   fourth nav tab for the owner account), shipped 9/7. A3 and the display
   half of C18 shipped 9/7 too: every pin on the chart is now an
   OpenStreetMap point from `place_display` (`src/lib/display.ts`,
   `npm run backfill-display`), the hometown included; picks are the open
   half, behind `PIN_PICKS` (brief: `dev/briefs/places-on-non-google-map.md`).
1. **Shaders / atmosphere** — paper first, then limb. See
   [`SHADERS.md`](SHADERS.md). The terminator piece there is now done as
   geometry (above); a shader would only smooth it further.
2. **Coin-colour legend.** The slate/sienna ramp now carries meaning nothing
   on screen explains. One quiet line near the "places remembered" count.
3. **Share-as-image** on the taste card (GROWTH.md calls this the viral loop):
   canvas render of the cartouche + a share button.
4. **Logo** (roadmap 19). Direction chosen: the search coin's face with the
   original m-shaped birds and crescent over a graticule sliver (artifact
   "Muninn Marks"). Wire it as brand mark, app icon on warm black, favicon
   once Ian confirms. `public/icon.svg` stays for the classic app.
6. **Onboarding synopsis** (21) — why Muninn vs Google, in `renderOnboarding`
   (shared with the classic app, keep it compatible).
7. **Too-broad favourites** (22) — server-side, reject locality/political
   types as favourites and flag them back to onboarding.
8. ~~Search radius~~ — done as the reach.
9. **Lens settings** — the lenses and their coin are resting (9/6, "The
   ring is the search"); if they return, Ian wanted them configurable
   (pick ≤ 4, persisted in localStorage).
10. **Taste-engine UI** — the anchor prompts (after a save, under results,
    on the taste page), per `TASTE_ENGINE.md` §4.2.1. The weight dial shipped
    (ruled slider on saved cards); the server side (`/api/anchor`) is in place.
11. ~~Search latency~~ — two passes shipped 9/4–9/6 and the rest is the
    API's own speed (see the Claude API facts below). Cold town 15–25 s,
    warm 8–12 s. Do not instrument further; the next gains are structural:
    pre-brief and pre-dossier likely towns, or a faster rank model.
12. **Eval baseline** — `npx tsx scripts/eval.ts --engine v2 --limit 30` now
    that a dozen towns have briefs and dossiers; record recall@6 in
    TASTE_ENGINE.md.
13. ~~Home-country lean in search~~ — shipped 9/6 (below, "Ian's two bugs").
14. ~~Saving a pick makes two pins~~ — shipped 9/6 (same entry).
15. ~~One brief per town~~ — shipped 9/7 ("One town, one brief", above).

## Hard-won facts (measured, not assumed)

- **The map draws display coordinates only.** A saved row carries two
  points: `lat/lng` (Google's, for the engine, the distance sorts and the
  "saved here" groups) and `dlat/dlng` (an OpenStreetMap point from
  `place_display`, the only one a MapLibre layer may read). A row with the
  first and not the second has no pin *by design*; it is not a bug to fix
  with a fallback. Maps Platform ToS §3.2.3. See `mapPt`/`anyPt` in
  `beta.js` and `dev/briefs/places-on-non-google-map.md`.

Every one of these cost a debugging round. They are not obvious from the docs.

**MapLibre**
- `line-gradient` and `line-dasharray` **cannot coexist**. A gradient set on a
  dashed line is a silent no-op — this is why the first animated arcs looked
  static. Dashes live on the twin layer; the gradient lane carries no dashes.
- `line-gradient` can only read `line-progress` — never a feature property or
  feature-state. Per-feature animation therefore needs **one layer per
  feature** (hence `mn-route-0..7`).
- `flyTo` normalises longitude to the shortest path. An unwrapped target
  (e.g. 730°) is folded back, so a deliberate long-way-round spin must be
  driven by hand and handed to `flyTo` near the end.
- `icon-allow-overlap` / `text-allow-overlap` opt a layer **out of collision
  detection entirely** — that is what stacked two place labels on top of each
  other. Use `symbol-sort-key` and layer order for priority instead.
- Symbol **placement priority goes to the layer listed LATER**, not earlier.
  Verified on a deliberately crowded fixture.
- `text-variable-anchor` lets a crowded token flip to another side of its dot
  before giving up its place — much better than losing labels outright.
- Heatmap, clustering (`clusterProperties`), 9-slice stretchable images and
  `line-gradient` all work fine under globe projection. So do large fill
  polygons, as long as nothing contains a pole: the night side is built as
  latitude strips clipped at ±180 for exactly that reason.
- Expressions have real trig (`sin`, `cos`, `asin`…), so a per-feature sun
  altitude can be computed in paint, with the sun baked in as constants and
  re-set each minute — no `setData` over 2,000 points.
- A zoom `interpolate` must be the outermost expression; data-driven pieces
  go in its output slots (`lightFade(k)`).
- **Never update a source per frame.** The reach ring redrew on `move`, and
  at planet zoom the ambient drift moves the map every frame: an unbounded
  worker queue. Sources update on `moveend`, on their own change, or on a
  timer — never on `move`.
- **Never move the camera from inside a per-tick event.** A `setBearing`
  (a `jumpTo`) on every `zoom` tick fired a full move cycle each frame
  (clouds resample, geocoder debounces, ring redraws) and made pinching
  choppy on the phone. The lean is applied once, on `zoomend`, as an
  `easeTo`; every `flyTo`/`easeTo` carries its own bearing target.
- The night side keeps the sun factor in the light features' data and never
  touches paint after creation; the three sources update on separate frames.
  That was found by bisecting a 13 GB renderer OOM in the rig, but note the
  next fact before spending more time on it.
- `sourcedataloading` carries no tile, and vector tiles are fetched inside
  the workers, so neither MapLibre events nor the page's resource timings
  can time a tile. `areTilesLoaded()` and the time from map creation to the
  `load` event are the only honest link measurements.
- The service worker's cache-first tiles were verified in the rig: 41 tiles
  cached on the first load, one network fetch on the second.
- OpenFreeMap's glyph stacks: `Noto Sans Regular/Italic/Bold` only.
- Nominatim at city granularity (`zoom=10`) returns only the county for a
  beach town like Cape Canaveral; ask at `zoom=14` and read the hierarchy
  (city, town, village, municipality, suburb, neighbourhood) from the
  finer feature's address instead.

**The engine-locked doctrine**
- Never model the camera — **measure it**. The globe's silhouette is only
  screen-centred at the equator; `fitSphereCircle()` probes the map's own
  `project`/`unproject` to find it. Clouds sample `unproject` per cell. The
  ring, limb and (formerly) the raven all derive from the measured circle.
  Anything that assumes a formula drifts off the sphere.

**Browser / platform**
- `overflow: hidden` on a flex item makes `min-height: auto` resolve to **0**,
  so it will shrink to nothing. This collapsed every result card.
- `pointercancel` is the system taking the gesture away — treating it like
  `pointerup` fires actions nobody asked for. On touch, capture the pointer on
  `pointerdown` (not later), and set `touch-action: none`,
  `user-select: none`, `-webkit-touch-callout: none` or the browser will bid
  for the gesture and iOS will show its long-press callout.
- Cloudflare drops a request the origin has not started answering within
  100 s (a 524; Safari on the phone shows "Load failed"). A search is 60–100 s
  of model time at a slow hour, so the near-me route heartbeats ("The
  heartbeat", above). Any new long route needs the same.
- Cloudflare caches by file extension regardless of origin headers — hence
  `__V__` stamping.
- A form with **two text inputs and no submit button gets no implicit
  submission**: Enter (and the phone keyboard's Go) does nothing. The search
  form carries a visually-hidden submit button for this.
- A CSS `display:` rule on an id beats the UA `[hidden]` rule. Write
  `#thing:not([hidden]){display:flex}`.
- Sheets are `inset:0`; the nav has to out-z-index them and the sheets pad
  their bottom, or the tabs vanish whenever a sheet is open.

**The test rig**
- The app's map canvas has no `preserveDrawingBuffer`, so reading pixels back
  from it returns an empty buffer. **Pixel-diffing the live map is invalid** —
  use screenshots (which go through the compositor) or a purpose-built map
  with the flag set.
- `pg.evaluate(() => someAsyncFn())` **awaits the returned promise**. Sampling
  a running animation this way silently measures the state after it finished.
- Software GL (swiftshader) renders correctly but says nothing about
  performance — judge that on a real device.
- Never `pkill -f` / `pgrep -f <pattern>` where the pattern appears in the
  shell's own command line — even the `[t]sx` bracket trick killed the shell
  here (exit 144). Kill by exact process name instead: `pgrep -x chrome`,
  and start the server with `nohup … & echo $! > srv.pid`.
- Timed-out rigs leave **orphaned Chromium renderers** behind; a few of them
  push the load average past 20 and everything, including a two-line Python
  edit, starts timing out. Kill `chrome` by name before blaming the code.
- `waitForFunction("window.__muninn.map")` resolves before the layers exist;
  wait on a late layer (`getLayer('mn-grid-major')`) instead.
- A registered service worker takes requests away from `page.route`. Block
  it (`serviceWorkers: "block"`) in ordinary rigs; to test the worker, route
  at the context level with `PW_EXPERIMENTAL_SERVICE_WORKER_NETWORK_EVENTS=1`.
  Playwright's `proxy` option sends localhost through the proxy too (405).
- "Target page, context or browser has been closed" a couple of seconds
  after a call is the OOM killer, not a JS error: `dmesg | grep Killed`.
- **The rig cannot re-bake the night side.** Under swiftshader every
  world-spanning polygon becomes CPU-side globe geometry; a second
  `paintNight` at a new sun reproducibly reaches 13 GB, while the phone
  re-bakes it every minute all day. Verify night-side changes on the phone;
  the rig is for everything else.
- `tsx`'s esbuild service was found spinning at 87% CPU for over an hour
  with nothing to do, holding the load average above 20 with the orphaned
  renderers. If a two-line edit times out, `ps --sort=-%cpu` first.
- The sandbox proxy does not reach nominatim; geocoder answers can only be
  checked on the phone.

**Claude API**
- `place_dossier` needs a `snapshot jsonb` column and the 9/3 schema block
  did not create it. The store selects and upserts that column, so with it
  missing every dossier read and write returned 400, `warnOnce` logged it
  once, and the cache stayed empty for a day while the app looked fine.
  Added by migration 9/4 and to both schema files. Check the counts, not
  the logs: `select count(*) from place_dossier` after a search.
- With structured outputs, a `max_tokens` cut does **not** show up as broken
  JSON. The constrained decoder closes the document validly, so the rank
  came back as one pick with an empty `why` and `fit_score` 0 and the app
  rendered it as a real result (the Marco Island search, 9/4 11:54). The
  rank's cap was 200 + 140 per pick; adaptive thinking shares that budget.
  It is 4000 now. Any structured call needs a cap far above the answer,
  and the diag's `rank.stop` shows `max_tokens` when it happens.
- Measured on the first Marco Island search (cumulative `ms`): search 0.9 s,
  brief join 0.2 s, dossier extraction 19.4 s, rank 10.5 s. Both model
  stages are output-bound: Haiku wrote sixteen dossiers in one call, Sonnet
  wrote six 40-word reasons. Fix shipped 9/4: small dossier batches in
  parallel, tighter tag counts, `why` capped at 22 words. Second search
  (12 of 17 dossiers cached): dossiers 7.1 s, rank 8.6 s, 17 s total. Then:
  picks name a shortlist row number instead of echoing the place id,
  thinking off on the rank (all the evidence is in the context; the answer
  is what costs time), batches of three. Judge the picks' quality with
  thinking off before trusting the number; turn it back on in `rank.ts` if
  the reasons get vague.
- **`web_search` rejects an `allowed_domains` list that names any site
  blocking Anthropic's crawler** (400: "domains are not accessible to our
  user agent"). Eater, Reddit, nytimes.com, Bon Appétit, Condé Nast
  Traveler, Thrillist and Southern Living all block it, so every town brief
  failed from 9/3 to 9/4 and the failure lived only in the Render log. The
  brief now sends a short `blocked_domains` list (only one of the two lists
  may be sent) and names the wanted sources in the instruction. A failed
  brief now writes `{ error, attempted_at }` into its own `town_research`
  row, readable with one query.
- **Sonnet with thinking off answered a cold town with one blank pick**
  (`{"n":1,"why":"","fit_score":0}`, 36 output tokens, `end_turn`) twice
  out of three cold searches; with adaptive thinking on it never did.
  Thinking stays on at low effort; the principles forbid a blank pick; a
  blank answer is retried once at medium effort, then the pre-rank speaks
  from the dossiers. `diag.rank.retried` and `usable` show it.
- **Haiku's throughput swings threefold by the hour**, not by load on our
  side: Clearwater's first-text times were 1.6 s for every batch, then one
  batch generated 400 tokens in 10 s. The search waits at most
  `DOSSIER_DEADLINE_MS` for dossiers; batches still running finish in the
  background and are cached as they land, and the rows they were for rank
  on thin dossiers for that one search (`diag.dossiers.late` counts them).
  Nine seconds was too tight: St Pete 9/5 left nine of sixteen thin to save
  three seconds. It is fifteen now, an outlier guard, and every batch runs
  at once again (the "three at a time" was a guess at rate-limit retries the
  API counter has since ruled out: five searches, every response 200).
- **Haiku is the slow one.** Across 9/5–9/6 Haiku wrote a three-place
  dossier batch in 11–13 s (~35 tokens a second, first text at 1.7 s)
  while Sonnet wrote a 935-token rank at ~190 a second in the same minute.
  `DOSSIER_MODEL` on the host overrides the dossier model; `diag.dossiers.model`
  shows which ran. Try `claude-sonnet-5` and compare `batchMs`.
- **The rank's clock, split (St Pete 9/5, 16 rows):** 1.4 s to the first
  byte, 4.3 s of thinking at low effort, 9.2 s writing 1,266 tokens. The
  writing is the cost. Rows below 50 now return an empty why (the client
  never shows them), which is a few hundred tokens fewer per search.
- **The ranker will call another candidate "your" place.** West Palm
  Beach: "matches your Northwood/Chik•Monk" named two shortlist rows as if
  they were the person's anchors, and "echoes The Lab-style vibe" compared a
  row to row nine. Cause: coffee had five rows and the anchors carried one
  café, so the model reached for the coffee names in front of it. Fixes:
  the prompt says the rows are places the person has never been and forbids
  naming one row in another's why; the orchestrator checks every why
  against the other candidates' names (full name and first distinctive
  word) and replaces a hit with the dossier's own words, counted as
  `diag.rank.crossRefs` and logged with the offending line; general-search
  anchors now give two to any kind with three or more shortlist rows.
- **A town brief is minutes of server-side searching** and the SDK's default
  request limit is ten minutes: North Naples's brief died with "Request
  timed out". The call streams now with a twenty-minute limit (`BRIEF_TIMEOUT_MS`).
  Its search loop is also token-heavy on Sonnet, and while it ran beside a
  search the rank took 19 s instead of 4: the job now starts after the
  rank returns. Rate-limit contention is the working theory for both the
  slow rank and the two-of-five slow Haiku batches; check the org's tier
  in the console if it persists.
- **The durable card broke its own cache.** It printed exact favourite
  counts and a two-decimal price mean, so every save changed the cached
  prefix (`cache_read 0` on a search twenty minutes after the last).
  Numbers are bucketed now; a cache miss on a search with no deploy in
  between means something else is varying.
- A **new structured-output schema costs a one-time compilation** on its
  first request, then is cached for 24 hours (Anthropic docs). Every 9/4
  deploy changed the rank or dossier schema, so every first search after a
  deploy measured the compile, not the engine: the rank read 10.5 s, 8.6 s,
  then 18.5 s for 392 output tokens. Measure latency on the second search
  after a deploy, never the first, and do not churn schemas casually.
- Structured outputs (`output_config.format` with a `json_schema`) reject
  `minimum` / `maximum` / `multipleOf` and `minLength` / `maxLength` with a
  400. Put the range in `description` and clamp in code. The rank schema
  carried `fit_score: 0..100` and the dossier schema `confidence: 0..1`, so
  from 9/3 to 9/4 every v2 search threw at the rank and the engine switch
  fell back to v1 (`diag.v2_error`), while the dossier call failed silently
  into thin dossiers. Neither model call may swallow its error again: all
  three now `console.warn`.

**Search box**
- The one-line search box sends whatever was typed as the *area*, and the
  engine keyed town briefs on it: "coffee in naples fl" and a bare "coffee"
  each became a town and each burned a brief (eight web searches and a
  Sonnet call) that named no places. `splitPlaceQuery` in `src/lib/plan.ts`
  now splits "X in|near|around|at Y" on the last joiner into request and
  area on the server, and the sheet title uses the server's area. A bare
  non-place ("coffee") still becomes an area; only the geocoder could tell,
  and it can't be trusted to.

**Taste data**
- **Favourite rows from the original import carry the legacy Places
  types** ("establishment; food; point_of_interest", or a bare "bar"), while
  the same place's later want-to-go twin has the real ones ("brewery"). So a
  beer search found no beer favourites and borrowed cocktail bars as
  "adjacent" anchors (Clearwater 9/5, with a dozen breweries saved).
  Two fixes: `facetOfPlace` reads strong words in the place's own name when
  its types are generic (brewing, taproom, roasters, pizzeria, sushi...),
  and `upgradeTypes` copies specific types across rows that share a
  place_id before anchors and the temperament are computed. A proper
  re-resolve of the import's types is still worth doing one day.

**Supabase / data**
- The SQL editor runs exactly what is pasted; a terminal command pasted there
  fails with `syntax error at or near "npx"`. Error `LINE n` counts from the
  start of the pasted text, which tells you which file was actually run.
- `added_places` rows imported from the Takeout CSV do not all carry the
  CSV's `place_id` (the import resolved some by feature id or name), so a
  backfill keyed on `place_id` matched 86 of 90 closed places; `db/
  backfill_status_by_name.sql` catches the rest by name and city.
- A REST insert with a column the deployment lacks fails 400; the store
  strips the v2 columns and retries once, so saves survive an unrun schema.

**Data**
- NASA's daily true-colour snapshot is pure black below about -66° in
  September — polar night, not a no-data ring. The procedural cloud mask also
  damps polar clouds to zero, so filling one from the other copies nothing.
- The pole is a singularity in an equirectangular mask: every longitude
  converges, so any texture stretches into a pinwheel there. Blend toward the
  row mean near the cap.
