<script>
  import { csvParse, autoType } from 'd3'
  import SmallMultiples from './components/SmallMultiples.svelte'
  import AdminView from './components/AdminView.svelte'

  const DATA_FROM = new Date(2025, 6, 1) // July 1 2025, local time

  const LOCATIONS = [
    { id: 'lbn', label: 'Lebanon', file: 'wfp_food_prices_lbn.csv' },
    { id: 'pse', label: 'Palestine', file: 'wfp_food_prices_pse.csv' },
  ]

  let data = $state([])
  let currentView = $state(new URLSearchParams(window.location.search).get('view') ?? 'admin')
  let selectedLocation = $state(LOCATIONS[0])

  async function loadData(location) {
    data = []
    const raw = await fetch(`${import.meta.env.BASE_URL}data/${location.file}`).then(r => r.text())
    const parsed = csvParse(raw, autoType)
    data = parsed.filter(d => d.date >= DATA_FROM)
  }

  $effect(() => {
    loadData(selectedLocation)
  })
</script>

<main>
  <h1 class="page-title">
    <select
      value={selectedLocation.id}
      onchange={e => selectedLocation = LOCATIONS.find(l => l.id === e.target.value)}
    >
      {#each LOCATIONS as loc}
        <option value={loc.id}>{loc.label}</option>
      {/each}
    </select>
    Food Prices
  </h1>

  {#if data.length === 0}
    <p>Loading...</p>
  {:else if currentView === 'overview'}
    <SmallMultiples {data} />
  {:else}
    <AdminView {data} dataFrom={DATA_FROM} />
  {/if}
</main>

<style>
  main {
    max-width: 1200px;
    margin: 2rem auto;
    font-family: var(--font-family);
    font-size: var(--font-size-base);
    padding: 0 1rem;
  }

  .page-title {
    display: flex;
    align-items: baseline;
    gap: 0.3rem;
    margin-bottom: 1.5rem;
  }

  .page-title select {
    font-family: var(--font-family);
    font-size: inherit;
    font-weight: inherit;
    color: var(--hdx-black);
    border: none;
    border-bottom: 2px solid var(--hdx-grey-mid);
    background: transparent;
    cursor: pointer;
    padding: 0 4px 1px;
    appearance: auto;
  }

  .page-title select:focus {
    outline: none;
    border-bottom-color: var(--hdx-sapphire);
  }
</style>
