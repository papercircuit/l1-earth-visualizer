<script lang="ts">
  import { onMount } from 'svelte';
  import * as d3 from 'd3';
  import { feature } from 'topojson-client';

  // X-band stations
  const xBandStations = [
    { name: "Thermopylae, GR", lat: 38.7965, lon: 22.5364, cone: 170, color: "#FF0000" },
    { name: "Nemea, GR", lat: 37.8103, lon: 22.7100, cone: 170, color: "#00FF00" },
    { name: "Canberra, AUS", lat: -35.3082, lon: 149.1244, cone: 170, color: "#0000FF" },
    { name: "Awarua, NZ", lat: -46.5, lon: 168.3, cone: 170, color: "#FFA500" },
    { name: "West Virginia, USA", lat: 39.5381, lon: -80.0000, cone: 170, color: "#FF00FF" },
    { name: "Virginia, USA", lat: 37.4316, lon: -76.5000, cone: 170, color: "#00FFFF" }
  ];

  let epicData: any = null;
  let satelliteData: any[] = [];
  let worldData: any = null;
  let hoveredStation: string | null = null;
  let selectedDate = new Date(Date.now());
  let loading = false;
  let error: string | null = null;

  // Projection setup
  const width = 800;
  const height = 800;
  let projection = d3.geoOrthographic()
    .scale(width / 2.5)
    .translate([width / 2, height / 2])
    .clipAngle(90);
  let pathGenerator = d3.geoPath().projection(projection);

  // Calculate time-adjusted Earth orientation
  const calculateEarthOrientation = (date: Date | string) => {
    // Ensure we're working with a Date object
    const dateObj = date instanceof Date ? date : new Date(date);
    const JD = (dateObj.getTime() / 86400000) + 2440587.5;
    const T = (JD - 2451545.0) / 36525;

    // Sun position calculations
    const L0 = (280.46646 + 36000.76983 * T) % 360;
    const M = (357.52911 + 35999.05029 * T) % 360;
    const C = 1.9148 * Math.sin(M * Math.PI/180) + 0.0200 * Math.sin(2 * M * Math.PI/180);
    const λ = (L0 + C) % 360;

    // Earth's axial tilt
    const ε = 23.43929111 - (46.8150 * T + 0.00059 * T**2) / 3600;

    // Calculate subsolar point (this is where DSCOVR looks at)
    const subsolarLon = (λ - 180 + (dateObj.getUTCHours() * 15)) % 360;
    const subsolarLat = Math.asin(Math.sin(ε * Math.PI/180) * Math.sin(λ * Math.PI/180)) * 180/Math.PI;

    projection.rotate([-subsolarLon, -subsolarLat, 0]);
    
    // Update EPIC data to always be at center (0, 0) after rotation
    if (epicData) {
      epicData.centroid_coordinates = {
        lon: subsolarLon,
        lat: subsolarLat
      };
    }
    
    pathGenerator = d3.geoPath().projection(projection);
  };

  // Fetch latest available EPIC data
  const fetchLatestEPICData = async () => {
    try {
      const apiKey = import.meta.env.VITE_NASA_API_KEY;
      const datesRes = await fetch(`https://api.nasa.gov/EPIC/api/natural/all?api_key=${apiKey}`);
      const dates = await datesRes.json();
      if (!dates || dates.length === 0) {
        throw new Error('No EPIC data available');
      }
      const latestDate = dates[0].date;
      
      const epicRes = await fetch(`https://api.nasa.gov/EPIC/api/natural/date/${latestDate}?api_key=${apiKey}`);
      const data = await epicRes.json();
      return data[data.length - 1]; // Latest image of the day
    } catch (err: any) {
      console.error('EPIC data error:', err);
      error = err.message;
      return null;
    }
  };

  // Add a helper function to format dates for the input
  const formatDateForInput = (date: Date) => {
    // Get local ISO string by adjusting for timezone offset
    const tzOffset = date.getTimezoneOffset() * 60000; // offset in milliseconds
    return new Date(date.getTime() - tzOffset).toISOString().slice(0, 16);
  };

  // Helper function to format coordinates
  const formatCoords = (lat: number, lon: number) => {
    return `${Math.abs(lat).toFixed(2)}°${lat >= 0 ? 'N' : 'S'}, ${Math.abs(lon).toFixed(2)}°${lon >= 0 ? 'E' : 'W'}`;
  };

  // Helper function to get local time at a location
  const getLocalTime = (lon: number) => {
    const date = new Date(selectedDate);
    const utcHours = date.getUTCHours() + (date.getUTCMinutes() / 60);
    const localHours = (utcHours + (lon / 15)) % 24;
    const hours = Math.floor(localHours);
    const minutes = Math.floor((localHours - hours) * 60);
    return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}`;
  };

  // Update the visibility check function
  const isPointVisible = (coords: [number, number], lon: number, lat: number): boolean => {
    // Get the current rotation of the projection
    const [lambda, phi] = projection.rotate();
    
    // Convert to radians
    const lonRad = lon * Math.PI / 180;
    const latRad = lat * Math.PI / 180;
    const lambdaRad = -lambda * Math.PI / 180;
    const phiRad = -phi * Math.PI / 180;
    
    // Calculate cosine of great circle distance
    const cosDistance = Math.sin(latRad) * Math.sin(phiRad) +
                       Math.cos(latRad) * Math.cos(phiRad) * Math.cos(lonRad - lambdaRad);
    
    // Point is visible if it's on the front hemisphere (cos > 0)
    return cosDistance > 0;
  };

  // Update visualization
  const updateVisualization = async () => {
    loading = true;
    error = null;
    
    try {
      // Get latest EPIC data (up to 2 days old)
      epicData = await fetchLatestEPICData();
      
      // Calculate current orientation
      calculateEarthOrientation(selectedDate);
      
      // Update world data
      if (!worldData) {
        const response = await fetch('/data/world-atlas.json');
        const topology = await response.json();
        worldData = feature(topology, topology.objects.countries);
      }
    } finally {
      loading = false;
    }
  };

  onMount(updateVisualization);
</script>

<div class="visualization-container">
  <div class="visualization-wrapper">
    {#if loading}
      <div class="loader">Loading Earth data...</div>
    {:else if error}
      <div class="error">{error}</div>
    {:else}
      <svg {width} {height} viewBox="0 0 {width} {height}">
        <!-- Earth sphere -->
        <circle cx={width/2} cy={height/2} r={width/2.5} fill="#1a3650" />

        <!-- Graticule -->
        <path d={pathGenerator(d3.geoGraticule()())} 
              stroke="rgba(255,255,255,0.3)" fill="none" />

        <!-- Land masses -->
        {#if worldData}
          <path d={pathGenerator(worldData)} 
                fill="#3a5f5e" 
                stroke="#4a7a79" 
                stroke-width="0.4" />
        {/if}

        <!-- EPIC centroid projection -->
        {#if epicData}
          <circle cx={width/2} cy={height/2}
                  r="8" fill="none" stroke="#FFD700" stroke-width="2">
            <title>EPIC View Center</title>
          </circle>
        {/if}

        <!-- Ground stations -->
        {#each xBandStations as station}
          {@const coords = projection([station.lon, station.lat])}
          {#if coords && isPointVisible(coords, station.lon, station.lat)}
            <g role="button"
               tabindex="0"
               transform={`translate(${coords})`}
               on:mouseenter={() => hoveredStation = station.name}
               on:mouseleave={() => hoveredStation = null}>
              <!-- Station marker -->
              <circle r="4" 
                      fill={station.color} 
                      stroke="white" 
                      stroke-width="1" />
              
              <!-- Coverage cone -->
              <path d={pathGenerator(d3.geoCircle()
                .center([station.lon, station.lat])
                .radius(station.cone/2)())}
                fill={station.color} 
                fill-opacity="0.15"
                stroke={station.color} 
                stroke-width="1" />
              
              <!-- Hover tooltip -->
              {#if hoveredStation === station.name}
                <g transform="translate(10, -10)">
                  <rect x="0" y="-35" 
                        width="200" height="35" 
                        fill="rgba(0,0,0,0.8)" 
                        rx="4" />
                  <text x="5" y="-20" fill="white" font-size="12">
                    {station.name}
                  </text>
                  <text x="5" y="-5" fill="white" font-size="12">
                    {formatCoords(station.lat, station.lon)}
                    Local: {getLocalTime(station.lon)}
                  </text>
                </g>
              {/if}
            </g>
          {/if}
        {/each}
      </svg>
    {/if}
  </div>

  <!-- Controls -->
  <div class="controls">
    <input type="datetime-local" 
           value={formatDateForInput(selectedDate)}
           max={formatDateForInput(new Date())}
           on:change={(e: Event) => {
             const target = e.target as HTMLInputElement;
             selectedDate = new Date(target.value);
             updateVisualization();
           }}>
    
    {#if epicData}
      <div class="epic-info">
        Using EPIC data from: {new Date(epicData.date).toLocaleDateString()}<br>
        Extrapolated position for: {selectedDate.toLocaleString()}
      </div>
    {/if}
  </div>
</div>

<style>
  .visualization-container {
    position: relative;
    max-width: 800px;
    margin: 0 auto;
    background: #000;
    border-radius: 8px;
    overflow: hidden;
  }

  .loader, .error {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    color: white;
    padding: 1rem;
    background: rgba(0,0,0,0.7);
    border-radius: 4px;
  }

  .controls {
    position: absolute;
    bottom: 20px;
    left: 20px;
    background: rgba(0, 0, 0, 0.7);
    padding: 1rem;
    border-radius: 8px;
    color: white;

    input {
      background: #333;
      border: 1px solid #555;
      color: white;
      padding: 0.5rem;
      border-radius: 4px;
    }
  }

  .epic-info {
    margin-top: 1rem;
    font-size: 0.9rem;
    opacity: 0.9;
  }

  path {
    transition: all 0.3s ease-out;
  }
</style>