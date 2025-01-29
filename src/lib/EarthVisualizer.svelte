<script lang="ts">
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    import {type GeoProjection, type GeoPath } from 'd3-geo';
  
    // X-band stations
    const xBandStations = [
      { name: "Thermopylae, GR", lat: 38.7965, lon: 22.5364, cone: 170, color: "#FF0000" },
      { name: "Nemea, GR", lat: 37.8103, lon: 22.6544, cone: 170, color: "#00FF00" },
      { name: "TBD, AUS", lat: -35.3082, lon: 149.1244, cone: 170, color: "#0000FF" },
      { name: "Avarua, NZ", lat: -21.2291, lon: -159.7763, cone: 170, color: "#FFA500" }
    ];
  
    let earthImage = '';
    let satelliteData: Array<{ name: string; gseY: number; gseZ: number; time: string }> = [];
    let subsolarPoint = { lat: 0, lon: 0 };
    let projection: GeoProjection, pathGenerator: GeoPath;
    const width = 800;
    const height = 800;
  
    // Access environment variable
    const NASA_API_KEY = import.meta.env.VITE_NASA_API_KEY;
  
    // Add reactive variable for hover state
    let hoveredStation: string | null = null;
  
    // Add function to get local time for a station
    function getStationLocalTime(lon: number): string {
      const date = new Date();
      const utc = date.getTime() + (date.getTimezoneOffset() * 60000);
      const localTime = new Date(utc + (3600000 * (lon/15)));
      return localTime.toLocaleTimeString();
    }
  
    // Coverage area generator
    function createCoverageGeoJSON(station: typeof xBandStations[number]): GeoJSON.Feature {
      const radius = (90 - station.cone/2) * (Math.PI/180);
      return {
        type: "Feature" as const,
        geometry: {
          type: "Polygon" as const,
          coordinates: [d3.geoCircle()
            .center([station.lon, station.lat])
            .radius(radius)()
            .coordinates[0]]
        },
        properties: {}
      };
    }
  
    async function fetchEPICData() {
      const epicRes = await fetch(
        `https://api.nasa.gov/EPIC/api/natural?api_key=${NASA_API_KEY}`
      );
      const epicData = await epicRes.json();
      
      if (epicData?.[0]) {
        const date = epicData[0].date.split(' ')[0].replace(/-/g, '/');
        const imageName = epicData[0].image;
        
        subsolarPoint = {
          lat: epicData[0].centroid_coordinates.lat,
          lon: epicData[0].centroid_coordinates.lon
        };
        earthImage = `https://epic.gsfc.nasa.gov/archive/natural/${date}/png/${imageName}.png`;
      }
      return epicData;
    }
  
    function formatSatelliteDate(date: Date) {
      return date.toISOString()
        .replace(/[-:]/g, '')    // Remove dashes and colons
        .replace(/\.\d{3}/, '')  // Remove milliseconds
        .replace(/Z$/, 'Z');     // Keep the Z
    }
  
    async function fetchSatelliteData() {
      const endDate = new Date();
      const startDate = new Date(endDate);
      startDate.setHours(startDate.getHours() - 1);
      
      const satURL = `https://sscweb.gsfc.nasa.gov/WS/sscr/2/locations/ace,dscovr/${formatSatelliteDate(startDate)},${formatSatelliteDate(endDate)}/gse/`;
      
      const satRes = await fetch(satURL);
      const xmlText = await satRes.text();
      const parser = new DOMParser();
      const xmlDoc = parser.parseFromString(xmlText, "text/xml");
      
      const dataElements = xmlDoc.querySelectorAll('Data');
      return Array.from(dataElements).map(data => {
        const coords = data.querySelector('Coordinates');
        const name = data.querySelector('Id')?.textContent || '';
        // Get the last Y and Z values from the arrays
        const yValues = coords?.querySelectorAll('Y');
        const zValues = coords?.querySelectorAll('Z');
        const lastY = yValues?.[yValues.length - 1]?.textContent;
        const lastZ = zValues?.[zValues.length - 1]?.textContent;
        
        return {
          name: name,
          gseY: parseFloat(lastY || '0'),
          gseZ: parseFloat(lastZ || '0'),
          time: data.querySelector('Time')?.textContent || '',
          imageDate: data.querySelector('Time')?.textContent || '',
          satelliteURL: `https://sscweb.gsfc.nasa.gov/WS/sscr/2/locations/ace,dscovr/${formatSatelliteDate(startDate)},${formatSatelliteDate(endDate)}/gse/`
        };
      });
    }
  
    async function fetchData() {
      try {
        const epicData = await fetchEPICData();
        satelliteData = await fetchSatelliteData();
  
        console.log('EPIC Data:', epicData);
        console.log('Earth Image URL:', earthImage);
        console.log('Satellite Data:', satelliteData);
      } catch (err) {
        console.error('Data fetch error:', err);
      }
    }
  
    // Add this helper function to log visible features
    function logVisibleFeatures() {
      if (!projection) return;
      
      // Test points for major landmarks
      const testPoints = [
        { name: "Australia (Center)", lat: -25.2744, lon: 133.7751 },
        { name: "New Zealand", lat: -40.9006, lon: 174.8860 },
        { name: "Greece", lat: 39.0742, lon: 21.8243 },
        { name: "Antarctica", lat: -90, lon: 0 }
      ];
      
      testPoints.forEach(point => {
        const projected = projection([point.lon, point.lat]);
        const visible = projection.rotate();
        console.log(`${point.name}: `, {
          position: projected,
          visible: !!projected,
          rotation: visible
        });
      });
    }
  
    // Update the projection function to match EPIC perspective
    function updateProjection() {
      // EPIC views Earth from L1 point, approximately 1.5 million km away
      projection = d3.geoOrthographic()
        .scale(250)
        .translate([width/2, height/2])
        // Adjust rotation based on EPIC image orientation
        // The subsolar point helps us align the sunlit portion correctly
        .rotate([
          -subsolarPoint.lon,  // Longitude rotation
          -subsolarPoint.lat,  // Latitude rotation
          0                    // Roll angle
        ])
        .clipAngle(90);
  
      pathGenerator = d3.geoPath(projection);
      
      // Log current orientation for debugging
      logVisibleFeatures();
    }
  
    // Add a function to check if a point should be visible
    function isPointVisible(lon: number, lat: number): boolean {
      if (!projection) return false;
      
      const coords = projection([lon, lat]);
      if (!coords) return false;
      
      // Get the current rotation
      const [lambda, phi] = projection.rotate();
      
      // Convert to radians
      const lonRad = -lon * Math.PI / 180;
      const latRad = lat * Math.PI / 180;
      const lambdaRad = lambda * Math.PI / 180;
      const phiRad = phi * Math.PI / 180;
      
      // Calculate great circle distance to visible center
      const cosDistance = Math.sin(latRad) * Math.sin(-phiRad) +
                         Math.cos(latRad) * Math.cos(-phiRad) * 
                         Math.cos(lonRad - lambdaRad);
      
      // Point is visible if less than 90 degrees from center
      return cosDistance > 0;
    }
  
    onMount(() => {
      fetchData();
      updateProjection();
      
      const interval = setInterval(async () => {
        await fetchData();
        updateProjection();
      }, 3600000);

      return () => clearInterval(interval);
    });
  </script>
  
  <div class="visualization-container">
    <svg {width} {height} viewBox="0 0 {width} {height}" class="visualization">
      {#if earthImage}
        <image href={earthImage} x="0" y="0" width={width} height={height} />
      {/if}
    
      <!-- Coverage Areas -->
      {#each xBandStations as station}
        {#if pathGenerator && isPointVisible(station.lon, station.lat)}
          <path
            d={pathGenerator(createCoverageGeoJSON(station))}
            fill={station.color}
            fill-opacity="0.2"
            stroke={station.color}
          />
        {/if}
      {/each}
    
      <!-- Ground Stations -->
      {#each xBandStations as station}
        {#if projection && isPointVisible(station.lon, station.lat)}
          <g 
            role="button"
            tabindex="0"
            on:mouseenter={() => hoveredStation = station.name}
            on:mouseleave={() => hoveredStation = null}
          >
            <circle
              cx={projection([station.lon, station.lat])?.[0]}
              cy={projection([station.lon, station.lat])?.[1]}
              r="4"
              fill={station.color}
              class:highlighted={hoveredStation === station.name}
            />
            {#if hoveredStation === station.name}
              {#if projection}
                {@const coords = projection([station.lon, station.lat]) || [0, 0]}
                <g transform="translate({coords[0]}, {coords[1] - 10})">
                  <rect
                    x="-100"
                    y="-40"
                    width="200"
                    height="35"
                    fill="white"
                    stroke={station.color}
                    rx="4"
                  />
                  <text
                    x="0"
                    y="-20"
                    text-anchor="middle"
                    fill="black"
                    font-size="12px"
                  >{station.name}</text>
                  <text
                    x="0"
                    y="-5"
                    text-anchor="middle"
                    fill="black"
                    font-size="12px"
                  >Local Time: {getStationLocalTime(station.lon)}</text>
                </g>
              {/if}
            {/if}
          </g>
        {/if}
      {/each}
    
      <!-- Satellites -->
      {#each satelliteData as sat}
        <circle
          cx={width/2 + (sat.gseY / 6371) * 250}
          cy={height/2 - (sat.gseZ / 6371) * 250}
          r="5"
          fill={sat.name === 'DSCOVR' ? '#00FF00' : '#0000FF'}
        />
      {/each}
    </svg>
  </div>
  
  <div class="info-section">
    <h2>Ground Station Information</h2>
    <div class="station-list">
      {#each xBandStations as station}
        <div 
          class="station-item"
          class:highlighted={hoveredStation === station.name}
          role="button"
          tabindex="0"
          on:mouseenter={() => hoveredStation = station.name}
          on:mouseleave={() => hoveredStation = null}
        >
          <div class="color-dot" style="background-color: {station.color}"></div>
          <span>{station.name} ({station.lat}°, {station.lon}°)</span>
        </div>
      {/each}
    </div>
  </div>

  <!-- <div class="info-section">
    <h2>API URLs</h2>
    <div class="api-url">
      <strong>EPIC Image:</strong>
      <code>https://epic.gsfc.nasa.gov/api/natural/date/{imageDate.split('T')[0]}</code>
    </div>
  </div> -->

  <div class="antenna-image">
    <img src="/SWFO-Antenna-Locations.png" alt="SWFO Antenna Locations" />
  </div>
  
  <style>
    .visualization-container {
      display: flex;
      justify-content: center;
      width: 100%;
      margin: 0 auto;
    }

    .visualization {
      background: #f0f4f8;
      border-radius: 8px;
      box-shadow: 0 4px 6px rgba(0,0,0,0.1);
    }

    .info-section {
      margin-top: 2rem;
      padding: 1rem;
      border-radius: 8px;
      background-color: #f5f5f5;
    }

    h2 {
      margin: 0 0 1rem 0;
      font-size: 1.5rem;
      color: #333;
    }

    .station-list {
      display: flex;
      flex-direction: column;
      gap: 0.5rem;
    }

    .station-item {
      padding: 4px 8px;
      border-radius: 4px;
      transition: background-color 0.2s;
    }

    .station-item.highlighted {
      background-color: rgba(0, 0, 0, 0.1);
    }

    .color-dot {
      width: 12px;
      height: 12px;
      border-radius: 50%;
    }

    .antenna-image {
      margin-top: 2rem;
      text-align: center;
    }

    .antenna-image img {
      max-width: 100%;
      height: auto;
      border-radius: 8px;
      box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    }

    circle {
      transition: r 0.2s;
    }

    circle.highlighted {
      r: 6;
    }

    g text {
      user-select: none;
    }
  </style>