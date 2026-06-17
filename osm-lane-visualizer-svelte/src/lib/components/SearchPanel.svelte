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

  // --- search by: one selector + one input, instead of four rows ----------
  const SEARCH_BY = [
    { key: 'relref', label: 'Relation by ref', placeholder: 'QL.51', build: relRefQuery },
    { key: 'relname', label: 'Relation by name', placeholder: 'Quốc lộ 51', build: relNameQuery },
    { key: 'relid', label: 'Relation by id', placeholder: '123456', build: relQuery },
    { key: 'wayid', label: 'Way by id', placeholder: '123456', build: wayQuery }
  ] as const;

  let searchBy = $state<(typeof SEARCH_BY)[number]['key']>('relname');
  let term = $state('Quốc lộ 51');
  const current = $derived(SEARCH_BY.find((s) => s.key === searchBy)!);

  // --- configuration toggles + their explanations -------------------------
  const CONFIG = [
    { key: 'usePlacement', label: 'Use placement', help: 'Use the placement tag to position lanes more naturally on the carriageway.' },
    { key: 'adjacent', label: 'Use adjacent ways', help: "Also draw ways that join at each segment's end node." },
    { key: 'lanewidth', label: 'Use lane width', help: 'Read lane width from the width tag instead of a fixed size.' },
    { key: 'usenodes', label: 'Use tags on nodes', help: 'Show node tags (signals, stop, crossing…) along the way.' },
    { key: 'extendway', label: 'Include ways before & after', help: 'Add arrows to step to the way before and after this one.' },
    { key: 'intersections', label: 'Detect intersections', help: 'Find roads crossing this one (one extra Overpass query).' }
  ] as const;

  let cfg = $state<Record<string, boolean>>({
    usePlacement: false,
    adjacent: false,
    lanewidth: false,
    usenodes: false,
    extendway: false,
    intersections: false
  });
  let start = $state(1);

  function go() {
    if (loading) return;
    const opts: Options = { country, ...cfg } as Options;
    onsearch({ query: current.build(term.trim()), start: Number(start) || 1, opts });
  }
</script>

<section class="module">
  <header>
    <h2>Search</h2>
    <label>Country
      <select bind:value={country}>
        <option value="vn">vn</option>
        <option value="de">de</option>
        <option value="be">be</option>
      </select>
    </label>
  </header>

  <div class="row search-row">
    <label>Search by
      <select bind:value={searchBy}>
        {#each SEARCH_BY as s (s.key)}<option value={s.key}>{s.label}</option>{/each}
      </select>
    </label>
    <input
      type="text"
      bind:value={term}
      placeholder={current.placeholder}
      onkeydown={(e) => e.key === 'Enter' && go()}
      style="flex:1;min-width:160px"
    />
    <button class="btn btn-primary" disabled={loading} onclick={go}>
      {loading ? 'Searching…' : 'Search'}
    </button>
  </div>

  <details class="config">
    <summary class="btn">Configuration</summary>
    <div class="config-grid">
      {#each CONFIG as c (c.key)}
        <div class="opt">
          <label title={c.help}><input type="checkbox" bind:checked={cfg[c.key]} /> {c.label}</label>
        </div>
      {/each}
      <div class="opt">
        <label title={`Which chain end to begin drawing from (${totalStartPoints} found).`}>Start at end number <input type="text" bind:value={start} style="width:42px" /></label>
      </div>
    </div>
  </details>
</section>

<style>
  .search-row {
    margin-bottom: 8px;
  }
  .config-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 6px 16px;
    margin-top: 10px;
  }
  .opt label {
    display: inline-flex;
    align-items: center;
    gap: 5px;
  }
</style>
