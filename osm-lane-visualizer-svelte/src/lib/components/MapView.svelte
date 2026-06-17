<script lang="ts">
  import { onMount } from 'svelte';
  import type { WayGeom } from '$lib/osm/types';

  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  let L: any;
  let map: any;
  let marker: any;
  let polyline: any;
  let crossLayer: any;
  let mapEl: HTMLDivElement;

  onMount(async () => {
    L = (await import('leaflet')).default;
    await import('leaflet/dist/leaflet.css');
    map = L.map(mapEl).setView([16.0, 107.5], 5); // Vietnam
    L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: 'Map © <a href="https://www.openstreetmap.org">OpenStreetMap</a>'
    }).addTo(map);
    marker = L.marker(map.getCenter()).addTo(map);
    polyline = L.polyline([[0, 0]]).addTo(map);
    // the map sits in a CSS-grid column; recompute size once layout settles
    setTimeout(() => map.invalidateSize(), 0);
  });

  function draw(c: WayGeom | undefined, color: string, weight: number) {
    if (!c || !map) return;
    map.removeLayer(marker);
    marker = L.marker(c.begin).addTo(map);
    map.removeLayer(polyline);
    polyline = L.polyline(c.line, { color, weight }).addTo(map);
    if (crossLayer) map.removeLayer(crossLayer);
    crossLayer = L.layerGroup();
    for (const x of c.crossings ?? []) {
      if (x.lat == null || x.lon == null) continue;
      const m = `https://www.mapillary.com/app/?lat=${x.lat.toFixed(6)}&lng=${x.lon.toFixed(6)}&z=17`;
      L.circleMarker([x.lat, x.lon], {
        radius: 6,
        color: '#b45309',
        weight: 2,
        fillColor: '#f59e0b',
        fillOpacity: 1
      })
        .bindTooltip(x.label)
        .bindPopup(
          `<b>${x.label}</b><br><a href="${m}" target="_blank" rel="noopener">📷 Mapillary here</a>`
        )
        .addTo(crossLayer);
    }
    crossLayer.addTo(map);
    map.fitBounds(polyline.getBounds(), { padding: [30, 30] });
  }

  // imperative API used by the page (via bind:this)
  export function preview(c: WayGeom | undefined) {
    draw(c, '#44f', 4);
  }
  export function zoom(c: WayGeom | undefined) {
    draw(c, '#e6194b', 5);
  }
  export function resize() {
    map?.invalidateSize();
  }
</script>

<div class="mapwrap">
  <div class="map" bind:this={mapEl}></div>
  <p class="hint">Hover a way to preview · click a way row to zoom here.</p>
</div>

<style>
  .mapwrap {
    position: sticky;
    top: 8px;
  }
  .map {
    width: 100%;
    height: calc(100vh - 90px);
    min-height: 400px;
    border: 1px solid #ccc;
    border-radius: 6px;
  }
  .hint {
    font-size: 11px;
    color: #888;
    margin: 6px 2px 0;
  }
</style>
