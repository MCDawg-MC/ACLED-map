# ACLED-map
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <title>ACLED Conflict Map</title>
  <meta name="viewport" content="initial-scale=1,maximum-scale=1,user-scalable=no" />
  <script src="https://api.mapbox.com/mapbox-gl-js/v2.15.0/mapbox-gl.js"></script>
  <link href="https://api.mapbox.com/mapbox-gl-js/v2.15.0/mapbox-gl.css" rel="stylesheet" />
  <style>
    body { margin:0; padding:0; }
    #map { position:absolute; top:0; bottom:0; width:100%; }
    #filters {
      position: absolute;
      background: white;
      padding: 10px;
      z-index: 1;
      top: 10px;
      left: 10px;
      border-radius: 6px;
      font-family: sans-serif;
    }
  </style>
</head>
<body>
<div id="filters">
  <label for="eventType">Conflict Type:</label>
  <select id="eventType">
    <option value="">All</option>
    <option value="Battles">Battles</option>
    <option value="Protests">Protests</option>
    <option value="Riots">Riots</option>
    <option value="Strategic developments">Strategic developments</option>
    <option value="Violence against civilians">Violence against civilians</option>
    <option value="Explosions/Remote violence">Explosions/Remote violence</option>
  </select>
</div>
<div id="map"></div>

<script>
  mapboxgl.accessToken = 'YOUR_MAPBOX_ACCESS_TOKEN';
  const map = new mapboxgl.Map({
    container: 'map',
    style: 'mapbox://styles/mapbox/light-v10',
    center: [10, 20],
    zoom: 2
  });

  // Build date range for past 2 weeks
  const today = new Date();
  const start = new Date();
  start.setDate(today.getDate() - 14);

  const format = d => d.toISOString().split("T")[0];
  const startDate = format(start);
  const endDate = format(today);

  let rawEvents = [];

  // Fetch ACLED data
  async function fetchData(filter = '') {
    const url = `https://api.acleddata.com/acled/read?key=YOUR_ACLED_KEY&event_date=${startDate}|${endDate}&limit=5000&format=json`;
    const res = await fetch(url);
    const data = await res.json();
    rawEvents = data.data;

    const geojson = {
      type: 'FeatureCollection',
      features: rawEvents
        .filter(e => e.longitude && e.latitude && (!filter || e.event_type === filter))
        .map(e => ({
          type: 'Feature',
          geometry: {
            type: 'Point',
            coordinates: [parseFloat(e.longitude), parseFloat(e.latitude)]
          },
          properties: {
            event_type: e.event_type,
            location: e.location,
            country: e.country,
            notes: e.notes
          }
        }))
    };

    if (map.getSource('conflicts')) {
      map.getSource('conflicts').setData(geojson);
    } else {
      map.addSource('conflicts', {
        type: 'geojson',
        data: geojson,
        cluster: true,
        clusterMaxZoom: 6,
        clusterRadius: 40
      });

      map.addLayer({
        id: 'clusters',
        type: 'circle',
        source: 'conflicts',
        filter: ['has', 'point_count'],
        paint: {
          'circle-color': '#ff3300',
          'circle-radius': ['step', ['get', 'point_count'], 15, 100, 25, 750, 40],
          'circle-stroke-width': 1,
          'circle-stroke-color': '#fff'
        }
      });

      map.addLayer({
        id: 'cluster-count',
        type: 'symbol',
        source: 'conflicts',
        filter: ['has', 'point_count'],
        layout: {
          'text-field': '{point_count_abbreviated}',
          'text-font': ['DIN Offc Pro Medium', 'Arial Unicode MS Bold'],
          'text-size': 12
        }
      });

      map.addLayer({
        id: 'unclustered-point',
        type: 'circle',
        source: 'conflicts',
        filter: ['!', ['has', 'point_count']],
        paint: {
          'circle-color': '#0044ff',
          'circle-radius': 6,
          'circle-stroke-width': 1,
          'circle-stroke-color': '#fff'
        }
      });

      map.on('click', 'unclustered-point', e => {
        const { event_type, location, country, notes } = e.features[0].properties;
        new mapboxgl.Popup()
          .setLngLat(e.lngLat)
          .setHTML(`<strong>${event_type}</strong><br>${location}, ${country}<br><small>${notes}</small>`)
          .addTo(map);
      });

      map.on('mouseenter', 'unclustered-point', () => map.getCanvas().style.cursor = 'pointer');
      map.on('mouseleave', 'unclustered-point', () => map.getCanvas().style.cursor = '');
    }
  }

  map.on('load', () => {
    fetchData();

    document.getElementById('eventType').addEventListener('change', e => {
      fetchData(e.target.value);
    });
  });
</script>
</body>
</html>
