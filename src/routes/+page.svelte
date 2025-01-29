<script lang="ts">
    import EarthVisualizer from '../lib/EarthVisualizer.svelte';
  </script>
  
  <main>
    <h1>L1 Earth Visualizer</h1>
    <EarthVisualizer />
  </main>
  
  <style>
    main {
      max-width: 1200px;
      margin: 0 auto;
      padding: 2rem;
    }
    
    h1 {
      text-align: center;
      margin-bottom: 2rem;
      color: #1a237e;
    }
  </style>