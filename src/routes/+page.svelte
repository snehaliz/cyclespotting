<script>
  import mapboxgl from 'mapbox-gl';
  import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
 
  mapboxgl.accessToken = 'pk.eyJ1Ijoic25laGFsaXoiLCJhIjoiY20zajRpZ3g0MDd0ZTJucHg5d3I5NmR0MSJ9.EAuw3SMameVRcvTKWkznwQ';

  let map;
  let stations = [];
  let mapViewChanged = 0;
  let arrivals, departures;
  let timeFilter = -1;
  let trips = [];
  let stationFlow = d3.scaleQuantize()
  .domain([0, 1])
  .range([0, 0.5, 1]);

  function getCoords(station) {
    let point = new mapboxgl.LngLat(+station.Long, +station.Lat);
    let {x, y} = map.project(point);
    return {cx: x, cy: y};
  }

  function minutesSinceMidnight(date) {
    return date.getHours() * 60 + date.getMinutes();
  }

  $: map?.on("move", () => mapViewChanged++);

  $: timeFilterLabel = new Date(0, 0, 0, 0, timeFilter)
                     .toLocaleString("en", {timeStyle: "short"});

  $: filteredTrips = timeFilter === -1 ? trips : 
  trips.filter(trip => {
    let startedMinutes = minutesSinceMidnight(trip.started_at);
    let endedMinutes = minutesSinceMidnight(trip.ended_at);
    return Math.abs(startedMinutes - timeFilter) <= 60
           || Math.abs(endedMinutes - timeFilter) <= 60;
  });

  $: filteredArrivals = d3.rollup(filteredTrips, v => v.length, d => d.end_station_id);
  $: filteredDepartures = d3.rollup(filteredTrips, v => v.length, d => d.start_station_id);

  $: filteredStations = stations.map(station => {
    let newStation = {...station};
    let id = newStation.Number;
    newStation.arrivals = filteredArrivals.get(id) ?? 0;
    newStation.departures = filteredDepartures.get(id) ?? 0;
    newStation.totalTraffic = newStation.arrivals + newStation.departures;
    return newStation;
  });
  
  $: radiusScale = d3.scaleSqrt()
  .domain([0, d3.max(stations, d => d.totalTraffic)])
  .range(timeFilter === -1 ? [0, 25] : [3, 50]);

  onMount(async () =>{
    map = new mapboxgl.Map({
    container: 'map', 
    style: 'mapbox://styles/mapbox/streets-v12',
    center: [-71.09416, 42.3601], 
    zoom: 12,
    minZoom: 9,
    maxZoom: 15
    });
  
  await new Promise(resolve => map.on("load", resolve));

  stations = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-stations.csv");
  
  trips = await d3.csv("https://vis-society.github.io/labs/8/data/bluebikes-traffic-2024-03.csv", d3.autoType)
  .then(trips => {
        for (let trip of trips) {
          trip.started_at = new Date(trip.started_at);
          trip.ended_at = new Date(trip.ended_at);
        }
        return trips;
      });

  departures = d3.rollup(trips, v => v.length, d => d.start_station_id);
  arrivals = d3.rollup(trips, v => v.length, d => d.end_station_id);

  stations = stations.map(station => {
    let id = station.Number;
    station.arrivals = arrivals.get(id) ?? 0;
    station.departures = departures.get(id) ?? 0;
    station.totalTraffic = station.arrivals + station.departures;
    return station;
  });

  const bikeLayerStyle = {
      "line-color": "#00FF00",
      "line-width": 3,
      "line-opacity": 0.6
    };

  map.addSource("boston_route", {
	    type: "geojson",
	    data: "https://bostonopendata-boston.opendata.arcgis.com/datasets/boston::existing-bike-network-2022.geojson?outSR=%7B%22latestWkid%22%3A3857%2C%22wkid%22%3A102100%7D",
    });

  map.addLayer({
      id: "boston-bike-lanes",
      type: "line",
      source: "boston_route",
      paint: bikeLayerStyle
    });

  map.addSource("cambridge_route", {
      type: "geojson",
      data: "https://raw.githubusercontent.com/cambridgegis/cambridgegis_data/main/Recreation/Bike_Facilities/RECREATION_BikeFacilities.geojson"
    });

  map.addLayer({
      id: "cambridge-bike-lanes",
      type: "line",
      source: "cambridge_route",
      paint: bikeLayerStyle
    });
  });
 </script>

<svelte:head>
   <title>Home Page</title>
</svelte:head>

<header>
<h1>🚴🏼‍♀️CycleSpotting</h1>
<p>This website will spot bike traffic.</p>
<label>
  Filter by time:
  <input type="range" min="-1" max="1440" step="1" bind:value={timeFilter}>
  <span>
    {#if timeFilter === -1}
      <em>(any time)</em>
    {:else}
      <time>{timeFilterLabel}</time>
    {/if}
  </span>
</label>
</header>

<div id="map">
  <svg>
    {#key mapViewChanged}
      {#each filteredStations as station}
        <circle 
          {...getCoords(station)} 
          r={radiusScale(station.totalTraffic)} 
          style="--departure-ratio: {stationFlow(station.departures / station.totalTraffic)}"
          stroke="white"
          stroke-width="1" 
        />
      {/each}
    {/key}
  </svg>
</div>

<div class="legend">
  <div style="--departure-ratio: 1">More departures</div>
  <div style="--departure-ratio: 0.5">Balanced</div>
  <div style="--departure-ratio: 0">More arrivals</div>
</div>

<style>
    @import url('$lib/global.css');

    header {
      display: flex;
      gap: 1em;
      align-items: baseline;
    }

    label {
      margin-left: auto;
    }

    label span {
      display: block;
    }

    em {
      color: #888;
      font-style: italic;
    }

    #map {
      width: 100%;
      height: 400px;
      position: relative;
    }
    #map svg {
      position: absolute;
      width: 100%;
      height: 100%;
      z-index: 1;
      pointer-events: none;
    }
    #map svg circle, 
    .legend > div{
      --color-departures: steelblue;
      --color-arrivals: darkorange;
      --color: color-mix(
        in oklch,
        var(--color-departures) calc(100% * var(--departure-ratio)),
        var(--color-arrivals)
      );
    fill: var(--color);
    fill-opacity: 0.6;
   }

   .legend {
    display: flex;
    justify-content: space-between;
    margin-block: 1em;
  }

  .legend > div {
    flex: 1;
    padding: 0.5em 1em;
    text-align: center;
    background-color: var(--color);
    color: white;
    font-weight: bold;
  }

  .legend > div:first-child {
    text-align: left;
  }

  .legend > div:last-child {
    text-align: right;
  }
</style>