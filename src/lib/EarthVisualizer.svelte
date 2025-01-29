<script lang="ts">
    // Add this type definition
    type EpicData = {
        centroid_coordinates: {
            lat: number;
            lon: number;
        };
    };

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
    let satelliteData: Array<{ 
      name: string; 
      gseY: number; 
      gseZ: number; 
      time: string;
      satelliteURL: string;
    }> = [];
    let subsolarPoint = { lat: 0, lon: 0 };
    const width = 800;
    const height = 800;
  
    let projection = d3.geoOrthographic()
      .scale(width / 2.5)
      .translate([width / 2, height / 2])
      .clipAngle(90);
    let pathGenerator = d3.geoPath().projection(projection);
  
    // Access environment variable
    const NASA_API_KEY = import.meta.env.VITE_NASA_API_KEY;
  
    // Add reactive variable for hover state
    let hoveredStation: string | null = null;
  
    // Initialize with two days ago by default
    const twoDaysAgo = new Date();
    twoDaysAgo.setDate(twoDaysAgo.getDate() - 2);
    let selectedDate = twoDaysAgo.toISOString().split('T')[0]; // Two days ago in YYYY-MM-DD format
    let availableTimes: string[] = [];
    let selectedTime: string = '';
  
    // Set minimum date to 1 year ago
    const oneYearAgo = new Date();
    oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1);
    const minDate = oneYearAgo.toISOString().split('T')[0];
  
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
  
    // Modify fetchEPICData to handle the date string correctly
    async function fetchEPICData() {
      try {
        console.log('Fetching EPIC data for date:', selectedDate);
        const epicURL = `https://epic.gsfc.nasa.gov/api/enhanced/date/${selectedDate}`;
        const epicRes = await fetch(epicURL);
        
        if (!epicRes.ok) {
          throw new Error(`HTTP error! status: ${epicRes.status}`);
        }
        
        const epicData = await epicRes.json();
        
        if (!epicData || epicData.length === 0) {
          throw new Error('No data available for selected date');
        }
        
        // Update available times for this date
        availableTimes = epicData.map((data: any) => data.date.split(' ')[1]);
        
        // If no time is selected, use the most recent
        if (!selectedTime && availableTimes.length > 0) {
          selectedTime = availableTimes[availableTimes.length - 1];
        }
        
        // Find the data point matching our selected time
        const selectedData = epicData.find((data: any) => data.date.split(' ')[1] === selectedTime);
        
        if (selectedData) {
          const date = selectedData.date.split(' ')[0].replace(/-/g, '/');
          const imageName = selectedData.image;
          
          subsolarPoint = {
            lat: selectedData.centroid_coordinates.lat,
            lon: selectedData.centroid_coordinates.lon
          };
          earthImage = `https://epic.gsfc.nasa.gov/archive/enhanced/${date}/png/${imageName}.png`;
          return selectedData;
        }
        throw new Error('No matching time data found');
      } catch (error) {
        console.error('Error fetching EPIC data:', error);
        return null;
      }
    }
  
    function formatSatelliteDate(date: Date) {
      return date.toISOString()
        .replace(/[-:]/g, '')    // Remove dashes and colons
        .replace(/\.\d{3}/, '')  // Remove milliseconds
        .replace(/Z$/, 'Z');     // Keep the Z
    }
  
    // Modify satellite data fetch to handle the date string correctly
    async function fetchSatelliteData() {
      try {
        const dateObj = new Date(selectedDate);
        const endDate = new Date(dateObj);
        endDate.setHours(endDate.getHours() + 1);
        const startDate = new Date(dateObj);
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
            satelliteURL: satURL
          };
        });
      } catch (error) {
        console.error('Error fetching satellite data:', error);
        return [];
      }
    }
  
    async function fetchData() {
      try {
        const epicData = await fetchEPICData();
        if (!epicData) {
          console.error('No EPIC data received');
          return null;
        }
        
        satelliteData = await fetchSatelliteData();

        console.log('EPIC Data:', epicData);
        console.log('Earth Image URL:', earthImage);
        console.log('Satellite Data:', satelliteData);
        
        return epicData;
      } catch (err) {
        console.error('Data fetch error:', err);
        return null;
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
  
    // Update the projection function to correctly match EPIC perspective
    function updateProjection(epicData: EpicData) {
      if (!projection) return;
      
      const lon = epicData.centroid_coordinates.lon;
      const lat = epicData.centroid_coordinates.lat;
      
      // Adjust scale to match EPIC image exactly
      projection = d3.geoOrthographic()
        .scale(width / 2.55)  // Make slightly smaller to match EPIC image
        .translate([width / 2, height / 1.94])
        .clipAngle(90);
        
      // Let's try inverting the longitude to fix NZ/AUS positions
      projection.rotate([lon, -lat, 0]);
      
      pathGenerator = d3.geoPath().projection(projection);
      
      // Debug log to check coordinates
      console.log('Station coordinates:', {
        aus: {
          raw: [149.1244, -35.3082],
          projected: projection([149.1244, -35.3082])
        },
        nz: {
          raw: [-159.7763, -21.2291],
          projected: projection([-159.7763, -21.2291])
        },
        epicCenter: { lon, lat },
        rotation: projection.rotate()
      });
    }
  
    // Update visibility check to match EPIC perspective
    function isPointVisible(lon: number, lat: number): boolean {
      return (lon > 90 || lon < -90);  // Pacific side
    }
  
    // Add this helper function to log station positions
    function logStationPositions() {
      if (!projection) return;
      
      xBandStations.forEach(station => {
        const pos = projection([station.lon, station.lat]);
        console.log(`Station ${station.name}:`, {
          rawCoords: [station.lon, station.lat],
          projectedPos: pos,
          currentRotation: projection.rotate()
        });
      });
    }
  
    // Add handler for date/time changes
    async function handleDateTimeChange() {
      console.log('Date/time changed to:', selectedDate, selectedTime);
      const epicData = await fetchData();
      if (epicData) {
        updateProjection(epicData);
        logStationPositions();
      }
    }
  
    // Initialize data on mount
    onMount(async () => {
      const epicData = await fetchData();
      if (epicData) {
        updateProjection(epicData);
        logStationPositions();
      }
    });
  </script>
  
  <div class="visualization-container">
    <div class="visualization-wrapper">
      <svg {width} {height} viewBox="0 0 {width} {height}" class="visualization">
        <!-- EPIC image first -->
        {#if earthImage}
          <image href={earthImage} x="0" y="0" width={width} height={height} />
        {/if}
      
        <!-- Border circle -->
        <circle
          cx={width/2}
          cy={height/1.94}
          r={width/2.55}
          fill="none"
          stroke="red"
          stroke-width="1"
        />
      
        <!-- Graticule on top -->
        {#if pathGenerator}
          <path
            d={pathGenerator(d3.geoGraticule()())}
            stroke="rgba(255,255,255,0.3)"
            fill="none"
            stroke-width="0.5"
          />
        {/if}
      
        <!-- Station rendering -->
        {#each xBandStations as station}
          {@const visible = isPointVisible(station.lon, station.lat)}
          {@const position = projection([station.lon, station.lat])}
          {#if visible && position}
            <g>
              <!-- Add coordinate debug text -->
              <text
                x={position[0] + 10}
                y={position[1] - 10}
                fill="white"
                font-size="10px"
              >{station.name}: {station.lon}°, {station.lat}°</text>
              
              <circle
                cx={position[0]}
                cy={position[1]}
                r="4"
                fill={station.color}
                class:highlighted={hoveredStation === station.name}
              />
            </g>
          {/if}
        {/each}
      
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
          {@const visible = isPointVisible(station.lon, station.lat)}
          {@const position = projection([station.lon, station.lat])}
          {#if visible && position}
            <g 
              role="button"
              tabindex="0"
              on:mouseenter={() => hoveredStation = station.name}
              on:mouseleave={() => hoveredStation = null}
            >
              <!-- Add coordinate debug text -->
              <text
                x={position[0] + 10}
                y={position[1] - 10}
                fill="white"
                font-size="10px"
              >{station.name}: {station.lon}°, {station.lat}°</text>
              
              <circle
                cx={position[0]}
                cy={position[1]}
                r="4"
                fill={station.color}
                class:highlighted={hoveredStation === station.name}
              />
              {#if hoveredStation === station.name}
                {@const coords = position || [0, 0]}
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
      
      <!-- Repositioned EPIC timestamp -->
      {#if earthImage}
        <div class="epic-timestamp">
          Image taken: {satelliteData[0]?.time || 'Loading...'} UTC
        </div>
      {/if}
    </div>
  </div>

   <!-- Reposition controls and add more user-friendly labels -->
   <div class="controls">
    <div class="control-group">
        <label for="date-picker">Select Date:</label>
        <input 
            id="date-picker"
            type="date" 
            bind:value={selectedDate}
            min={minDate}
            max={new Date().toISOString().split('T')[0]}
            on:change={handleDateTimeChange}
        />
    </div>
    
    <div class="control-group">
        <label for="time-picker">Select Time (UTC):</label>
        <select 
            id="time-picker"
            bind:value={selectedTime}
            on:change={handleDateTimeChange}
            disabled={availableTimes.length === 0}
        >
            {#each availableTimes as time}
                <option value={time}>{time} UTC</option>
            {/each}
        </select>
    </div>
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

  <!-- Add API URLs section -->
  <div class="info-section">
    <h2>API URLs</h2>
    <div class="api-urls">
      <div class="api-url">
        <strong>EPIC Data:</strong>
        <code><a href="https://api.nasa.gov/EPIC/api/natural">https://api.nasa.gov/EPIC/api/natural</a></code>
      </div>
      {#if satelliteData[0]}
        <div class="api-url">
          <strong>Satellite Data:</strong>
          <code><a href={satelliteData[0].satelliteURL}>{satelliteData[0].satelliteURL}</a></code>
        </div>
      {/if}
    </div>
  </div>

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

    .visualization-wrapper {
      position: relative;
      width: fit-content;
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
      display: flex;
      align-items: center;
      gap: 0.5rem;
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

    .epic-timestamp {
      position: absolute;
      bottom: 23px;
      left: 50%;
      transform: translateX(-50%);
      background-color: rgba(0, 0, 0, 0.6);
      color: rgba(255, 255, 255, 0.802);
      padding: 8px 16px;
      border-radius: 4px;
      font-size: 1rem;
      text-align: center;
    }

    .api-urls {
      display: flex;
      flex-direction: column;
      gap: 0.5rem;
    }

    .api-url {
      display: flex;
      align-items: center;
      gap: 0.5rem;
      border-radius: 4px;
      font-size: 0.9rem;
    }

    .api-url code a {
      color: #000;
      display: block;
      margin-top: 0.25rem;
      border-radius: 4px;
      overflow-x: auto;
    }

    /* Add styles for the new controls */
    .controls {
      margin: 2rem auto;
      padding: 1rem;
      display: flex;
      gap: 2rem;
      justify-content: center;
      background-color: #f5f5f5;
      border-radius: 8px;
      max-width: 800px;
    }

    .control-group {
      display: flex;
      flex-direction: column;
      gap: 0.5rem;
    }

    .control-group label {
      font-weight: bold;
      color: #333;
    }

    .controls input,
    .controls select {
      padding: 0.5rem;
      border: 1px solid #ccc;
      border-radius: 4px;
      font-size: 1rem;
    }

    .controls select:disabled {
      background-color: #eee;
      cursor: not-allowed;
    }
  </style>