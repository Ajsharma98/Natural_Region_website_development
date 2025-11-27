<script>
  import { onMount } from 'svelte';
  import Map from './lib/Components/Map.svelte';

  let parameters = [];
  let selectedParameter = '';
  let yearInput = ''; // user types year or fiscal year (e.g., 2015 or 2015-16)
  let loading = false;
  let error = '';

  // load parameter list from backend on mount
  onMount(async () => {
    try {
      const res = await fetch('http://localhost:5000/api/parameters');
      if (!res.ok) throw new Error('Failed to fetch parameters');
      parameters = await res.json();
      if (parameters.length > 0) {
        selectedParameter = parameters[0].name;
      }
    } catch (err) {
      console.error(err);
      error = 'Could not load parameters. Check backend.';
    }
  });

  // will be set by Map component via binding
  let mapRef;

  function handleLoadMap() {
    error = '';
    if (!selectedParameter || !yearInput) {
      error = 'Please choose a parameter and enter a year.';
      return;
    }
    // instruct Map component to load data
    mapRef.loadMapData(selectedParameter, yearInput);
  }
</script>

<div class="app-container">
  <div class="toolbar">
    <label>
      Parameter:
      <select bind:value={selectedParameter}>
        {#each parameters as p}
          <option value={p.name}>{p.name}</option>
        {/each}
      </select>
    </label>

    <label>
      Year:
      <input placeholder="e.g. 2015 or 2015-16" bind:value={yearInput} />
    </label>

    <button on:click={handleLoadMap}>Load Map</button>

    <span style="margin-left:auto;color:#666">{#if loading}Loading...{/if}</span>
  </div>

  {#if error}
    <div style="padding:8px;background:#fee;border:1px solid #fbb;margin:8px;color:#900">
      {error}
    </div>
  {/if}

  <div class="map-wrap">
    <Map bind:this={mapRef} />
  </div>
</div>
