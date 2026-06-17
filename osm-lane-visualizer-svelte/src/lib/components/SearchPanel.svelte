<script lang="ts">
  import { wayQuery, relQuery, relNameQuery, relRefQuery } from '$lib/osm/visualizer';
  import type { Options, SearchRequest } from '$lib/osm/types';

  let {
    country = $bindable('vn'),
    totalStartPoints = 0,
    loading = false,
    onsearch
  }: {
    country: string;
    totalStartPoints: number;
    loading: boolean;
    onsearch: (r: SearchRequest) => void;
  } = $props();

  // configuration toggles
  let usePlacement = $state(false);
  let adjacent = $state(false);
  let lanewidth = $state(false);
  let usenodes = $state(false);
  let extendway = $state(false);
  let intersections = $state(true);
  let start = $state(1);

  // search inputs
  let relref = $state('QL.51');
  let relname = $state('');
  let relid = $state('');
  let wayid = $state('');

  function opts(): Options {
    return { country, usePlacement, adjacent, lanewidth, usenodes, extendway, intersections };
  }
  function go(q: string) {
    onsearch({ query: q, start: Number(start) || 1, opts: opts() });
  }
</script>

<section class="search-panel">
  <h2 class="panel-title">Search</h2>
  <p class="lead">
    Look up a way or relation. Lane attributes and a completeness dashboard render below;
    hover or click a way to drive the map.
  </p>

  <div class="config">
    <h3>Configuration</h3>
    <label title="Evaluate the placement tag for a more natural lane arrangement">
      <input type="checkbox" bind:checked={usePlacement} /> Use placement</label
    >
    <br /><label title="Show geometry of ways joining at each segment's end node">
      <input type="checkbox" bind:checked={adjacent} /> Use adjacent ways</label
    >
    <br /><label title="Determine lane width from the width tag">
      <input type="checkbox" bind:checked={lanewidth} /> Use lane width</label
    >
    <br /><label title="Use tags on nodes (signals, stop, crossing, …)">
      <input type="checkbox" bind:checked={usenodes} /> Use tags on nodes</label
    >
    <br /><label title="Show navigation arrows to step way-by-way">
      <input type="checkbox" bind:checked={extendway} /> Include ways before &amp; after</label
    >
    <br /><label title="Find roads crossing this one (one extra Overpass query)">
      <input type="checkbox" bind:checked={intersections} /> Detect intersections</label
    >
    <br /><label>Country
      <select bind:value={country}>
        <option value="vn">vn</option>
        <option value="de">de</option>
        <option value="be">be</option>
      </select></label
    >
    <br /><label>Start at end number
      <input type="text" bind:value={start} style="width:30px" /></label
    >
    <span>(found {totalStartPoints} end nodes)</span>
  </div>

  <div class="selectquery">
    <h3>Search for:</h3>
    <p>
      <label>A relation with ref = <input type="text" bind:value={relref} /></label>
      <input type="button" value=" Go " disabled={loading} onclick={() => go(relRefQuery(relref))} />
      <br /><label>A relation with name = <input type="text" bind:value={relname} /></label>
      <input type="button" value=" Go " disabled={loading} onclick={() => go(relNameQuery(relname))} />
      <br /><label>A relation with id = <input type="text" bind:value={relid} /></label>
      <input type="button" value=" Go " disabled={loading} onclick={() => go(relQuery(relid))} />
      <br /><label>A way with id = <input type="text" bind:value={wayid} /></label>
      <input type="button" value=" Go " disabled={loading} onclick={() => go(wayQuery(wayid))} />
    </p>
  </div>
</section>

<style>
  .search-panel {
    display: flow-root; /* contains the floated .config / .selectquery */
    border: 1px solid #ccc;
    border-radius: 6px;
    background: #fff;
    padding: 10px 14px 14px;
    margin: 8px 0 16px;
  }
  .panel-title {
    margin: 0 0 4px;
    font-size: 16px;
  }
  .lead {
    margin: 0 0 8px;
    font-size: 12px;
    color: #666;
  }
</style>
