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
  let layersControl: any;
  let baseLayers: any[] = []; // built-in bases, so a custom layer can replace them
  let customLayer: any;
  let geomOpacity = $state(0.85); // road geometry line opacity (slider)
  let customUrl = $state('');

  // overzoom past a source's native max instead of showing blank/grey tiles
  const TILE_OPTS = { maxNativeZoom: 19, maxZoom: 22 };

  function applyOpacity() {
    polyline?.setStyle({ opacity: geomOpacity });
    marker?.setStyle?.({ opacity: geomOpacity, fillOpacity: geomOpacity });
  }

  // add (or replace) a custom XYZ tile source and switch to it. {z}/{x}/{y} template.
  function addCustom() {
    const url = customUrl.trim();
    // need all three placeholders, in any order (e.g. Google: ?x={x}&y={y}&z={z})
    if (!map || !['{z}', '{x}', '{y}'].every((t) => url.includes(t))) return;
    if (customLayer) {
      map.removeLayer(customLayer);
      layersControl.removeLayer(customLayer);
    }
    customLayer = L.tileLayer(url, { attribution: 'Custom XYZ', ...TILE_OPTS });
    layersControl.addBaseLayer(customLayer, 'Custom');
    for (const l of baseLayers) map.removeLayer(l);
    customLayer.addTo(map);
  }

  onMount(async () => {
    L = (await import('leaflet')).default;
    await import('leaflet/dist/leaflet.css');
    map = L.map(mapEl, { maxZoom: 22 }).setView([16.0, 107.5], 5); // Vietnam
    const osm = L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: 'Map © <a href="https://www.openstreetmap.org">OpenStreetMap</a>',
      ...TILE_OPTS
    }).addTo(map);
    const sat = L.tileLayer(
      'https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
      { attribution: 'Imagery © <a href="https://www.esri.com">Esri</a>', ...TILE_OPTS }
    );
    baseLayers = [osm, sat];
    layersControl = L.control.layers({ OSM: osm, Satellite: sat }, {}, { position: 'topright' }).addTo(map);
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
  <details class="map-opts">
    <summary class="btn">Map options</summary>
    <div class="hint">
      <label class="opacity">Road opacity {Math.round(geomOpacity * 100)}%
        <input type="range" min="0.1" max="1" step="0.05" bind:value={geomOpacity} oninput={applyOpacity} />
      </label>
      <label class="custom">XYZ
        <input
          type="text"
          placeholder={'https://…/{z}/{x}/{y}.png'}
          bind:value={customUrl}
          onkeydown={(e) => e.key === 'Enter' && addCustom()}
        />
      </label>
      <button class="btn" onclick={addCustom}>Add</button>
    </div>
  </details>
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
  .map-opts {
    margin: 6px 2px 0;
  }
  .hint {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 10px;
    font-size: 11px;
    color: #888;
    margin: 8px 0 0;
  }
  .opacity,
  .custom {
    display: inline-flex;
    align-items: center;
    gap: 5px;
  }
  .custom {
    flex: 1;
    min-width: 140px;
  }
  .custom input {
    flex: 1;
    min-width: 0;
    font: inherit;
    font-size: 11px;
    padding: 3px 6px;
    border: 1px solid #cbd2dc;
    border-radius: 4px;
  }
  .hint .btn {
    font-size: 11px;
    padding: 3px 10px;
  }
</style>
