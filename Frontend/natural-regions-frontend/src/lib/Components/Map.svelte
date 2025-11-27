<script>
  import { onMount } from 'svelte';
  import L from 'leaflet';

  let map;
  let geoJsonLayer;
  let currentParameter = '';
  let currentYear = '';

  // color palette (5 classes)
  const colors = ['#f7fbff','#c6dbef','#6baed6','#2171b5','#08306b'];

  // utility: compute quantile breaks for values array
  function computeQuantileBreaks(values, numBuckets = 5) {
    if (!values.length) return [];
    const sorted = Array.from(values).sort((a,b) => a - b);
    const breaks = [];
    for (let i = 1; i < numBuckets; i++) {
      const q = i / numBuckets;
      const idx = Math.floor(q * (sorted.length - 1));
      breaks.push(sorted[idx]);
    }
    return breaks;
  }

  // get color for a value based on breaks
  function getColor(value, breaks) {
    if (value === null || value === undefined) return '#999999'; // gray for no data
    for (let i = breaks.length - 1; i >= 0; i--) {
      if (value >= breaks[i]) return colors[i+1] || colors[colors.length-1];
    }
    return colors[0];
  }

  // style function for GeoJSON features
  let currentBreaks = [];

  function styleFeature(feature) {
    const val = feature.properties ? feature.properties.value : null;
    return {
      fillColor: getColor(val, currentBreaks),
      weight: 1,
      opacity: 1,
      color: 'white',
      dashArray: '0',
      fillOpacity: val === null || val === undefined ? 0.4 : 0.8
    };
  }

  // popup content
  function onEachFeature(feature, layer) {
    const props = feature.properties || {};
    const val = props.value === null || props.value === undefined ? 'No data' : props.value;
    const name = props.name || props.region_code || 'Unknown';
    layer.bindPopup(`<strong>${name}</strong><br/>${currentParameter}: ${val} (${props.year ?? ''})`);
    layer.on({
      mouseover: (e) => {
        e.target.setStyle({ weight: 2, color: '#333' });
      },
      mouseout: (e) => {
        geoJsonLayer.resetStyle(e.target);
      }
    });
  }

  // public method to load data (exposed to parent via bind:this)
  async function loadMapData(parameter, year) {
    currentParameter = parameter;
    currentYear = year;

    const encoded = encodeURIComponent(parameter);
    const url = `http://localhost:5000/api/map?parameter=${encoded}&year=${encodeURIComponent(year)}`;

    try {
      if (geoJsonLayer) {
        geoJsonLayer.clearLayers();
      }

      const resp = await fetch(url);
      if (!resp.ok) {
        const text = await resp.text();
        throw new Error(text || 'Failed to fetch map data');
      }
      const geojson = await resp.json();

      // extract numeric values for breaks
      const values = [];
      (geojson.features || []).forEach(f => {
        const v = f.properties ? f.properties.value : null;
        if (v !== null && v !== undefined && !isNaN(Number(v))) {
          values.push(Number(v));
        }
      });

      currentBreaks = computeQuantileBreaks(values, 4); // 4 breaks => 5 classes

      // create new geoJson layer with style & popup
      geoJsonLayer = L.geoJSON(geojson, {
        style: styleFeature,
        onEachFeature: onEachFeature
      }).addTo(map);

      // fit bounds
      try {
        const bounds = geoJsonLayer.getBounds();
        if (bounds.isValid()) map.fitBounds(bounds, { padding: [20,20] });
      } catch (err) {
        console.warn('Could not fit bounds', err);
      }

      // render legend
      renderLegend(currentBreaks);

    } catch (err) {
      alert('Error loading map: ' + err.message);
      console.error(err);
    }
  }

  // create / update legend
  let legendControl;
  function renderLegend(breaks) {
    if (legendControl) map.removeControl(legendControl);

    legendControl = L.control({ position: 'bottomright' });
    legendControl.onAdd = function () {
      const div = L.DomUtil.create('div', 'legend');
      if (!breaks || breaks.length === 0) {
        div.innerHTML = '<div>No data available</div>';
        return div;
      }
      // build legend HTML
      let html = '<div><strong>Legend</strong></div>';
      // compute intervals
      const intervals = [];
      let prev = -Infinity;
      for (let i = 0; i < breaks.length; i++) {
        intervals.push([prev, breaks[i]]);
        prev = breaks[i];
      }
      intervals.push([prev, Infinity]);

      for (let i = 0; i < intervals.length; i++) {
        const label = (intervals[i][0] === -Infinity ? `< ${intervals[i][1]}` :
                      (intervals[i][1] === Infinity ? `≥ ${intervals[i][0]}` : `${Math.round(intervals[i][0]*100)/100} - ${Math.round(intervals[i][1]*100)/100}`));
        html += `<div><span class="swatch" style="background:${colors[i]}"></span>${label}</div>`;
      }

      div.innerHTML = html;
      return div;
    };
    legendControl.addTo(map);
  }

  onMount(() => {
    map = L.map('map', { preferCanvas: true }).setView([22.5, 78.9], 5);
    // L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    //   maxZoom: 19,
    //   attribution: '&copy; OpenStreetMap contributors'
    // }).addTo(map);

    // initialize empty layer
    geoJsonLayer = L.geoJSON().addTo(map);

    // expose loadMapData via component instance (bind:this in parent)
    // done automatically by defining function in module scope
  });

  // export the loadMapData method so parent can call it
  export { loadMapData };
</script>

<div id="map"></div>
