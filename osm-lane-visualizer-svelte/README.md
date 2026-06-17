# OSM Lane Visualizer — Svelte port

A SvelteKit (static) rewrite of [mueschel/OSMLaneVisualizer](https://github.com/mueschel/OSMLaneVisualizer)
(originally Perl CGI). It renders OSM lane attributes — lanes, `maxspeed[:lanes]`,
`turn:lanes`, destination/exit signs, access, shoulders, sidewalks, placement,
change-lines, node signs — directly in the browser. No server: the Overpass API
is called client-side and the map uses Leaflet.

A **Vietnam country pack** (`static/css/vn.css`) is included alongside the
original `de`/`be` packs. Route shields are mapped in
`src/lib/osm/visualizer.ts` → `refClass()`:

| class | Vietnam meaning            | example ref |
|-------|----------------------------|-------------|
| A     | CT — đường cao tốc (expressway) | `CT.02`, `CT01` |
| B     | QL — quốc lộ (national highway) | `QL1A` |
| K     | DT/ĐT/TL — đường tỉnh (provincial) | `DT743` |
| E     | AH — Asian Highway          | `AH1` |
| S     | other / default            | |

> VN traffic-sign and destination-symbol images currently reuse the German
> placeholders — swap the URLs in `static/css/vn.css` for Vietnamese signage.

## Run

```bash
npm install
npm run dev      # http://localhost:5173
npm run build    # static site in ./build
```

## Try it

Pick country **vn**, set **Search by → Way by id**, enter `1228612274`
(a CT.02 expressway segment with `turn:lanes=left;through|through|none`) and click
**Search**. Or search a whole route with **Relation by ref** (`QL.51`) or **by name**.
Expand **Configuration** to toggle adjacent ways, intersection detection, lane width and more.

## Structure

| File | Origin |
|------|--------|
| `src/lib/osm/visualizer.ts` | port of `OSMData.pm` + `OSMLanes.pm` + `OSMDraw.pm` + `render.pl` loop |
| `src/lib/osm/wikidata.ts` | official road length (Wikidata property P2043) |
| `src/lib/osm/types.ts` | data model |
| `src/routes/+page.svelte` | orchestrates the modules; diagram↔map wiring; draggable split |
| `src/lib/components/*.svelte` | SearchPanel · StatsDashboard · DiagramView · MapView · ExportPanel |
| `static/css/{style,de,be,vn}.css` | reused verbatim from the original — the port emits the same class names |
