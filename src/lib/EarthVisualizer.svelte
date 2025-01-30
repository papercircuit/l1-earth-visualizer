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

  // Update projection when size changes
  $: if (width && height && width > 0 && height > 0) {
    const size = Math.min(width, height);
    projection = d3.geoOrthographic()
      .scale(size / 2.5)
      .translate([width / 2, height / 2])
      .clipAngle(90);
    pathGenerator = d3.geoPath().projection(projection);
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

  // Modify calculateEarthOrientation to always update station visibility
  const calculateEarthOrientation = (date: Date | string, animate = false) => {
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

  onMount(() => {
    document.documentElement.style.setProperty('--transition-duration', `${TRANSITION_DURATION}ms`);
    updateVisualization(true);
  });
</script>

<style>
  .visualization-container {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    width: 100%;
    max-width: min(100vw, 100vh);
    margin: 0 auto;
  }

  .visualization-wrapper {
    width: 100%;
    aspect-ratio: 1;
    background: #000;
    border-radius: 8px;
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
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
    width: 100%;
    background: #2a2a2a;
    padding: 1rem;
    border-radius: 8px;
    color: white;
    box-sizing: border-box;
  }

  .datetime-input {
    font-size: 1.1rem;
    padding: 0.5rem;
    border-radius: 4px;
    border: 1px solid #444;
    background: #333;
    color: white;
    width: 40%;
  }

  .epic-info {
    font-size: 0.9rem;
    color: #ccc;
    line-height: 1.4;
  }

  /* Replace the global SVG transition with specific transitions */
  :global(svg path:not(.station-marker)) {
    transition: all var(--transition-duration) ease-out;
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
    gap: 1rem;
    align-items: center;
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

  .station-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
    gap: 0.5rem;
    margin-top: 0.5rem;
  }

  .station-item {
    background: #333;
    padding: 0.5rem;
    border-radius: 4px;
    display: flex;
    align-items: center;
    gap: 0.5rem;
    cursor: pointer;
    transition: background-color 0.2s;
  }

  .station-item:hover,
  .station-item.hovered {
    background: #444;
  }

  .station-color {
    width: 14px;
    height: 13px;
    border-radius: 50%;
    border: 1px solid white;
  }

  .satellite-status {
    display: flex;
    gap: 1rem;
    margin-left: auto;
  }

  .status-indicator {
    display: flex;
    align-items: center;
    gap: 0.25rem;
  }

  /* Make hovered stations more prominent */
  :global(path.station-marker[data-hovered="true"]) {
    stroke-width: 2;
    filter: drop-shadow(0 0 2px white);
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
                    DSCOVR: {station.canSeeDscovr ? '✓' : '✗'}
                    ACE: {station.canSeeAce ? '✓' : '✗'}
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
      <input class="datetime-input"
             type="datetime-local" 
             value={formatDateForInput(selectedDate)}
             max={formatDateForInput(new Date())}
             on:input={(e: Event) => {
               const target = e.target as HTMLInputElement;
               selectedDate = new Date(target.value);
               calculateEarthOrientation(selectedDate, true);
             }}>
      <button class="reset-button" on:click={resetToCurrentTime}>
        Reset to Now
      </button>
    </div>
    
    <div class="station-grid">
      {#each xBandStations as station}
        <div class="station-item"
             role="button"
             tabindex="0"
             class:hovered={hoveredStation === station.name}
             on:mouseenter={() => hoveredStation = station.name}
             on:mouseleave={() => hoveredStation = null}>
          <div class="station-color" style="background-color: {station.color}"></div>
          <span>{station.name}</span>
          <div class="satellite-status">
            <span class="status-indicator">
              DSCOVR: {station.canSeeDscovr ? '✓' : '✗'}
            </span>
            <span class="status-indicator">
              ACE: {station.canSeeAce ? '✓' : '✗'}
            </span>
          </div>
        </div>
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