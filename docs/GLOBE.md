# Muninn — Globe-as-Interface (design direction)

**Status:** endorsed direction; **interactive prototype built** (globe.gl), not yet
wired into the app. This is the visual/interaction spec for the beta home surface.

## Decisions locked (as of the beta design pass)

- **Isolation:** the revamp ships at a **`/beta` path on the same deploy** — new files
  under `public/beta/`, reusing the existing `/api/*` backend + real Supabase data.
  Zero risk to the live app (its files are never touched). Promote later by pointing
  `/` at the beta files.
- **Globe library:** **globe.gl** (three.js) for the real interactive globe, with a
  **cobe** "Lite / low-power" mode toggle (production; the prototype's Lite is a
  stripped-down globe.gl for now).
- **Framework:** stay **vanilla, no build step** (matches the app; CDN libs like the
  live app already uses Leaflet).
- **Aesthetic (refined):** **Studio-Ghibli mythical × modern digital.** The globe,
  map, atmosphere, and raven are painterly/illustrated and storybook-warm; the text,
  buttons, and chrome stay crisp modern-digital. Time-of-day sky (below) sets the mood.
- **Type:** Instrument Serif (mythical serif wordmark/headers) + Hanken Grotesk (clean
  modern body/UI).

## Interactive prototype (built, live)

A working globe.gl prototype is published as an Artifact (not in the repo — it's a
design deliverable). It demonstrates the real feel; the `/beta` build ports these:

- **Flick-to-spin, no button** — real OrbitControls with damping/inertia; drag+release
  flings the globe, then it eases back into slow ambient rotation. Pinch-zoom on.
- **Clay planet** — matte `globeMaterial`, a warm "sun" light + cool fill giving a soft
  terminator so it reads as a touchable object.
- **Real geography** — 177 countries via `polygonsData`, extruded slightly with a dark
  side-wall + faint warm borders (raised clay continents). Prototype inlines simplified
  **110m** borders; **production uses 50m** (crisper coasts) fetched/bundled server-side.
- **Time-of-day sky** — the background gradient is computed from the **device clock**
  (night → dawn → morning → midday → golden hour → dusk → nightfall), re-checked every
  30s; the globe atmosphere colour follows it.
- **Zoom → city labels → "Search this area"** — zoom into a region and city labels
  (`labelsData`) fade in; an accent pill reads the globe's centre and offers *Search near
  <city>*. Tap → raven loader → ranked picks for that area. This is the spatial entry to
  Muninn's search: in `/beta` it reverse-geocodes the centre to a town and calls
  `/api/recommend` (picks in the prototype are labelled samples).
- **Saved places** glow as warm points; recent ones pulse (`ringsData`); travel arcs
  animate from home (`arcsData`).
- **Lite toggle** — low-power/low-data mode (flattens polygons, drops arcs/rings, lowers
  pixel ratio); production swaps to cobe.

## How to render geography on the globe (reference)

- **Countries:** `polygonsData(geojson.features)` with `polygonCapColor` (matte land),
  `polygonSideColor` (dark wall = depth), `polygonStrokeColor` (borders), `polygonAltitude`
  (raise). Data = `world-atlas` TopoJSON → GeoJSON via `topojson-client`. Alt style:
  `hexPolygonsData` (dotted-hex countries).
- **Cities:** `labelsData([{lat,lng,text}])` with `labelText/labelSize/labelColor`.
- **Graticule:** `showGraticules(true)`. **In the artifact sandbox** the GeoJSON must be
  inlined (no runtime fetch); **in `/beta`** just bundle/fetch it.

## Ghibli-mythical × modern-digital treatment

Push the *globe/map/raven/atmosphere* painterly; keep *UI chrome* crisp:
- Painterly, warmer skies; drifting soft clouds/mist over the globe (SVG turbulence or a
  cloud layer); a faint starfield at night; lantern/firefly-warm point glows.
- Warm, storybook land colours; a softer, larger atmosphere halo; gentle bloom + grain.
- A more graceful, ink-wash raven (character, not just a silhouette).
- Buttons, inputs, nav, cards stay modern-digital: clean grotesk, sharp controls, glass.

## Painted assets (Hades-style) — to generate

Target look: **Supergiant / Hades** — hand-painted surfaces with light and depth
*baked into the brushwork*, ink outlines, gold rim-light, dramatic mood. A lit 3D
sphere can't fake this; it needs real painted textures wrapped on the globe, plus
bloom post-processing. **Split the work:** an image model paints the *surfaces*
(no geographic accuracy needed); the accurate continents come from our existing
GeoJSON, composited on top into one seamless equirectangular globe texture.

Generate these (Midjourney / DALL·E / Firefly / etc.), drop the files in the repo,
and the wiring composites + bloom is a code step:

1. **`ocean.png`** — seamless, horizontally-tileable painterly water. *No land, no
   horizon, flat top-down texture.* ~1024² (I mirror/blend edges + tile to 2:1).
   > *Prompt:* "seamless tileable hand-painted deep ocean texture, dark teal and
   > indigo with subtle warm gold caustic highlights, Hades video game art style by
   > Supergiant Games, painterly visible brush strokes, moody and dramatic, flat
   > top-down, no land, no coastline, no horizon, repeating pattern"
2. **`land.png`** — seamless, tileable painterly terrain to fill the continents.
   *No shapes, no map — just surface.* ~1024².
   > *Prompt:* "seamless tileable hand-painted terrain texture, warm ochre earth and
   > deep mossy green with gold flecks and ink shadows, Hades video game art style,
   > painterly brush strokes, flat, no borders, no shapes, repeating pattern"
3. **`raven-fly.png`** — the character raven, **transparent background**, side
   profile, wings spread mid-flight. ~1200px wide.
   > *Prompt:* "a mythological raven in flight, side profile, wings spread,
   > ink-black feathers with glowing amber-gold rim light, painterly illustration,
   > Hades video game art style by Supergiant Games, dramatic lighting, isolated on
   > transparent background"
4. **`raven-perch.png`** *(optional)* — resting raven for empty states. Same style,
   perched, transparent background.

Delivery: PNG (raven must be transparent), sRGB. Put them in `public/beta/art/`.
In the Artifact prototype they're base64-embedded (CSP blocks fetch); in `/beta`
they're normal files.

Wiring (code step, after assets land): composite `ocean` + `land` + GeoJSON
coastlines (gold ink) into one equirectangular canvas → `globeImageUrl`; add an
`UnrealBloomPass` via `postProcessingComposer()` so points/atmosphere glow like
Hades; raven art replaces the SVG in the loader + wordmark.

## One engine — planet to street (round 2 BUILT — live at /beta)

**Status:** the beta now runs entirely on **MapLibre GL v5 globe projection** —
one continuous camera from spinning planet to street level, which is what makes
it Google-Earth smooth: there is no globe↔map handoff left to mask. The round-1
two-engine build (globe.gl painted sphere + cloud-dive swap to MapLibre) is
retired; the baked 4K painting, texture cache, world.geo.json and the dive state
machine are gone. The whole world is ink-styled vectors (`inkify()` recolors
OpenFreeMap's style, planet included), the NASA/procedural cloud canvas rides
the sphere and you descend through the cloud layer naturally around z3.2–4.3,
and search does a flyTo swoop around the planet. Sky/stars/grain/vignette DOM
wrap, saved-place lanterns, "Search near X" (cities.json + Nominatim), and the
raven recommendation sheet all carried over. Verified headlessly at planet,
region and street zooms with real tiles.

Historical (round 1, superseded): painted globe.gl sphere + MapLibre town layer
joined by a cloud-dive transition; kept in git history if the baked-painting
look is ever wanted for marketing art.

The globe's zoom floor is architectural: a whole-planet texture can't hold street
detail. Town zoom needs streamed tiles, so the beta becomes **two connected
surfaces** with a cloud transition between them. Decisions locked with Ian:
**MapLibre custom style** for the town layer; **clouds all-in incl. NASA daily**.

Why not stylized satellite: photos are the wrong raw material for this look — a
filter over satellite pixels reads muddy, and Google's imagery ToS forbids
modifying/re-serving it (plus billing). Vector data = shapes you apply paint
rules to; NASA is the free satellite source and supplies only the clouds.

1. **Town layer — MapLibre GL + "Muninn ink" style.** MapLibre GL JS rendering
   OSM vector tiles from **OpenFreeMap** ($0, **no API key, no billing surface**);
   scale-up path: self-hosted Protomaps `.pmtiles`, still $0. Custom style JSON:
   moss/parchment land, painted-teal water, gold-ink roads on ink casing,
   clay-extruded buildings at close zoom, serif-flavored labels, minimal POI
   noise. The existing grain/vignette/bloom stack wraps the map so it stays
   painterly. Saved places + picks render as gold lantern markers (clustered
   low-zoom); **"Search this area"** = map center → reverse geocode →
   `/api/recommend`.
2. **The cloud dive.** Zoom past the globe floor (or tap "Search near X") →
   procedural clouds billow over (~600ms) → part onto the town map at the same
   lat/lng, same sky, same HUD. Reverse on zoom-out. Reduced-motion: crossfade.
   Map instantiated lazily on first dive, kept warm after; globe render pauses
   while the map is up.
3. **Clouds, three layers.** (a) Dive puffs — procedural, no assets. (b) Ambient
   globe clouds — a second slightly-larger sphere with generated fractal-noise
   alpha texture rotating slower than the planet (replaces the DOM mist).
   (c) **NASA GIBS daily** (free, no key): fetch a low-res daily true-color
   snapshot, extract a cloud mask (bright/desaturated → white alpha), drape
   yesterday's real weather on the cloud-sphere; cached daily, procedural
   fallback. GIBS only works in `/beta` (artifact sandbox blocks fetch).

**Build order:** ① scaffold `public/beta/`, port the globe prototype → ② MapLibre
+ ink style + dive transition → ③ wire search/markers to real API + saved places
→ ④ clouds (procedural → NASA) → ⑤ perf/Lite/reduced-motion pass. ~2–3 sessions.
New recurring cost: $0.

## The core idea

The home surface is an **interactive 3D globe**, not a text box. Muninn's whole
conceit is a raven that flies out over the world and returns with what's good —
so the globe makes the app's mental model literal: *here is the world, here is
everywhere your taste has already touched, point me somewhere new and I'll go
look.* The globe is the interface, not a splash screen.

## What the globe does (functional, not decoration)

- **Your memory, made visible.** Every saved place is a soft glowing point on the
  globe. Spinning it shows the *shape of your taste* — dense clusters around home
  markets, scattered pins along travel routes. This is the "accumulated memory"
  thesis rendered as a picture no competitor can show.
- **Arcs = your travels.** Thin animated great-circle arcs from home base out to
  each town (or between places saved on the same trip) — the raven-flight
  metaphor, drawn.
- **The globe IS the search.** Drag/spin to a region, tap a city → that's "search
  this town." Tap a glowing point → your saved place there. The text box stays as
  the fast path for people who already know where they're going.
- **Spin to discover.** Flick the globe; it decelerates onto a city; the raven
  flies there; picks load. (Promotes today's "Spin the globe" button in Near-me to
  the literal centerpiece.)

The globe earns its place by absorbing three things that already exist — Search,
the Near-me framing, and Spin-the-globe — into one surface.

## The raven loader (the soul of it)

Every load state shows the **raven flying a dotted great-circle arc** — ideally
from the user's location toward the place being searched, so the animation depicts
"flying out to look." Results land as the raven "arrives" and picks fade in. On
the globe, the raven flies the arc across the actual globe to the searched city —
the best version, unifying loader and home surface into one gesture.

- **Implementation lean:** inline **SVG + path animation** (`offset-path` /
  `animateMotion`) — lightest, theme-able, crisp, no dependency. Lottie only if we
  want designed wing-flap fidelity (heavier). Sprite sheet is the middle ground.
- **Restraint:** ~1.2s min so it's never jarringly quick, a hard cap so a slow API
  never traps the user watching a bird, honor `prefers-reduced-motion` (static
  fallback), and keep the motion *predictable* — predictable reads as polished,
  random reads as gimmicky.
- **Raven as a recurring character:** resting raven on empty states ("nothing here
  yet — send me out"), so the bird is a through-line, not just a spinner.

## Technical options for the globe

| Option | Weight | Feel | Notes |
|---|---|---|---|
| **globe.gl** (three.js) | Heavy (~150KB+) | Best — points, arcs, atmosphere, labels built-in | Purpose-built for points+arcs on a globe; fastest path to the vision. Watch mobile perf. |
| **cobe** | Tiny (~5KB WebGL) | Gorgeous minimalist dotted globe, buttery on mobile | Great ambient hero; tap-a-city interactivity is DIY. |
| **three.js custom** | Heavy | Unlimited | Most control, most work; overkill unless globe.gl can't do it. |
| **Leaflet (already shipped)** | Light | 2D | Not a globe, but the reduced-motion / low-power fallback. |

**Lean:** `globe.gl` for the interactive home, with a **2D Leaflet fallback** for
low-power devices and `prefers-reduced-motion`. `cobe` if we'd rather a stunning
*ambient* globe and keep search text-first.

## Palette / identity

Lean into "raven at dusk over a lit world": deep night-sky ground (the existing
dark theme), warm point-glows for saved places, the burnt-orange accent
(`--accent`) as the glow color for new picks. Warm lights against deep blue/
charcoal.

## Honest cautions

- **Mobile perf & battery** — pause render when idle/backgrounded, cap FPS, static
  until interaction, 2D fallback.
- **Time-to-interactive** — the core moment is "open app in a strange town, need a
  pick now." Fast text search must stay reachable; the globe hydrates behind it.
- **Don't bury the utility** — the globe is a 1-tap detour to results, never a toll
  booth.
- **Accessibility** — reduced-motion path + the existing list/search as a
  non-spatial equivalent.

## Phased build (highest joy-per-effort first)

1. **Raven loader** — inline-SVG arc flight wired into every existing load state.
   Small, high-delight, ships independently, validates the motion language.
2. **Ambient `cobe` globe** on the home/Search header with saved points; search
   stays text-first. Low risk.
3. **Promote to interactive `globe.gl`** — tap-a-city, arcs, spin-to-discover as
   the primary gesture, with the 2D fallback. The full vision.

## Open decisions

- Globe **replaces** the Search tab as landing, or a **new "Explore" tab** so fast
  text search stays untouched?
- Raven loader: lightweight **SVG** (lean) vs richer **Lottie** fidelity.
