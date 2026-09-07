# Place Cards — session brief

> **Status (2026-09-06, night).** Built, on the branch, in the headless rig;
> phone check next. One renderer (`public/beta/cards.js`), the vial in place
> of the dial, the popover carrying the real card, and the place page
> (`public/beta/place.js`, `#place/<id>`) with the journal, the full dossier
> and a personal correction, the anchor chip, and the list moves. Server:
> `GET /api/place`, `POST /api/note`, `/api/correction`, `/api/pass-reason`;
> tables `place_notes`, `place_corrections`; `scripts/backfill-dossiers.ts`.
> ROADMAP "Place cards" has the account. The teaching actions (Save asks
> the list and offers a note, Pass asks a reason, the after-save anchor
> chip) shipped later the same night. Still open: receipt undo. A page for
> an unsaved pick was deliberately left out.

Prepared 2026-09-04 for the "Refining Place Cards UI" session. Written from the
code on `main`, not from memory; every gap below was checked in the source.

Why cards, why now. Ian's 9/1 notes ask, in three places, for the card UI to
be finished: "finish building the card ui on our beta model", "pins need to be
clickable, bringing up their card, or an expanded card", and "after loading,
you can drag up for the card ui". Cards are where every search ends and where
every taste signal is collected (save, pass, relist, weight, note, anchor). The
v2 engine now produces richer output per pick (a caution, a dossier, a town-
brief mention, anchors by name) and the card has nowhere to put most of it.

## 1. Where a card appears today

All beta card code is in `public/beta/beta.js`, `public/beta/saved.js` and
`public/beta/beta.css`. Cards are string HTML built with `esc()`, actions are a
delegated click handler on `#cards` keyed by `data-act`, and `saved.js` gets
its helpers by injection: `mountSaved({ esc, metaLine, mapsAnchor,
makeSwipeable, collapse, toast, getSaved, setSaved })`.

| Surface | Code | Shows | Actions | Gesture |
|---|---|---|---|---|
| **Pick** (results sheet) | `beta.js` `pickCardHtml` | rank number, name, fit pill, `why`, meta (type · $ · ★), Open in Maps | Save (always to want-to-go, no note), Pass (reason sent as `""`) | swipe right saves, left passes |
| **Saved here** (results sheet) | `beta.js` `savedCard` | name, note, meta, Maps | relist (favorite ↔ want-to-go) | swipe either way relists |
| **Ledger** (Saved sheet) | `saved.js` `card` | name, remembered / closed-for-now tag, origin tag (anchor / imported / month), note, meta · locality · miles, Maps, weight dial | relist, Remove | want-to-go only: right = favorite, left = not for me |
| **Found** (add by name) | `saved.js` `foundCard` | name, "already saved", meta · locality, Maps | Favorite / Want to go, then a note input | right = favorite, left = want-to-go |
| **Passed** (ledger, third tab) | `saved.js` `card` with `seg === "passed"` | name, "not for you", the reason | Undo | none |
| **Pin popover** (map) | `beta.js` `showPop`, `.card.pop` | name, fit, `why`, ★ / ◇, meta, Maps | none | none |
| **Receipt** | `collapse()` | one italic line ("◇ Added X to your want-to-go list.") | none | none |
| **Story** (Who's Muninn) | `index.html` `.card.story` | prose | close | leave it |

### What is wrong with each

- **Pick.** `caution` comes back from the engine (`Pick.caution`, `src/engine/v2/types.ts`) and is never rendered anywhere in the beta. There is no distance from the sight on a near-me search, though the client has the sight coordinates and the reach. The rank number and the title row are inline styles. Save has no favorite path and no note, so "I've been there" from a pick is two steps in two sheets. Pass sends an empty reason: the classic app's reason box (`public/app.js`, `.dislike-reason`) was never ported, so the dislikes-that-learn signal is dead in the beta.
- **Saved here vs. ledger.** The same saved place renders two different cards: the results-sheet one has no remembered tag, no origin, no locality or miles, and no weight dial. One renderer should serve both.
- **Ledger.** A note can only be written at add time. Notes are worth +1 weight and corroborate the dossier on the web (`src/engine/v2/onsave.ts`), so they should be editable on the card. The weight dial sets `touch-action:none` on the whole row, so a vertical scroll that starts on it is eaten. No anchor chip, though `/api/anchor` and the `anchor_state` table exist (TASTE_ENGINE §4.2.1).
- **Popover.** Different markup from the sheet card, no actions (a pick tapped on the map cannot be saved or passed; a saved pin cannot be relisted or weighted), and the geojson properties carry only `name, fit, why, type, rating, price, address, place_id`, so there is nothing more it could show without a lookup into `curRecs` / `SAVED` by `place_id`. Ian asked for "their card, or an expanded card" from a pin.
- **Receipt.** Fine, but the pick receipt offers no undo.

## 2. Data the card could show

**Already on the client, unused:** `caution`; `user_rating_count`;
`business_status` on candidates; `added_at`, `origin`, `weight`,
`anchor_facets`, `note` on saved places; the sight (`near`) and `reachM` for a
distance; `data.anchors` with provenance and `data.facetLabel` (only used for
the one italic line above the picks); `data.briefed`.

**On the server, not sent.** For every shortlisted candidate the engine holds
`Scored.dossier` (`signature[]`, `vibe[]`, `best_for[]`, `format[]`,
`caveats[]`, `price_band`, `thin`, `confidence`), `briefMention.why` (the
line the town brief gave for it), `distanceKm` and the pre-rank `parts`.
`diag.shortlist` already ships name / score / dossier-kind on every response
(the client prints it only with `?debug=1`). Adding a trimmed `dossier` and
`brief` to `Pick` in `src/engine/v2/types.ts` and where picks are assembled
(`src/engine/v2/recommend.ts`, the `recommendations.push` near line 276) is
a small server change and the single biggest unlock for the card: a "what
it's like" line from evidence rather than from the model's prior.

**Not available without a cost decision:** photos, hours, phone, website.
Photos and hours move the Places SKU (CLAUDE.md field-mask rule); the README's
line still holds: until Muninn has its own photos and hours, every card hands
you off to Maps. Keep the Maps link; do not add photo fields in this session.

**ToS.** Cards render the live response; nothing new gets persisted for them.

## 3. Constraints the design has to respect

- **The chart.** Cream paper, ink, no gloss (SHADERS.md, "the identity is flat
  ink on cream"). Tokens live in `beta.css` `:root` with `html.dark` overrides;
  every new colour needs both. Instrument Serif for titles and italics, Hanken
  Grotesk for body. The fit pill's green and the two swipe tints are the only
  non-ink colours on a card; keep it that way.
- **Geometry.** The stage is at most 520px wide. `@media (orientation:landscape)
  and (max-height:520px)` moves the nav to a left rail. Sheets are `inset:0`
  under the nav; the nav out-z-indexes them and sheets pad their bottom.
- **Gestures.** `makeSwipeable` captures the pointer on `pointerdown`, ignores
  targets inside `a, button`, and hands a mostly-vertical move back to the
  sheet. Anything interactive inside a card must be an `a` or a `button` or a
  touch on it starts a swipe. The weight dial does its own pointer capture.
- **Hard-won facts that bit cards** (ROADMAP.md): `.card{flex:0 0 auto}`, or
  `overflow:hidden` collapses every card to nothing; `#thing:not([hidden])`
  beats the UA `[hidden]` rule; `pointercancel` is the system taking the
  gesture away, not a `pointerup`; 16px inputs or iOS zooms on focus.
- **Scope wall.** `public/app.js` and `public/index.html` stay untouched.
  `auth.js` is shared.

## 4. Proposed scope, in order

1. **One renderer.** A `public/beta/cards.js` that renders a place card from
   `{ place, kind, context }` and is handed to both `beta.js` and `saved.js`,
   replacing `pickCardHtml`, `savedCard`, `card`, `foundCard` and the
   popover's inline markup. Variants by context, not by copy-paste; no inline
   styles left.
2. **The pick card, fed by the engine.** Caution as an italic ink line;
   distance from the sight on near-me; a "what it's like" line from the
   dossier and a "locals say" line from the brief once the server sends them
   (§2); the anchor the `why` names, already in prose, left as prose.
3. **Actions that teach.** Save asks favorite (been there) or want-to-go and
   offers the optional note, like the found card. Pass offers the one-line
   optional reason. After a save, the anchor chip ("One of your coffee
   anchors? yes / not quite") when the facet has fewer than five, at most one
   per session, posting to `/api/anchor` (TASTE_ENGINE §4.2.1). Receipts get
   an undo.
4. **Saved cards, once.** The results-sheet saved card becomes the ledger
   card. Note editable in place. Weight dial only captures a horizontal drag.
5. **Pin → card.** Tapping a pin opens the real card (with actions) in the
   popover, looked up by `place_id` from `curRecs` or `SAVED`; a pick can be
   saved from the map. Tap another pin and the card swaps without a flash.
6. **Check on the phone** (Ian's default). The rig only for the gesture work
   in 3 and 4.

Not in this session: photos or hours (SKU), the sheet itself, the raven
choreography, the classic app, per-aspect ratings (decided as a single dial,
TASTE_ENGINE §4.5).

## 5. Testing notes

- Local: `PORT=4600 npx tsx server/server.ts`, then `/beta/`. Real searches
  need `GOOGLE_PLACES_API_KEY` and `ANTHROPIC_API_KEY`; `/beta/?debug=1`
  prints the engine's diag, including the shortlist and which places had a
  full or thin dossier, under the cards.
- Layout-only work can be checked without keys: `window.__muninn` exposes
  `saved` and the sheet openers, so a fixture of picks can be rendered from
  the console once the renderer is a module.
- The headless rig (playwright, `/opt/pw-browsers/chromium`,
  `--enable-unsafe-swiftshader`, tile cache) is described under "The test
  rig" in ROADMAP.md. Block the service worker in ordinary rigs. Kill
  `chrome` by exact name, never `pkill -f`.
- Syntax check before a push: copy `beta.js` to a `.mjs` and `node --check`
  it (ES module).
