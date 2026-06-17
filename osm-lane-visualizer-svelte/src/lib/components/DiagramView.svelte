<script lang="ts">
  let {
    html = '',
    country = 'vn',
    onhover,
    onzoom,
    ondrill
  }: {
    html: string;
    country?: string;
    onhover: (id: number) => void;
    onzoom: (id: number) => void;
    ondrill: (wayid: string) => void;
  } = $props();

  let scale = $state(0.7); // shrink so multi-lane roads fit without a wide scroll

  // hatch fill matching div.lane.nolane in style.css
  const HATCH =
    'repeating-linear-gradient(-45deg,#666,#666 4px,#fff 4px,#fff 9px,#666 9px,#666 13px)';

  function over(e: MouseEvent) {
    const label = (e.target as HTMLElement).closest('.label') as HTMLElement | null;
    if (label?.dataset.way) onhover(Number(label.dataset.way));
  }

  function click(e: MouseEvent) {
    const target = e.target as HTMLElement;
    // drill-in links ((V), navigation arrows) reload to a single way
    const drill = target.closest('[data-wayid]') as HTMLElement | null;
    if (drill?.dataset.wayid) {
      ondrill(drill.dataset.wayid);
      return;
    }
    // clicking a way row's label zooms the map to that way
    const label = target.closest('.label') as HTMLElement | null;
    if (label?.dataset.way) onzoom(Number(label.dataset.way));
  }
</script>

<!-- single root element: this component is one cell of the page's 3-col grid -->
<div class="diagram-pane">
  {#if html}
    <div class="diagram-tools">
      <label>Size <input type="range" min="0.4" max="1.2" step="0.05" bind:value={scale} /></label>
      <span>{Math.round(scale * 100)}%</span>
    </div>

    <details class="legend-disc">
      <summary class="btn">Legend</summary>
      <div class="legend">
        <div class="lg-group">
          <span class="lg-h">Lanes</span>
          <span class="lg"><i class="sw" style="background:#cdc"></i>Forward</span>
          <span class="lg"><i class="sw" style="background:#dcc"></i>Backward</span>
          <span class="lg"><i class="sw" style="background:#ccccc5"></i>Both ways</span>
          <span class="lg"><i class="sw" style={`background:${HATCH}`}></i>No lane data</span>
          <span class="lg"><i class="sw" style="background:#ccc"></i>Shoulder</span>
          <span class="lg"><i class="sw" style="background:#bde"></i>Sidewalk</span>
        </div>
        <div class="lg-group">
          <span class="lg-h">Signs</span>
          <span class="lg"><i class="circle">50</i>Speed limit</span>
          <span class="lg"><i class="badge">2</i>Intersections on segment</span>
          <span class="lg"><i class="arrow">↰↑↱</i>Turn lanes</span>
        </div>
        <div class="lg-group">
          <span class="lg-h">Route shield</span>
          {#if country === 'vn'}
            <span class="lg"><i class="shield" style="background:#168039;color:#fff">CT</i>Expressway</span>
            <span class="lg"><i class="shield" style="background:#1a5fb4;color:#fff">QL</i>National</span>
            <span class="lg"><i class="shield" style="background:#fff;color:#000;border:1px solid #000">TL</i>Provincial</span>
          {:else}
            <span class="lg"><i class="shield" style="background:#1a5fb4;color:#fff">A1</i>Coloured by road class</span>
          {/if}
        </div>
      </div>
    </details>
  {/if}

  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <!-- svelte-ignore a11y_mouse_events_have_key_events -->
  <div class="diagram-scroll">
    <div class="diagram" style={`zoom:${scale}`} onmouseover={over} onclick={click} role="presentation">
      {@html html}
    </div>
  </div>
</div>

<style>
  .diagram-tools {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 12px;
    color: #6b7280;
    margin-bottom: 6px;
  }
  .diagram-pane {
    min-width: 0; /* allow the grid cell to shrink instead of forcing the column wide */
  }

  .legend {
    display: flex;
    flex-wrap: wrap;
    gap: 6px 22px;
    padding: 10px 12px;
    margin-bottom: 8px;
    border: 1px solid #d9dee6;
    border-radius: 8px;
    background: #fff;
    font-family: sans-serif;
    font-size: 12px;
  }
  .lg-group {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 6px 12px;
  }
  .lg-h {
    font-weight: 700;
    color: #374151;
  }
  .lg {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    color: #4b5563;
  }
  .sw {
    width: 18px;
    height: 12px;
    border: 1px dashed #fff;
    outline: 1px solid #cbd2dc;
    flex: none;
  }
  .circle {
    width: 18px;
    height: 18px;
    border: 3px solid red;
    border-radius: 50%;
    background: #fff;
    color: #000;
    font-size: 9px;
    font-weight: 700;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    flex: none;
  }
  .badge {
    min-width: 16px;
    height: 16px;
    padding: 0 3px;
    border: 1px solid #000;
    border-radius: 9px;
    background: #f0e060;
    font-size: 10px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    flex: none;
  }
  .arrow {
    font-size: 13px;
    color: #333;
    letter-spacing: -1px;
  }
  .shield {
    min-width: 22px;
    height: 16px;
    padding: 0 3px;
    border-radius: 3px;
    font-size: 10px;
    font-weight: 700;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    flex: none;
  }
  /* Horizontal scroll only. overflow-y is set explicitly so the browser can't
     promote it to `auto` (which it does when the other axis is auto) — that
     promotion + zoom's sub-pixel rounding was spawning a phantom vertical
     scrollbar next to the page's. Height is auto, so nothing real is clipped. */
  .diagram-scroll {
    min-width: 0;
    overflow-x: auto;
    overflow-y: hidden;
    overscroll-behavior: contain;
  }
</style>
