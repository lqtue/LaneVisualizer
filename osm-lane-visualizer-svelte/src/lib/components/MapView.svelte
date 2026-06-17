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
  let geomOpacity = $state(0.85); // road geometry line opacity (slider)

  function applyOpacity() {
    polyline?.setStyle({ opacity: geomOpacity });
    marker?.setStyle?.({ opacity: geomOpacity, fillOpacity: geomOpacity });
  }

  onMount(async () => {
    L = (await import('leaflet')).default;
    await import('leaflet/dist/leaflet.css');
    map = L.map(mapEl).setView([16.0, 107.5], 5); // Vietnam
    const osm = L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: 'Map © <a href="https://www.openstreetmap.org">OpenStreetMap</a>'
    }).addTo(map);
    const sat = L.tileLayer(
      'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
      { attribution: 'Imagery © <a href="https://www.esri.com">Esri</a>', maxZoom: 19 }
    );
    L.control.layers({ OSM: osm, Satellite: sat }, {}, { position: 'topright' }).addTo(map);
    marker = beginMarker(map.getCenter());
    polyline = L.polyline([[0, 0]]).addTo(map);
    // the map sits in a CSS-grid column; recompute size once layout settles
    setTimeout(() => map.invalidateSize(), 0);
  });

  // circle marker for the begin point (avoids Leaflet's missing default icon PNG)
  function beginMarker(latlng: [number, number]) {
    return L.circleMarker(latlng, {
      radius: 6,
      color: '#1d4ed8',
      weight: 2,
      fillColor: '#3b82f6',
      fillOpacity: 1
    }).addTo(map);
  }

  function draw(c: WayGeom | undefined, color: string, weight: number) {
    if (!c || !map) return;
    map.removeLayer(marker);
    marker = beginMarker(c.begin);
    map.removeLayer(polyline);
    polyline = L.polyline(c.line, { color, weight, opacity: geomOpacity }).addTo(map);
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
  <div class="hint">
    <span>Hover a way to preview · click a way row to zoom here.</span>
    <label class="opacity">Road opacity {Math.round(geomOpacity * 100)}%
      <input type="range" min="0.1" max="1" step="0.05" bind:value={geomOpacity} oninput={applyOpacity} />
    </label>
  </div>
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
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 10px;
    font-size: 11px;
    color: #888;
    margin: 6px 2px 0;
  }
  .opacity {
    margin-left: auto;
    display: inline-flex;
    align-items: center;
    gap: 5px;
  }
</style>
