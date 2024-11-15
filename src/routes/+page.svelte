<script>
   import mapboxgl from 'mapbox-gl';
   import '../../node_modules/mapbox-gl/dist/mapbox-gl.css';
   import { onMount } from 'svelte';
 
   mapboxgl.accessToken = 'pk.eyJ1Ijoic25laGFsaXoiLCJhIjoiY20zajRpZ3g0MDd0ZTJucHg5d3I5NmR0MSJ9.EAuw3SMameVRcvTKWkznwQ';

  onMount(() => {
    const map = new mapboxgl.Map({
      container: 'map', // This should match the id of your map div
      style: 'mapbox://styles/mapbox/streets-v12',
      center: [-71.09416, 42.3601], // Longitude, Latitude of Boston
      zoom: 12,
      minZoom: 9,
      maxZoom: 15
    });
  });
 </script>

<svelte:head>
   <title>Home Page</title>
</svelte:head>

<h1>🚴🏼‍♀️CycleSpotting</h1>
<p>This website will spot bike traffic within Boston.</p>

<div id="map"></div>

<style>
   @import url('$lib/global.css');

   #map {
    width: 100%;
    height: 400px;
  }
</style>