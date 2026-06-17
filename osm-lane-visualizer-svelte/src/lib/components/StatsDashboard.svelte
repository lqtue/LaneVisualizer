<script lang="ts">
  import type { WayStat, RelationMeta, Criterion, Intersection } from '$lib/osm/types';

  let {
    stats = [],
    rawLengthKm = 0,
    officialKm = null,
    meta = null,
    intersections = []
  }: {
    stats: WayStat[];
    rawLengthKm: number;
    officialKm: number | null;
    meta: RelationMeta | null;
    intersections: Intersection[];
  } = $props();

  const MAJOR = new Set([
    'motorway', 'motorway_link', 'trunk', 'trunk_link',
    'primary', 'primary_link', 'secondary', 'secondary_link', 'tertiary', 'tertiary_link'
  ]);
  const majorCount = $derived(
    intersections.filter((x) => x.roads.some((r) => MAJOR.has(r.highway))).length
  );

  // labels + order for the coverage criteria
  // meaningful completeness criteria (name/ref/oneway dropped — not survey gaps)
  const CRITERIA: { key: Criterion; label: string }[] = [
    { key: 'maxspeed', label: 'Maxspeed' },
    { key: 'lanes', label: 'Lane count' },
    { key: 'turn', label: 'Turn lanes' },
    { key: 'surface', label: 'Surface' },
    { key: 'lit', label: 'Lit' },
    { key: 'width', label: 'Width' },
    { key: 'shoulder', label: 'Shoulder' },
    { key: 'sidewalk', label: 'Sidewalk' },
    { key: 'structure', label: 'Bridge/Tunnel' },
    { key: 'access', label: 'Access' }
  ];

  const totalM = $derived(stats.reduce((s, w) => s + w.length, 0));

  // decoupled estimate: non-oneway counted fully, oneway halved (dual-carriageway heuristic)
  const decoupledKm = $derived(
    stats.reduce((s, w) => s + (w.oneway ? w.length / 2 : w.length), 0) / 1000
  );

  // length-weighted coverage per criterion
  const coverage = $derived(
    CRITERIA.map(({ key, label }) => {
      const covered = stats.reduce((s, w) => s + (w.present[key] ? w.length : 0), 0);
      const missingM = totalM - covered;
      const missingWays = stats.filter((w) => !w.present[key]).length;
      return {
        key,
        label,
        pct: totalM > 0 ? (covered / totalM) * 100 : 0,
        missingKm: missingM / 1000,
        missingWays
      };
    })
  );

  // distribution of numeric maxspeeds (km of road at each value)
  const speedDist = $derived(() => {
    const m = new Map<number, number>();
    for (const w of stats) if (w.maxspeed != null) m.set(w.maxspeed, (m.get(w.maxspeed) || 0) + w.length);
    return [...m.entries()].sort((a, b) => a[0] - b[0]).map(([v, len]) => ({ v, km: len / 1000 }));
  });

  function barColor(pct: number): string {
    if (pct >= 90) return '#2e7d32';
    if (pct >= 60) return '#f9a825';
    return '#c62828';
  }

  const osmVsOfficial = $derived(
    officialKm != null && officialKm > 0 ? (decoupledKm / officialKm) * 100 : null
  );
</script>

<div class="dash">
  <div class="dash-head">
    <h2>{meta?.ref || meta?.name || 'Road'} — data completeness</h2>
    <div class="lengths">
      <div class="len"><span class="num">{rawLengthKm.toFixed(1)}</span><span class="unit">km</span><span class="cap">OSM raw (all ways)</span></div>
      <div class="len"><span class="num">~{decoupledKm.toFixed(1)}</span><span class="unit">km</span><span class="cap">decoupled est.<sup>*</sup></span></div>
      <div class="len">
        <span class="num">{officialKm != null ? officialKm.toFixed(1) : '—'}</span><span class="unit">km</span>
        <span class="cap">official {meta?.wikidata ? `(${meta.wikidata})` : '(Wikidata)'}</span>
      </div>
      {#if osmVsOfficial != null}
        <div class="len"><span class="num">{osmVsOfficial.toFixed(0)}%</span><span class="cap">decoupled / official</span></div>
      {/if}
      <div class="len" title="nodes shared with a crossing highway">
        <span class="num">{intersections.length}</span><span class="cap">intersections ({majorCount} major)</span>
      </div>
    </div>
    <p class="foot"><sup>*</sup> oneway ways counted as half (assumed dual carriageway); not geometry-paired.</p>
  </div>

  <div class="cards">
    {#each coverage as c (c.key)}
      <div class="card" title={`${c.missingWays} ways · ${c.missingKm.toFixed(1)} km missing`}>
        <div class="card-top"><span class="clabel">{c.label}</span><span class="pct">{c.pct.toFixed(0)}%</span></div>
        <div class="track"><div class="fill" style={`width:${c.pct}%;background:${barColor(c.pct)}`}></div></div>
        <div class="sub">missing {c.missingKm.toFixed(1)} km · {c.missingWays} ways</div>
      </div>
    {/each}
  </div>

  {#if speedDist().length}
    <div class="speeds">
      <span class="clabel">Maxspeed values:</span>
      {#each speedDist() as s (s.v)}
        <span class="chip">{Math.round(s.v)} <small>· {s.km.toFixed(1)}km</small></span>
      {/each}
    </div>
  {/if}
</div>

<style>
  .dash {
    clear: both;
    border: 1px solid #ccc;
    border-radius: 6px;
    padding: 12px 14px;
    margin: 8px 0 16px;
    background: #f3f6fb;
    font-family: sans-serif;
  }
  .dash-head h2 {
    margin: 0 0 8px;
    font-size: 16px;
  }
  .lengths {
    display: flex;
    flex-wrap: wrap;
    gap: 18px;
    align-items: flex-end;
  }
  .len {
    display: flex;
    flex-direction: column;
    line-height: 1.1;
  }
  .len .num {
    font-size: 22px;
    font-weight: 700;
  }
  .len .unit {
    font-size: 11px;
    color: #666;
  }
  .len .cap {
    font-size: 11px;
    color: #888;
  }
  .foot {
    font-size: 10px;
    color: #999;
    margin: 6px 0 0;
  }
  .cards {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    gap: 8px;
    margin-top: 12px;
  }
  .card {
    border: 1px solid #e0e0e0;
    border-radius: 4px;
    padding: 6px 8px;
    background: #fff;
  }
  .card-top {
    display: flex;
    justify-content: space-between;
    align-items: baseline;
  }
  .clabel {
    font-size: 12px;
    color: #444;
  }
  .pct {
    font-size: 16px;
    font-weight: 700;
  }
  .track {
    height: 6px;
    border-radius: 3px;
    background: #eee;
    overflow: hidden;
    margin: 4px 0 3px;
  }
  .fill {
    height: 100%;
  }
  .sub {
    font-size: 10px;
    color: #999;
  }
  .speeds {
    margin-top: 12px;
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    align-items: center;
  }
  .chip {
    background: #eef;
    border: 1px solid #ccd;
    border-radius: 10px;
    padding: 1px 8px;
    font-size: 12px;
  }
  .chip small {
    color: #779;
  }
</style>
