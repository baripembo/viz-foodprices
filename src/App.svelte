<script>
  import { onMount } from 'svelte'
  import { csvParse, autoType } from 'd3'
  import SmallMultiples from './components/SmallMultiples.svelte'
  import AdminView from './components/AdminView.svelte'

  const DATA_FROM = new Date(2025, 6, 1) // July 1 2025, local time

  let data = $state([])
  let currentView = $state(new URLSearchParams(window.location.search).get('view') ?? 'admin')

  onMount(async () => {
    const raw = await fetch(`${import.meta.env.BASE_URL}data/wfp_food_prices_lbn.csv`).then(r => r.text())
    const parsed = csvParse(raw, autoType)
    data = parsed.filter(d => d.date >= DATA_FROM)
  })
</script>

<main>
  <h1>Lebanon Food Prices</h1>

  {#if data.length === 0}
    <p>Loading...</p>
  {:else if currentView === 'overview'}
    <SmallMultiples {data} />
  {:else}
    <AdminView {data} adminField="admin1" dataFrom={DATA_FROM} />
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

  main h1 {
    margin-bottom: 1.5rem;
  }

</style>
