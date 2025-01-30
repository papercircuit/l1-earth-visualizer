<script lang="ts">
  import { onMount } from 'svelte';
  import type { GeoProjection, GeoPath } from 'd3-geo';
  import * as d3 from 'd3';
  import { feature } from 'topojson-client';

  // X-band stations
  let xBandStations = [
    { name: "Thermopylae, GR", lat: 38.7965, lon: 22.5364, cone: 170, color: "#FF0000", canSeeDscovr: false, canSeeAce: false },
    { name: "Nemea, GR", lat: 37.8103, lon: 22.7100, cone: 170, color: "#00FF00", canSeeDscovr: false, canSeeAce: false },
    { name: "Canberra, AUS", lat: -35.3082, lon: 149.1244, cone: 170, color: "#0000FF", canSeeDscovr: false, canSeeAce: false },
    { name: "Awarua, NZ", lat: -46.5, lon: 168.3, cone: 170, color: "#FFA500", canSeeDscovr: false, canSeeAce: false },
    { name: "West Virginia, USA", lat: 39.5381, lon: -80.0000, cone: 170, color: "#FF00FF", canSeeDscovr: false, canSeeAce: false },
    { name: "Virginia, USA", lat: 37.4316, lon: -76.5000, cone: 170, color: "#00FFFF", canSeeDscovr: false, canSeeAce: false }
  ];

  let epicData: any = null;
  let satelliteData: Document | null = null;
  let worldData: any = null;
  let hoveredStation: string | null = null;
  let selectedDate = new Date(Date.now());
  let loading = false;
  let error: string | null = null;

  // Initialize with default values
  let width = 300;
  let height = 300;
  let container: HTMLDivElement;

  let projection: GeoProjection = d3.geoOrthographic()
    .scale(width / 2.5)
    .translate([width / 2, height / 2])
    .clipAngle(90);
  let pathGenerator: GeoPath = d3.geoPath().projection(projection);

  // Add graticule generator
  let graticule = d3.geoGraticule()();

  // Add a flag to track if we're rotating (vs resizing)
  let isRotating = false;

  // Update projection when size changes
  $: if (width && height && width > 0 && height > 0) {
    const size = Math.min(width, height);
    projection = d3.geoOrthographic()
      .scale(size / 2.5)
      .translate([width / 2, height / 2])
      .clipAngle(90);
    pathGenerator = d3.geoPath().projection(projection);
    // Don't regenerate graticule, just let it transition with the projection
    // Recalculate earth orientation with new projection
    if (selectedDate) {
      calculateEarthOrientation(selectedDate);
    }
  }

  // Add transition duration constant
  const TRANSITION_DURATION = 1000; // 1 second transition

  // Add a reactive statement to update station visibility when date changes
  $: if (selectedDate && epicData) {
    updateStationVisibility();
    xBandStations = [...xBandStations]; // Force reactivity update
  }

  const formatTime = (d: Date) => {
        return d.toISOString()
          .replace(/[-:]/g, '')  // Remove dashes and colons
          .replace(/\.\d+/, ''); // Remove milliseconds
      };

  // Add debounce helper
  const debounce = (fn: Function, ms: number) => {
    let timeoutId: ReturnType<typeof setTimeout>;
    return function (...args: any[]) {
      clearTimeout(timeoutId);
      timeoutId = setTimeout(() => fn.apply(null, args), ms);
    };
  };

  // Modify calculateEarthOrientation to always update station visibility
  const calculateEarthOrientation = (date: Date | string, animate = false) => {
    isRotating = animate;
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

    if (animate) {
      // Get current rotation
      const [oldLon, oldLat] = projection.rotate();
      
      // Animate to new rotation
      d3.select(container)
        .select('svg')
        .transition()
        .duration(TRANSITION_DURATION)
        .tween('rotate', () => {
          const lonInterpolator = d3.interpolate(oldLon, -subsolarLon);
          const latInterpolator = d3.interpolate(oldLat, -subsolarLat);
          
          return (t: number) => {
            projection.rotate([lonInterpolator(t), latInterpolator(t), 0]);
            pathGenerator = d3.geoPath().projection(projection);
            updateStationVisibility();
          };
        });
    } else {
      projection.rotate([-subsolarLon, -subsolarLat, 0]);
      pathGenerator = d3.geoPath().projection(projection);
      updateStationVisibility();
    }
    
    // Update EPIC data
    if (epicData) {
      epicData.centroid_coordinates = {
        lon: subsolarLon,
        lat: subsolarLat
      };
    }
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

  // Function to check if a satellite is within a station's cone
  const isInCone = (stationLat: number, stationLon: number, satX: number, satY: number, satZ: number, coneAngle: number): boolean => {
    // Convert station coordinates to 3D cartesian (Earth radius = 1)
    const stationRad = Math.PI / 180;
    const stationX = Math.cos(stationLat * stationRad) * Math.cos(stationLon * stationRad);
    const stationY = Math.cos(stationLat * stationRad) * Math.sin(stationLon * stationRad);
    const stationZ = Math.sin(stationLat * stationRad);

    // Normalize satellite coordinates
    const satDist = Math.sqrt(satX*satX + satY*satY + satZ*satZ);
    const satXNorm = satX / satDist;
    const satYNorm = satY / satDist;
    const satZNorm = satZ / satDist;

    // Calculate angle between station's zenith and satellite direction
    const dotProduct = stationX*satXNorm + stationY*satYNorm + stationZ*satZNorm;
    const zenithAngle = Math.acos(dotProduct) * 180 / Math.PI;
    
    console.log(`Station: ${stationLat},${stationLon} - Zenith angle: ${zenithAngle}°`);
    
    // For southern hemisphere stations, we need to check the supplementary angle
    const effectiveAngle = stationLat < 0 ? 180 - zenithAngle : zenithAngle;
    return effectiveAngle <= coneAngle / 2;
  };

  // Modify fetchSatellitePositions to use correct URL and handle XML
  const fetchSatellitePositions = async (date: Date) => {
    try {
      
      
      // Create end date 24 hours after start date
      const endDate = new Date(date.getTime() + (24 * 60 * 60 * 1000));
      
      const url = `https://sscweb.gsfc.nasa.gov/WS/sscr/2/locations/ace,dscovr/${formatTime(date)},${formatTime(endDate)}/gse/`;
      console.log('url', url);
      const response = await fetch(url);
      const xmlText = await response.text();
      const parser = new DOMParser();
      const xmlDoc = parser.parseFromString(xmlText, "text/xml");
      
      satelliteData = xmlDoc;
    } catch (err: any) {
      console.error('Satellite data fetch error:', err);
      error = 'Failed to fetch satellite positions';
    }
  };

  // Modify updateStationVisibility to parse XML data
  const updateStationVisibility = () => {
    if (epicData && satelliteData) {
      const xCoords = Array.from(satelliteData.getElementsByTagName('X')).map(x => parseFloat(x.textContent || '0'));
      const yCoords = Array.from(satelliteData.getElementsByTagName('Y')).map(y => parseFloat(y.textContent || '0'));
      const zCoords = Array.from(satelliteData.getElementsByTagName('Z')).map(z => parseFloat(z.textContent || '0'));
      
      const lastIndex = xCoords.length - 1;
      console.log('Satellite GSE position:', {
        x: xCoords[lastIndex],
        y: yCoords[lastIndex],
        z: zCoords[lastIndex]
      });
      
      xBandStations.forEach(station => {
        // First check if the station is on the visible side of Earth
        const coords = projection([station.lon, station.lat]);
        const visible = coords && isPointVisible(coords, station.lon, station.lat);
        
        // Only check cone angle if station is visible
        station.canSeeDscovr = (visible ?? false) && isInCone(
          station.lat,
          station.lon,
          xCoords[lastIndex],
          yCoords[lastIndex],
          zCoords[lastIndex],
          station.cone
        );

        station.canSeeAce = (visible ?? false) && isInCone(
          station.lat,
          station.lon,
          xCoords[lastIndex],
          yCoords[lastIndex],
          zCoords[lastIndex],
          station.cone
        );
      });


      
      xBandStations = [...xBandStations];
    }
  };

  // Add reset function
  const resetToCurrentTime = () => {
    selectedDate = new Date();
    calculateEarthOrientation(selectedDate, true);
  };

  // Update the visualization function to fetch satellite data
  const updateVisualization = async (initialLoad = false) => {
    loading = true;
    error = null;
    
    try {
      if (initialLoad) {
        epicData = await fetchLatestEPICData();
      }
      
      if (!worldData) {
        const response = await fetch('/data/world-atlas.json');
        const topology = await response.json();
        worldData = feature(topology, topology.objects.countries);
      }

      await fetchSatellitePositions(selectedDate);
      calculateEarthOrientation(selectedDate, !initialLoad);
    } finally {
      loading = false;
    }
  };

  // Add new date/time control variables
  const ONE_YEAR_AGO = new Date();
  ONE_YEAR_AGO.setFullYear(ONE_YEAR_AGO.getFullYear() - 1);
  
  // Format date for display
  const formatDateDisplay = (date: Date) => {
    return date.toLocaleDateString(undefined, { 
      year: 'numeric', 
      month: 'short', 
      day: 'numeric' 
    });
  };

  // Format time for display
  const formatTimeDisplay = (date: Date) => {
    return date.toLocaleTimeString(undefined, { 
      hour: '2-digit', 
      minute: '2-digit',
      hour12: false 
    });
  };

  // Modify the handlers to use a temporary rotation state while sliding
  const handleDateChange = (e: Event) => {
    const target = e.target as HTMLInputElement;
    const newDate = new Date(parseInt(target.value));
    
    // Update date without animation while sliding
    selectedDate.setFullYear(newDate.getFullYear());
    selectedDate.setMonth(newDate.getMonth());
    selectedDate.setDate(newDate.getDate());
    selectedDate = new Date(selectedDate);
    
    // Update immediately without animation
    calculateEarthOrientation(selectedDate, false);
  };

  // Add debounced version for smooth final animation
  const debouncedDateAnimation = debounce((date: Date) => {
    calculateEarthOrientation(date, true);
  }, 150);

  // Modify time handler similarly
  const handleTimeChange = (e: Event) => {
    const target = e.target as HTMLInputElement;
    const hours = parseInt(target.value);
    
    // Update time without animation while sliding
    selectedDate.setHours(hours);
    selectedDate.setMinutes(0);
    selectedDate.setSeconds(0);
    selectedDate = new Date(selectedDate);
    
    // Update immediately without animation
    calculateEarthOrientation(selectedDate, false);
  };

  // Add debounced version for smooth final animation
  const debouncedTimeAnimation = debounce((date: Date) => {
    calculateEarthOrientation(date, true);
  }, 150);

  onMount(() => {
    document.documentElement.style.setProperty('--transition-duration', `${TRANSITION_DURATION}ms`);
    graticule = d3.geoGraticule()();
    updateVisualization(true);
  });
</script>

<style>
  :global(body) {
    background: black;
    margin: 0;
    padding: 0;
  }

  :global(h1) {
    margin: 0;
    padding: 0;
    line-height: .1;
  }

  .visualization-container {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    width: 100%;
    max-width: 100vw;
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }

  .visualization-wrapper {
    flex: 1;
    min-width: 0;
    aspect-ratio: 1;
    background: #000;
    border-radius: 0;
    overflow: hidden;
    position: relative;
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
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    background: #2a2a2a;
    padding: 0.5rem;
    border-radius: 0;
    color: white;
    box-sizing: border-box;
    max-height: 60vh;
    overflow-y: auto;
  }

  .station-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
    gap: 0.5rem;
    margin-top: 0.5rem;
    background: #454343;
    padding: 0.5rem;
    border-radius: 4px;
  }

  .station-grid-title {
    font-size: 1.2rem;
    margin: 0.3rem;
    text-align: center;
  }

  .station-item {
    font-size: 1rem;
    padding: 1cqw;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    border: 1px solid transparent;
    transition: background-color 0.2s, border-color 0.2s;
    opacity: 0.5;
  }

  .station-item[data-visible="true"] {
    opacity: 1;
    border-color: rgba(255, 255, 255, 0.2);
  }

  .station-item[data-visible="true"][data-hovered="true"] {
    background: rgba(255, 255, 255, 0.1);
    border-color: rgba(255, 255, 255, 0.3);
  }

  .station-color {
    width: 12px;
    height: 12px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .station-info {
    flex: 1;
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
  }

  .satellite-status {
    display: flex;
    gap: 0.3rem;
    margin-left: auto;
    font-size: 0.9rem;
  }

  .epic-info {
    font-size: 0.9rem;
    color: #ccc;
    line-height: 1.4;
  }

  .api-urls {
    font-size: 0.8rem;
    padding: 0.5rem;
    background: rgba(0,0,0,0.2);
    border-radius: 4px;
    margin-top: 0.5rem;
    word-break: break-all;
  }

  .api-urls a {
    color: inherit;
    text-decoration: none;
  }

  .api-urls a:hover {
    text-decoration: underline;
  }

  @media (max-width: 900px) {
    .visualization-container {
      flex-direction: row;
    }

    .controls {
      width: 300px;
      max-height: 100vh;
    }

    .station-grid {
      grid-template-columns: 1fr;
    }

    .station-grid-title {
      font-size: 1.2rem;
      margin-bottom: 0.5rem;
      text-align: center;
    } 

    .controls {
      font-size: 0.9rem;
    }

    .epic-info {
      font-size: 0.8rem;
    }

    .api-urls {
      font-size: 0.7rem;
    }

    .station-item {
      padding: 0.3rem;
      font-size: 0.8rem;
    }

    .satellite-status {
      font-size: 0.8rem;
    }
  }

  /* Replace the global SVG transition with conditional transitions */
  :global(svg path:not(.station-marker)[data-rotating="true"]) {
    transition: d var(--transition-duration) ease-out;
  }

  :global(svg path:not(.station-marker)[data-rotating="false"]) {
    transition: none;
  }

  /* Faster transition for station markers */
  :global(.station-marker) {
    transition: transform 200ms ease-out;
  }

  /* Remove transition duration from station position */
  :global(svg g[role="button"]) {
    transition: none;
  }

  .datetime-controls {
    display: flex;
    flex-direction: row;
    gap: 1rem;
    justify-content: center;
  }

  .slider-group {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    align-items: center;
    text-align: center;
    width: 45%;
  }

  .slider-group label {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    align-items: center;
    width: 100%;
  }

  input[type="range"] {
    width: 90%;
    background: linear-gradient(to right, #666, #333);
    height: 4px;
    border-radius: 4px;
    -webkit-appearance: none;
    appearance: none;
    cursor: pointer;
    box-shadow: inset 0 1px 3px rgba(0,0,0,0.4);
    margin: 10px 0;
    position: relative;
  }

  input[type="range"]::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 20px;
    height: 20px;
    background: #fff;
    border-radius: 50%;
    cursor: pointer;
    box-shadow: 0 2px 6px rgba(0,0,0,0.3);
    border: 2px solid #fff;
    transition: all 0.2s ease;
    position: relative;
  }

  input[type="range"]::-moz-range-thumb {
    width: 20px;
    height: 20px;
    background: #fff;
    border-radius: 50%;
    cursor: pointer;
    box-shadow: 0 2px 6px rgba(0,0,0,0.3);
    border: 2px solid #fff;
    transition: all 0.2s ease;
  }

  /* Hover effects */
  input[type="range"]::-webkit-slider-thumb:hover {
    box-shadow: 0 0 10px rgba(255,255,255,0.5);
  }

  input[type="range"]::-moz-range-thumb:hover {
    transform: scale(1.2);
    box-shadow: 0 0 10px rgba(255,255,255,0.5);
  }

  /* Active state */
  input[type="range"]:active::-webkit-slider-thumb {
    transform: translateY(-8px) scale(1.1);
  }

  input[type="range"]:active::-moz-range-thumb {
    transform: scale(1.1);
  }

  /* Track focus state */
  input[type="range"]:focus {
    outline: none;
  }

  input[type="range"]:focus::-webkit-slider-thumb {
    box-shadow: 0 0 0 2px rgba(255,255,255,0.3);
  }

  input[type="range"]:focus::-moz-range-thumb {
    box-shadow: 0 0 0 2px rgba(255,255,255,0.3);
  }

  .reset-button {
    padding: 0.5rem 1rem;
    border-radius: 4px;
    border: 1px solid #444;
    background: #333;
    color: white;
    cursor: pointer;
    transition: background 0.2s;
  }

  .reset-button:hover {
    background: #444;
  }
  /* Make hovered stations more prominent */
  :global(path.station-marker[data-hovered="true"]) {
    stroke-width: 2;
    filter: drop-shadow(0 0 2px white);
  }

  .status-indicator {
    display: flex;
    gap: 0.3rem;
  }

  :global(.check) {
    color: #4CAF50;  /* Green */
  }

  :global(.x) {
    color: #F44336;  /* Red */
  }
</style>

<div class="visualization-container">
  <div class="visualization-wrapper" 
       bind:clientWidth={width} 
       bind:clientHeight={height} 
       bind:this={container}>
    {#if loading}
      <div class="loader">Loading Earth data...</div>
    {:else if error}
      <div class="error">{error}</div>
    {:else if width > 0 && height > 0}
      <svg width="100%" height="100%" viewBox="0 0 {width} {height}">
        <!-- Earth sphere -->
        <circle cx={width/2} cy={height/2} r={width/2.5} fill="#1a3650" />

        <!-- Graticule -->
        <path d={pathGenerator(graticule)} 
              stroke="rgba(255,255,255,0.3)" 
              fill="none"
              data-rotating={isRotating} />

        <!-- Land masses -->
        {#if worldData}
          <path d={pathGenerator(worldData)} 
                fill="#3a5f5e" 
                stroke="#4a7a79" 
                stroke-width="0.4"
                data-rotating={isRotating} />
        {/if}

        <!-- EPIC centroid projection -->
        {#if epicData}
          <g>
            <circle cx={width/2} cy={height/2}
                    r="8" fill="rgba(255, 215, 0, 0.2)" stroke="#FFD700" stroke-width="2" stroke-opacity="0.5"
                    role="button"
                    tabindex="0"
                    on:mouseenter={() => hoveredStation = 'EPIC'}
                    on:mouseleave={() => hoveredStation = null} />
            
            {#if hoveredStation === 'EPIC'}
              <g transform={`translate(${width/2 + 10}, ${height/2 - 10})`}>
                <rect x="0" y="-40" 
                      width="160" height="40" 
                      fill="rgba(0,0,0,0.8)" 
                      rx="4" />
                <text x="5" y="-25" fill="white" font-size="12">
                  EPIC View Center
                </text>
                <text x="5" y="-10" fill="white" font-size="12">
                  {formatCoords(epicData.centroid_coordinates.lat, epicData.centroid_coordinates.lon)}
                </text>
              </g>
            {/if}
          </g>
        {/if}

        <!-- Ground stations -->
        {#each xBandStations as station}
          {@const coords = projection([station.lon, station.lat])}
          {@const [lambda, phi] = projection.rotate()}
          {@const angle = Math.acos(
            Math.sin(station.lat * Math.PI/180) * Math.sin(-phi * Math.PI/180) +
            Math.cos(station.lat * Math.PI/180) * Math.cos(-phi * Math.PI/180) * 
            Math.cos((station.lon + lambda) * Math.PI/180)
          )}
          {@const scale = Math.max(0.3, Math.cos(angle))}
          {@const rotationAngle = Math.atan2(
            Math.cos(station.lat * Math.PI/180) * Math.sin((station.lon + lambda) * Math.PI/180),
            Math.cos(-phi * Math.PI/180) * Math.sin(station.lat * Math.PI/180) -
            Math.sin(-phi * Math.PI/180) * Math.cos(station.lat * Math.PI/180) * 
            Math.cos((station.lon + lambda) * Math.PI/180)
          ) * 180/Math.PI}
          {#if coords && isPointVisible(coords, station.lon, station.lat)}
            <g role="button"
               tabindex="0"
               transform={`translate(${coords})`}
               on:mouseenter={() => hoveredStation = station.name}
               on:mouseleave={() => hoveredStation = null}>
              <!-- Cylinder extending outward -->
              <path 
                class="station-cylinder"
                d={`M 0 -30 L 10 0 L -10 0 Z`}
                transform={`rotate(${rotationAngle})`}
                fill={station.color}
                opacity="0.4"
              />
              <!-- Station marker circle -->
              <path 
                class="station-marker"
                data-hovered={hoveredStation === station.name}
                d={`M ${-10} 0 A ${10} ${10} 0 1 1 ${10} 0 A ${10} ${10} 0 1 1 ${-10} 0`}
                transform={`rotate(${rotationAngle}) scale(${hoveredStation === station.name ? 1.5 : 1},${scale * (hoveredStation === station.name ? 1.5 : 1)})`}
                fill={station.color}
                stroke="white"
                stroke-width="1.5" />
              
              <!-- Hover tooltip -->
              {#if hoveredStation === station.name}
                <g transform="translate(10, -10)">
                  <rect x="0" y="-50" 
                        width="200" height="50" 
                        fill="rgba(0,0,0,0.8)" 
                        rx="4" />
                  <text x="5" y="-35" fill="white" font-size="12">
                    {station.name}
                  </text>
                  <text x="5" y="-20" fill="white" font-size="12">
                    {formatCoords(station.lat, station.lon)}
                    Local: {getLocalTime(station.lon)}
                  </text>
                  <text x="5" y="-5" fill="white" font-size="12">
                    DSCOVR: <tspan class={station.canSeeDscovr ? 'check' : 'x'}>{station.canSeeDscovr ? '✓' : '✗'}</tspan>
                    ACE: <tspan class={station.canSeeAce ? 'check' : 'x'}>{station.canSeeAce ? '✓' : '✗'}</tspan>
                  </text>
                </g>
              {/if}
            </g>
          {/if}
        {/each}
      </svg>
    {/if}
  </div>

  <div class="controls">
    <div class="datetime-controls">
      <div class="slider-group">
        <label>
          Date: {formatDateDisplay(selectedDate)}
          <input 
            type="range" 
            min={ONE_YEAR_AGO.getTime()} 
            max={Date.now()} 
            value={selectedDate.getTime()} 
            step={24 * 60 * 60 * 1000}
            on:input={handleDateChange}
          />
        </label>
      </div>
      <div class="slider-group">
        <label>
          Time: {formatTimeDisplay(selectedDate)}
          <input 
            type="range" 
            min="0" 
            max="23" 
            value={selectedDate.getHours()} 
            step="1"
            on:input={handleTimeChange}
          />
        </label>
      </div>
      <button class="reset-button" on:click={resetToCurrentTime}>
        Reset to Now
      </button>
    </div>
    <h2 class="station-grid-title">Currently Visible Stations</h2>
    <div class="station-grid">
      
      {#each xBandStations as station}
        {@const coords = projection([station.lon, station.lat])}
        {@const [lambda, phi] = projection.rotate()}
        {@const angle = Math.acos(
          Math.sin(station.lat * Math.PI/180) * Math.sin(-phi * Math.PI/180) +
          Math.cos(station.lat * Math.PI/180) * Math.cos(-phi * Math.PI/180) * 
          Math.cos((station.lon + lambda) * Math.PI/180)
        )}
        {@const scale = Math.max(0.3, Math.cos(angle))}
        {@const rotationAngle = Math.atan2(
          Math.cos(station.lat * Math.PI/180) * Math.sin((station.lon + lambda) * Math.PI/180),
          Math.cos(-phi * Math.PI/180) * Math.sin(station.lat * Math.PI/180) -
          Math.sin(-phi * Math.PI/180) * Math.cos(station.lat * Math.PI/180) * 
          Math.cos((station.lon + lambda) * Math.PI/180)
        ) * 180/Math.PI}
        {#if coords && isPointVisible(coords, station.lon, station.lat)}
          <div class="station-item"
               role="button"
               tabindex="0"
               data-visible={coords && isPointVisible(coords, station.lon, station.lat)}
               data-hovered={hoveredStation === station.name}
               on:mouseenter={() => hoveredStation = station.name}
               on:mouseleave={() => hoveredStation = null}>
            <div class="station-color" style="background-color: {station.color}"></div>
            <div class="station-info">
              <span>{station.name}</span>
              <div class="satellite-status">
                <span class="status-indicator">
                  DSCOVR: <span class={station.canSeeDscovr ? 'check' : 'x'}>{station.canSeeDscovr ? '✓' : '✗'}</span>
                </span>
                <span class="status-indicator">
                  ACE: <span class={station.canSeeAce ? 'check' : 'x'}>{station.canSeeAce ? '✓' : '✗'}</span>
                </span>
              </div>
            </div>
          </div>
        {/if}
      {/each}
    </div>

    {#if epicData}
      <div class="epic-info">
        Using EPIC data from: {new Date(epicData.date).toLocaleDateString()}<br>
        Extrapolated position for: {selectedDate.toLocaleString()}<br>
        <div>
          <div class="api-urls">
            EPIC: <code><a href="https://epic.gsfc.nasa.gov/api/enhanced/" target="_blank">https://epic.gsfc.nasa.gov/api/enhanced/</a></code><br>
            Satellite: <code><a href="https://sscweb.gsfc.nasa.gov/WS/sscr/2/locations/ace,dscovr/{formatTime(selectedDate)},{formatTime(new Date(selectedDate.getTime() + 24*60*60*1000))}/gse/" target="_blank">https://sscweb.gsfc.nasa.gov/WS/sscr/2/locations/ace,dscovr/{formatTime(selectedDate)},{formatTime(new Date(selectedDate.getTime() + 24*60*60*1000))}/gse/</a></code>
          </div>
        </div>
      </div>
    {/if}
  </div>
</div>