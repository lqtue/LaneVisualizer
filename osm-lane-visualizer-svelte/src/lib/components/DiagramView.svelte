<script lang="ts">
  let {
    html = '',
    onhover,
    onzoom,
    ondrill
  }: {
    html: string;
    onhover: (id: number) => void;
    onzoom: (id: number) => void;
    ondrill: (wayid: string) => void;
  } = $props();

  let scale = $state(0.7); // shrink so multi-lane roads fit without a wide scroll

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
