# Muninn Beta — Shader / Atmosphere Brief

For the artistic pass on Fable. The chart aesthetic is settled and Ian likes
it; this is about giving it **depth and life** without breaking the flat
ink-on-cream identity that took a dozen rounds to find.

Read [`ROADMAP.md`](ROADMAP.md#hard-won-facts-measured-not-assumed) first —
especially the engine-locked doctrine and the MapLibre facts. Several
"obvious" approaches are already known to be dead ends.

## What is on screen today, and how it is drawn

The stage is a stack of DOM layers over one MapLibre canvas. Nothing is a
shader yet; everything painterly is CPU canvas work or CSS.

| z | layer | what it is | how it is made |
|---|-------|-----------|----------------|
| 1 | `#map` | the chart itself | MapLibre GL v5.6, globe projection, OpenFreeMap vectors restyled at runtime by `inkify()`, plus a sepia hillshade from AWS terrarium tiles |
| 2 | `.orb` | limb tone | a CSS radial-gradient div, sized/placed from the measured globe circle |
| 2 | `#rimcv` | ring frame | inline SVG, two circles + degree ticks, two bands counter-rotating |
| 2 | `#clouds` | cloud deck | **CPU**: `renderClouds()` walks a screen grid, calls `map.unproject` per cell, bilinear-samples a 512×256 mask, writes `ImageData`. 8px cells while moving, 5px settled |
| 2 | `#cloudshadow` | cast shadow | the same deck re-tinted sepia and offset by the sun azimuth |
| 4 | `.grain` | paper tooth | a 256px generated PNG tiled with `mix-blend-mode: overlay` |
| 5-9 | chrome | label, coin, rose, sheets | DOM |

Land cover uses generated **pencil `fill-pattern`s** (`pencilTileData`) —
hatch, cross, scribble, stipple, and scallop "waves" for water. The sun's
azimuth drives `hillshade-illumination-direction` and the shadow offset
through the day (`paintSky`, `sunAzimuth`).

## Where a shader can actually plug in

Three viable routes, in rough order of power:

1. **MapLibre custom layer** (`{type: "custom", render(gl, args)}`). Runs
   inside the map's own WebGL2 context with its projection matrix, so it is
   correctly placed on the globe at every zoom for free, and composites with
   the map's own draw order. This is the right home for anything that must sit
   *on* the sphere: atmosphere, terminator, ocean shimmer, ink bleed.
2. **A WebGL canvas overlay** replacing `#clouds`. Decoupled from the map, so
   it must be engine-locked by hand — but the machinery already exists: pass
   the measured `globeCircle` plus the map centre/zoom as uniforms and do the
   inverse projection in the fragment shader instead of per-cell on the CPU.
3. **SVG filters / CSS** (`feTurbulence`, blend modes) for paper and plate
   effects that do not need to track the sphere. Cheapest, and honestly the
   right tool for the paper itself.

## The work, in the order I would do it

### 1. Paper that behaves like paper (start here — highest ratio)
The grain is a tiled 256px PNG at `opacity .45`. It reads as noise, not as a
sheet. A shader (or `feTurbulence`) can give: fibre with direction, a faint
plate-tone gradient that is *not* radially symmetrical, subtle blotching where
ink pools, and — the detail that sells it — the tooth staying **fixed to the
screen** while the map moves beneath, the way a sheet does not slide when you
move a magnifier over it. Cheap, always visible, affects every screenshot.

### 2. Clouds on the GPU
`renderClouds` is the one genuinely hot loop: an `unproject` call and a
bilinear sample per grid cell, every frame the map moves, which is why it runs
at 8px cells while moving and only sharpens when settled. Moving the inverse
projection into a fragment shader would allow per-pixel clouds, soft edges,
real drift, and self-shadowing, and would drop the coarse/fine switch
entirely. Keep the NASA mask as a texture; keep the procedural mask as the
polar fill (see the polar-night note in ROADMAP).

### 3. Limb and atmosphere
`.orb` is a CSS gradient approximating limb darkening. On a custom layer this
could be a real rim: a thin ink halo that thickens toward the terminator, a
touch of scatter on the lit edge, and the paper showing *through* the sphere
at the edge rather than a hard cut. This is the single biggest "it looks
painted" win after paper.

### 4. Day/night terminator + city lights — DONE as geometry (Sep 2026)

Built without a shader: ten twilight bands as clipped latitude strips, city
lights and saved-place lamps as circle layers with a sun-altitude expression.
See `paintNight` in `beta.js` and the ROADMAP entry. A shader version would
only smooth the band edges; not worth a GL context of its own.

#### (original brief, kept for the record)
Explicitly saved for this pass. `sunAzimuth(hf)` already tracks the real sun.
The plan discussed with Ian: a sepia-blue night wash over the dark half,
feathered at the terminator, with warm pinpricks at populated places
(`cities.json` carries 1,249 places with population). Must stay ink-on-paper —
this is a chart lit by a reading lamp, not a NASA night-lights photo.

### 5. Ocean and ink
The water is a static scallop `fill-pattern`. A slow shader-driven swell that
moves *with the sphere* (engine-locked, not screen-locked) would make the
planet feel alive at rest. Related: ink bleed and plate wear along coastlines,
which is where engravings get their character.

## Constraints — all of these are real

- **Mobile first.** Ian tests on a phone; the beta already has a **Lite**
  toggle that drops pixel ratio and disables the cloud deck. Every shader
  needs a Lite path and must honour `prefers-reduced-motion`.
- **$0 of new services**, and no build step. The app is plain ES modules
  served by a small Node server; libraries come from CDN if at all. Prefer
  hand-written GLSL over pulling in a framework.
- **The identity is flat ink on cream.** Two earlier attempts at "richer"
  (a lit 3D sphere, then heavy gold) were rejected. Depth should come from
  *drawing* — hatching density, plate tone, ink pooling — not from gloss,
  bloom or specular highlights.
- **Engine-locked or it drifts.** Anything that must sit on the globe derives
  from `fitSphereCircle()` / `map.unproject`, never from a zoom formula. This
  rule has been re-learned three times.
- **Verification.** The sandbox has software GL: it will tell you a shader
  compiles, runs and roughly what it looks like, but nothing about frame rate.
  Judge performance on device, and keep the Lite path honest.

## Suggested first commit

Paper (1) alone, shipped and looked at on the phone, before touching the
globe. It is the lowest risk, it is visible on every screen including the
sheets, and it establishes whether shader work belongs in the DOM stack or
inside the map's context — which decides the shape of everything after it.
