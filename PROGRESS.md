# Progress

The per-session handoff CLAUDE.md asks for: what changed, what is open, what
is next. Update before ending every session. Longer-lived planning stays in
`docs/ROADMAP.md`; this file is the current page. `PROGRESS.md`, `CLAUDE.md`,
`README.md` and `docs/` are mirrored to a public repo, so no keys, env values,
user emails or user data go in them.

## Next session — the cutover

Focus: retiring the classic app and serving the beta at `/`. The audited
task list is [`docs/CUTOVER.md`](docs/CUTOVER.md); work section A in
order, one item per session.

**A1, the owner sheet, shipped 9/7** (`public/beta/owner.js`, the fourth
nav tab that appears when `/api/me` says owner): import approvals, the
feedback inbox, the users list with drill-down. Driven in headless
Chromium against stubbed owner routes (approve, reject, resolve, the
drill-down, light and dark, the landscape rail); not yet opened on the
phone against the real queue. Worth one look there: approve a real
queued import and see the row flip to done after the refresh.

Next is **A2, feedback capture**: a way to send feedback from the beta
(kind: feature / bug / general, a message, `POST /api/feedback` with a
context string such as the current town), as a line on the taste page or
the "Who's Muninn?" panel. The owner inbox that reads it is now in place.
Then A3 (saved rows without coordinates vanish), A4 (near-me falls back
to the hometown), A5 (the Back button; the owner sheet will need its
history entry too), A6 (the saved-sheet demote asks a reason, the promote
offers a note), A7 (serving `/`, one commit with a deploy window), A8 (the
town route's heartbeat).

Rules that hold through the cutover: `public/app.js` and
`public/index.html` stay untouched until section D; `public/auth.js` and
`public/sw.js` are shared with the classic app, so a change there must
keep it working until the switch (A7 is where `sw.js` changes); every new
beta file carries the `__V__` token; a push to `main` restarts the server,
so batch docs with code and do not push while Ian is testing a town.

State on 9/7 night: `main` and the feature branch are level (task 5, the
miles setting, the heartbeat, the cutover list). Two things still worth a
glance on the phone: a cold town at a slow hour comes back (the heartbeat
on `/api/nearby-recommend`), and a second search in a re-keyed town
(Naples, Jupiter, Clearwater) reads its brief (`?debug=1`, `brief: "N
places"` not `"writing"`). Engine changes stay out unless a cutover item
needs a field the server does not send (`src/engine/v2/recommend.ts`, the
`Pick` type in `types.ts`, `handleNearbyRecommend` in `server/server.ts`).

### What shipped 9/6–9/7, for the record

Place cards (one renderer, the vial, the popover card, the place page with
journal, dossier, correction and anchor; ROADMAP "Place cards"); the
teaching actions on a pick (Save asks the list and a note, Pass asks a
reason, the anchor chip after a favourite); the "Place cards" schema block
applied in Supabase and the St. Pete-area favourites backfilled through
`BACKFILL_DOSSIERS` (134 places, about $6; a wider run is task 11 below);
the ring as the search instrument; task 18, miles or kilometres; task 5,
one town one brief (settlement geocoder, nicknames, canonical brief key,
the table re-keyed by hand); the heartbeat on the near-me route after a
cold Tampa outlived Cloudflare's 100 s limit.

## Where the engine stands (end of 2026-09-06)

v2 is the default on main and has been verified across a dozen towns on
the phone. What it does now, in one paragraph: three rich Places searches;
the town brief (Sonnet + Anthropic web search + Brave snippets for the
sites Anthropic cannot read) read from cache or started after the search's
own model calls; dossiers cached per place, Haiku writing the missing ones
in parallel batches with a fifteen-second guard; a deterministic pre-rank
to sixteen; Sonnet scores every row with a fit and a reason, with a cached
prompt prefix; the server checks reasons for cross-references to other
candidates; anchors are facet-conditional, classified by type or by name,
merged across favourite/want-to-go twins, and a general search covers every
kind on the shortlist. `?debug=1` shows the whole account of a search:
memory size, anchors, stage timings, dossier batch timings, API response
counts, the rank's stop reason, cache hit, usage, and the shortlist.

Caches in the database as of this afternoon: about 120 dossiers, 14 town
briefs (Apopka's the first with Brave snippets, 30 of them). Google's
Custom Search API turned out to be closed to new customers; Brave is the
provider (`BRAVE_SEARCH_API_KEY` on Render, set 9/6).

Speed, measured and not ours to fix: a cold town costs 15–25 s, a warm one
8–12 s. Both models run two to four times slower at some hours than others
with every response 200; the diag lines prove it. Stop instrumenting this.
The levers that remain are structural (pre-brief and pre-dossier likely
towns, or a faster rank model) and belong to a later, deliberate session.

## Tasks — fix or work on later

Ian went through the list on 9/6 night; this is what stands. Numbers are
his; the ones he closed are struck.

1. ~~Saving a pick makes two pins~~ — fixed 9/6.
2. ~~Home-country lean~~ — fixed 9/6.
3. ~~Anchor prompts in the UI~~ — the after-save chip (once a session, a
   specific facet with fewer than five anchors) and the page's "An anchor?"
   section shipped 9/6 night. Wording is "anchor", never "yardstick".
4. **Undo on the receipt line** after a save or a pass (Ian: later).
5. ~~"St Pete" is three towns and one of them is the beach~~ — shipped 9/7
   on the feature branch (ROADMAP "One town, one brief"). The client asks
   the geocoder for settlements, expands "st pete" and a few nicknames, and
   sends the canonical "City, Region"; `briefKey` canonicalises the metro
   (`canonicalTown`, `src/lib/geo.ts`). The table was re-keyed by hand the
   same night (no town re-briefs), the St. Pete triplicate collapsed to the
   richest brief, and the "Lunch" / "Coffee" / "Coffee in naples FL" rows
   deleted. Verified on the phone 9/7 after the merge to `main`: "st pete"
   read the cached brief under `briefKey: "st petersburg florida|food-drink"`
   (25 places, 30 snippets), 32 searched, 24 discoveries. Open: the classic
   app and the CLI still send the typed spelling (the key absorbs
   punctuation, "saint" and state codes, but a nickname there stays a
   nickname); a typed request with no town ("coffee") still geocodes to
   whatever carries the name.
6. ~~"N more" card spacing~~ — skipped for now.
7. ~~The fit scale~~ — a coincidence; towns land properly now.
8. **Eval baseline** (ROADMAP 12): `npx tsx scripts/eval.ts --engine v2
   --limit 30`; record recall@6 in TASTE_ENGINE.md.
9. **Re-resolve the import's types** (legacy "establishment; food" rows).
10. ~~Reddit coverage in Brave~~ — checked 9/6 night: across all 20 briefs,
    none of the sources the model cites are reddit.com; two of the three
    Brave queries per brief are `site:reddit.com`, but which hosts the
    snippets came from was never stored. Briefs now record
    `snippet_hosts` (host → count); read a few new briefs' rows to answer
    it properly.
11. **Wider dossier backfill.** St. Pete favourites are done (134, about
    $6). Everything else builds on first view. A wider run is
    `BACKFILL_DOSSIERS` on Render (4–5¢ a place; some 2,000 saved places).
12. **Coin-colour legend** — one quiet line near the "places remembered" count.
13. **Share-as-image** from the taste card, later the place page.
14. **Logo** — the coin face with the birds and crescent.
15. **Lens settings**, if the lens coin returns.
16. **A search with no memory** (9/6, once): `diag.memory` now reports it;
    watch for a repeat.
17. **Dossier model experiment** (`DOSSIER_MODEL=claude-sonnet-5`).
18. ~~Miles or kilometres, a setting~~ — shipped 9/7 (ROADMAP "Miles or
    kilometres"). Kept on the device (`mn-units`), not the profile: no
    schema change, works in single-user mode, and the locale default is
    per device anyway. Every label reads `distLabel` in `cards.js`; the
    reach scale re-rules and re-snaps in the unit. If Ian wants it to
    follow him across devices, add a `units` column beside the hometown
    and mirror it into localStorage when `/api/me` answers.

## What changed this session (9/4–9/6), for the record

Everything below is in `docs/ROADMAP.md` (shipped entries and hard-won
facts) and `docs/TASTE_ENGINE.md` (status); this is the index.

- v2 was silently falling back to v1 (structured-output schema keywords);
  the dossier cache had no `snapshot` column; briefs never ran (an
  `allowed_domains` list naming sites that block Anthropic's crawler),
  then timed out at the SDK's ten-minute limit. All fixed; every model
  call now logs its failure and a brief stores its own error.
- Every shortlisted place is scored; the sheet tiers at 70 / 50; a thin
  town shows its closest three honestly.
- Reasons may only name the person's own places; a cross-reference is
  replaced and counted.
- Anchors: general searches cover the shortlist's kinds; places with
  generic types are classified by name; favourite/want-to-go twins share
  types; an account with no favourites anchors on want-to-go.
- The search box splits "coffee in naples fl" into a request and a town.
- Brave Search snippets feed the brief (Google's API is closed to new
  customers).
- Latency: parallel brief-name resolves, cached dossiers read beside the
  brief, extraction capped and batched, the brief started after the rank,
  the durable card rounded so a save no longer breaks the prompt cache.
  Measured and documented; further gains are structural.
