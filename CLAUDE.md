# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A two-part workspace for visualizing OpenStreetMap lane/road attributes, focused on Vietnam road data as part of a crowdsourcing/quality-checking system.

- **`osm-lane-visualizer-svelte/`** — the active project. A SvelteKit (static, client-only) rewrite of the original Perl tool. **All work happens here.**
- **`OSMLaneVisualizer/`** — the original Perl CGI app (mueschel/OSMLaneVisualizer), kept **read-only as reference**. The Svelte port is a faithful translation of its `OSMData.pm` + `OSMLanes.pm` + `OSMDraw.pm` + `render.pl`. Consult it to understand intended behavior; do not modify it.

## Commands

All commands run inside `osm-lane-visualizer-svelte/`:

```bash
npm install
npm run dev      # vite dev server (note: has been running on port 5181)
npm run build    # static site -> ./build (adapter-static, prerendered SPA)
npm run preview  # serve the production build
npm run check    # svelte-kit sync + svelte-check (type check)
```

There is **no test suite**. After any change to `visualizer.ts` or the components, verify with **`npm run build` AND `npm run check`**.

**Known-acceptable `svelte-check` errors (pre-existing, do NOT try to "fix" as part of unrelated work):**
- 4× `makeShoulder`/`makeSidewalk` "comparison appears unintentional ... 'no'" in `visualizer.ts` (faithful port of Perl logic).
- 1× `leaflet` "Could not find a declaration file" in `MapView.svelte` (no `@types/leaflet` installed; Leaflet is used as `any`).

A clean run = exactly these 5 errors and nothing new.

## Architecture

Client-only SPA. There is no backend — all data is fetched from public APIs directly in the browser. `src/routes/+layout.ts` sets `ssr = false; prerender = true`.

### Data flow (one search)

```
SearchPanel (build Overpass query)
  -> +page.svelte runSearch()
       new LaneVisualizer(opts)
       v.readData(query, 0)            // main road ways  -> store slot 0
       v.organizeWays()                // chain ways by shared begin/end nodes
       [opts.adjacent]  v.readData(buildAdjacentQuery(), 1)
       [opts.intersections] v.readData(buildCrossingQuery(), 2); v.computeIntersections()
       result = v.render(start)        // -> { html, wayCoords, stats, intersections, rawLengthKm, ... }
       v.getRelationMeta() -> wikidata -> fetchOfficialLengthKm()
  -> StatsDashboard (coverage %, lengths, intersection counts)
  -> DiagramView  {@html result.html}  (left pane)
  -> MapView (right pane, Leaflet)
```

### `src/lib/osm/visualizer.ts` — the core (one big ported class)

`LaneVisualizer` holds a multi-slot `store` (`store.way[0|1|2]`, `store.node[...]`, `store.rel[...]`): slot 0 = the road itself, slot 1 = adjacent ways, slot 2 = crossing roads. Key pieces:

- **HTML emission**: `drawWay()` + `render()` emit HTML using the **exact same class names** as the original `style.css`/`<country>.css`. This is deliberate — the CSS in `static/css/` is reused verbatim, so changing emitted class names breaks styling. New visual elements get new classes added to `static/css/style.css`.
- **Lane parsing** (`getLanes`, `inspectLanes`, `getLaneTags`): faithful port. One quirk preserved intentionally — pushing `undefined` makes `definedMax` true, so the `lanes/2` symmetric-split default is dead code, matching Perl. Result: a non-oneway `lanes=4` way with no directional split renders 1 fwd + 1 bck + 2 hatched `nolane`.
- **Geometry**: `calcDistance`/`calcLength` (return meters, equirectangular approx), `calcDirection` (custom Perl-convention bearing), plus a standard `compass()` (atan2, 0=N clockwise) used only for intersection turn direction.
- **Stats** (added beyond the original): `collectStat()` per way feeds `stats: WayStat[]` (length + length-weighted coverage flags); `render()` returns `rawLengthKm`.
- **Intersections** (added): `buildCrossingQuery()` (`way(bn.<our nodes>)[highway]` minus our own, with `>;out skel` for coords) + `computeIntersections()` → a node shared with another highway is a junction. Filters non-vehicular classes and ways whose `ref` matches our road (continuations). Per-segment counts render as a `.xcount` badge; per-node markers + Mapillary links go on the map.
- **Country shields**: `refClass()` maps refs to shield classes per country. Vietnam (`vn`): `CT→A` (expressway, green), `QL→B` (national, blue), `AH|E→E`, `DT|ĐT|TL→K` (provincial).
- **Overpass resilience**: `fetchOverpass()` walks `OVERPASS_ENDPOINTS` (de → kumi → mail.ru mirror); treats any body not starting with `{` as an error page (avoids the `JSON Parse error: '<'` crash) and throws a friendly message only if all mirrors fail.

### Components (`src/lib/components/`)

Thin modules orchestrated by `+page.svelte`:
- **SearchPanel** — owns the form/config state, builds the Overpass query via the exported `wayQuery`/`relQuery`/`relNameQuery`/`relRefQuery`, emits `onsearch(SearchRequest)`. `country` is a `$bindable` shared with the page (drives the `<country>.css` link).
- **DiagramView** — renders `{@html html}` and delegates DOM events: hover `.label` → `onhover`, click `.label` → `onzoom`, click `[data-wayid]` (the `(V)`/arrow links) → `ondrill` (reload that single way).
- **MapView** — encapsulates Leaflet; exposes imperative `preview()`/`zoom()`/`resize()` via `bind:this`. `WayGeom.crossings` become amber markers with Mapillary popups.
- **StatsDashboard** — `$derived` coverage %, length headline (raw vs decoupled estimate vs Wikidata official), intersection counts.

`+page.svelte` also implements the draggable split between DiagramView and MapView (pointer events adjust `grid-template-columns`, calling `mapView.resize()` during drag).

### Conventions specific to this codebase

- Svelte 5 runes (`$state`, `$derived`, `$props`, `$bindable`). Callback props, not `createEventDispatcher`.
- The diagram emits raw HTML strings (a port of the Perl), injected with `{@html}`. Escape user/tag-derived text with `escapeEntities()`; HTML uses entities (`&#x2191;`), not decoded unicode.
- `import { sveltekit } from '@sveltejs/kit/vite'` in `vite.config.ts` (NOT from `vite-plugin-svelte` — that import name doesn't exist there).
- Adding a country = add `static/css/<code>.css` (reuse `style.css` classes) + a branch in `refClass()` + an `<option>` in SearchPanel.
