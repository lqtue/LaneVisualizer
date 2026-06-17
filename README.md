# LaneVisualizer

A browser-based visualizer for OpenStreetMap lane and road attributes, focused on
Vietnam road data as part of a crowdsourcing / data-quality workflow. It renders a road's
lanes, speed limits, turn lanes, destination signs, intersections and more as a schematic
diagram synced to a live map — entirely client-side, no backend.

It is a SvelteKit rewrite of the Perl tool
[mueschel/OSMLaneVisualizer](https://github.com/mueschel/OSMLaneVisualizer), extended with
a completeness dashboard, intersection detection, an official-length comparison, and a
resizable split diagram/map view.

The app lives in [`osm-lane-visualizer-svelte/`](osm-lane-visualizer-svelte/).

## Features

- **Lane diagram** — per-segment lanes with direction, `turn:lanes`, `maxspeed[:lanes]`,
  destination/exit signs, access, shoulders, sidewalks, placement and change lines.
  Emits the original tool's CSS class names, so the upstream stylesheets are reused as-is.
- **Completeness dashboard** — length-weighted coverage % for maxspeed, lanes, turn,
  surface, lit, width, shoulder, sidewalk, bridge/tunnel and access; plus a maxspeed
  value distribution.
- **Length comparison** — OSM raw length vs a decoupled (dual-carriageway) estimate vs the
  **official length from Wikidata** (property P2043, via the relation's `wikidata` tag).
- **Intersection detection** — finds roads crossing the loaded one (a node shared with
  another highway), shown as a per-segment count broken down by road type, with amber map
  markers that link straight to Mapillary imagery for sign checking.
- **Split view** — diagram on the left, full-height Leaflet map on the right, with a
  draggable divider; hover a way to preview it, click a way row to zoom the map to it.
- **Vietnam route shields** — `CT` expressway (green), `QL` national (blue),
  `DT/ĐT/TL` provincial, `AH` Asian Highway. German (`de`) and Belgian (`be`) packs included.

## Quick start

```bash
cd osm-lane-visualizer-svelte
npm install
npm run dev      # open the printed localhost URL
```

Pick country **vn**, enter a relation ref such as `QL.51` (or a way/relation id), and
Go. Data is fetched live from the Overpass API (with mirror fallback) in the browser.

```bash
npm run build    # static site -> ./build  (adapter-static, prerendered SPA)
npm run check    # type-check (svelte-check)
```

## How it works

A single ported class (`src/lib/osm/visualizer.ts`) loads OSM data from Overpass into a
multi-slot store (road / adjacent ways / crossing roads), chains the ways by their shared
end nodes, and emits an HTML lane diagram. `src/routes/+page.svelte` orchestrates the
`SearchPanel`, `StatsDashboard`, `DiagramView` and `MapView` modules and wires the
diagram to the map. Official lengths come from `src/lib/osm/wikidata.ts`.

See [`CLAUDE.md`](CLAUDE.md) for the full architecture notes.

## Tech

SvelteKit 2 · Svelte 5 (runes) · TypeScript · Leaflet · adapter-static (client-only SPA) ·
Overpass API · Wikidata · Mapillary links.

## Credits

Lane-rendering logic and stylesheets are derived from
[mueschel/OSMLaneVisualizer](https://github.com/mueschel/OSMLaneVisualizer). Map data ©
OpenStreetMap contributors.
