<script lang="ts">
  import { base } from '$app/paths';
  import { LaneVisualizer, wayQuery } from '$lib/osm/visualizer';
  import { fetchOfficialLengthKm } from '$lib/osm/wikidata';
  import SearchPanel from '$lib/components/SearchPanel.svelte';
  import DiagramView from '$lib/components/DiagramView.svelte';
  import MapView from '$lib/components/MapView.svelte';
  import StatsDashboard from '$lib/components/StatsDashboard.svelte';
  import ExportPanel from '$lib/components/ExportPanel.svelte';
  import '$lib/components/modules.css';
  import type {
    RenderResult,
    WayStat,
    RelationMeta,
    SearchRequest,
    Intersection,
    ExportRow
  } from '$lib/osm/types';

  // ---- form / page state ------------------------------------------------
  let country = $state('vn');

  // ---- render state -----------------------------------------------------
  let html = $state('');
  let totalStartPoints = $state(0);
  let loading = $state(false);
  let errorMsg = $state('');
  let wayCoords: RenderResult['wayCoords'] = {};

  // ---- dashboard state --------------------------------------------------
  let stats = $state<WayStat[]>([]);
  let rawLengthKm = $state(0);
  let officialKm = $state<number | null>(null);
  let meta = $state<RelationMeta | null>(null);
  let intersections = $state<Intersection[]>([]);
  let intersectionsRan = $state(false);
  let exportRows = $state<ExportRow[]>([]);

  // child component instance (imperative map API)
  let mapView: MapView;

  async function runSearch(req: SearchRequest) {
    loading = true;
    errorMsg = '';
    html = '';
    stats = [];
    rawLengthKm = 0;
    officialKm = null;
    meta = null;
    intersections = [];
    intersectionsRan = req.opts.intersections;
    exportRows = [];
    try {
      const v = new LaneVisualizer(req.opts);
      const r = await v.readData(req.query, 0);
      if (r !== 0) {
        errorMsg =
          r === -1 ? 'No elements returned by Overpass for this query.' : 'Need at least two ways.';
        return;
      }
      v.organizeWays();
      if (req.opts.adjacent) {
        try {
          await v.readData(v.buildAdjacentQuery(), 1);
        } catch {
          // adjacency is optional decoration; ignore failures
        }
      }
      if (req.opts.intersections) {
        try {
          await v.readData(v.buildCrossingQuery(), 2);
          v.computeIntersections();
        } catch {
          // intersection detection is optional; ignore failures
        }
      }
      const result = v.render(req.start);
      html = result.html;
      totalStartPoints = result.totalStartPoints;
      wayCoords = result.wayCoords;
      stats = result.stats;
      rawLengthKm = result.rawLengthKm;
      intersections = result.intersections;
      exportRows = result.exportRows;
      meta = v.getRelationMeta();
      if (!html.trim()) errorMsg = 'Query returned data but no highway ways could be drawn.';
      // official length from Wikidata (non-blocking; ignore failures)
      if (meta?.wikidata) {
        fetchOfficialLengthKm(meta.wikidata)
          .then((km) => (officialKm = km))
          .catch(() => {});
      }
    } catch (e) {
      errorMsg = (e as Error).message || String(e);
    } finally {
      loading = false;
    }
  }

  // diagram → map / drill-in wiring
  const onHover = (id: number) => mapView?.preview(wayCoords[id]);
  const onZoom = (id: number) => mapView?.zoom(wayCoords[id]);
  const onDrill = (wayid: string) =>
    runSearch({ query: wayQuery(wayid), start: 1, opts: lastOpts() });

  // keep the opts of the most recent search so drill-in reuses them
  let lastReq: SearchRequest | null = null;
  function lastOpts() {
    return (
      lastReq?.opts ?? {
        country,
        usePlacement: false,
        adjacent: false,
        lanewidth: false,
        usenodes: false,
        extendway: false,
        intersections: false
      }
    );
  }
  function handleSearch(req: SearchRequest) {
    lastReq = req;
    runSearch(req);
  }

  // ---- draggable split between diagram and map --------------------------
  let splitEl: HTMLDivElement;
  let leftFrac = $state(0.5); // share of width given to the diagram
  let dragging = $state(false);

  function dragStart(e: PointerEvent) {
    dragging = true;
    (e.target as HTMLElement).setPointerCapture(e.pointerId);
  }
  function dragMove(e: PointerEvent) {
    if (!dragging || !splitEl) return;
    const r = splitEl.getBoundingClientRect();
    let f = (e.clientX - r.left) / r.width;
    leftFrac = Math.min(0.85, Math.max(0.15, f));
    mapView?.resize(); // keep Leaflet tiles filling the changing column
  }
  function dragEnd(e: PointerEvent) {
    dragging = false;
    (e.target as HTMLElement).releasePointerCapture?.(e.pointerId);
    mapView?.resize();
  }
</script>

<svelte:head>
  <link rel="stylesheet" href={`${base}/css/style.css`} />
  <link rel="stylesheet" href={`${base}/css/${country}.css`} />
</svelte:head>

<h1>OSM Lane Visualizer <small>(Svelte port)</small></h1>

<!-- module 1: search & configuration -->
<SearchPanel bind:country {totalStartPoints} {loading} onsearch={handleSearch} />

{#if loading}<p class="status">Loading from Overpass…</p>{/if}
{#if errorMsg}<p class="status error">{errorMsg}</p>{/if}

<!-- module 2: completeness dashboard (export button lives in its header) -->
{#if stats.length}
  <StatsDashboard {stats} {rawLengthKm} {officialKm} {meta} {intersections} {intersectionsRan}>
    {#snippet actions()}
      <ExportPanel rows={exportRows} name={meta?.ref || meta?.name || 'osm-export'} />
    {/snippet}
  </StatsDashboard>
{/if}

<!-- module 3: split screen — diagram | (drag) | map -->
<div
  class="split"
  class:dragging
  bind:this={splitEl}
  style={`grid-template-columns:${leftFrac}fr 8px ${1 - leftFrac}fr`}
>
  <DiagramView {html} {country} onhover={onHover} onzoom={onZoom} ondrill={onDrill} />
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <div
    class="gutter"
    role="separator"
    aria-orientation="vertical"
    onpointerdown={dragStart}
    onpointermove={dragMove}
    onpointerup={dragEnd}
    title="Drag to resize"
  ></div>
  <MapView bind:this={mapView} />
</div>

<style>
  h1 small {
    font-size: 14px;
    color: #888;
  }
  .status {
    clear: both;
    font-weight: bold;
  }
  .status.error {
    color: #b00;
  }

  /* split: diagram | draggable gutter | full-height sticky map */
  .split {
    display: grid;
    align-items: start;
    clear: both;
  }
  .split.dragging {
    cursor: col-resize;
    user-select: none;
  }
  .gutter {
    align-self: stretch;
    min-height: 200px;
    cursor: col-resize;
    background: #e3e3e3;
    border-radius: 4px;
    position: sticky;
    top: 8px;
    height: calc(100vh - 90px);
  }
  .gutter:hover,
  .split.dragging .gutter {
    background: #9bb4d6;
  }

  @media (max-width: 900px) {
    .split {
      grid-template-columns: 1fr !important;
    }
    .gutter {
      display: none;
    }
  }
</style>
