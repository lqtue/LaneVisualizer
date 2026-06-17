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

<!-- svelte-ignore a11y_no_static_element_interactions -->
<!-- svelte-ignore a11y_mouse_events_have_key_events -->
<div class="diagram" onmouseover={over} onclick={click} role="presentation">
  {@html html}
</div>

<style>
  .diagram {
    min-width: 0;
    overflow-x: auto;
  }
</style>
