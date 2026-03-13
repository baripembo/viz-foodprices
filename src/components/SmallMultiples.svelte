<script>
  import { rollup, mean, extent } from 'd3'
  import CommodityChart from './CommodityChart.svelte'

  let { data } = $props()

  const EXCLUDE = new Set(['Exchange rate (unofficial)'])

  let commodities = $derived(
    [...new Set(
      data
        .filter(d => !EXCLUDE.has(d.commodity))
        .map(d => d.commodity)
    )].sort()
  )

  // Global date extent across all commodities for a consistent x-axis
  let xDomain = $derived(
    extent(
      data.filter(d => !EXCLUDE.has(d.commodity) && d.usdprice > 0),
      d => d.date
    )
  )

  // Average price and usdprice across markets per commodity per date
  let byCommodity = $derived(
    Object.fromEntries(
      commodities.map(commodity => {
        const subset = data.filter(d => d.commodity === commodity)
        const aggregated = Array.from(
          rollup(
            subset,
            v => ({
              price: mean(v, d => d.price),
              usdprice: mean(v, d => d.usdprice > 0 ? d.usdprice : null)
            }),
            d => d.date
          ),
          ([date, vals]) => ({ date, ...vals })
        ).sort((a, b) => a.date - b.date)
        return [commodity, aggregated]
      })
    )
  )

  // Unit and pricetype per commodity (constant — take from first row)
  let unitByCommodity = $derived(
    Object.fromEntries(
      commodities.map(commodity => {
        const row = data.find(d => d.commodity === commodity)
        return [commodity, { unit: row?.unit ?? '', pricetype: row?.pricetype ?? '' }]
      })
    )
  )
</script>

<section>
  <div class="small-multiples">
    {#each commodities as commodity}
      <CommodityChart title={commodity} unit={unitByCommodity[commodity].unit} pricetype={unitByCommodity[commodity].pricetype} data={byCommodity[commodity]} {xDomain} />
    {/each}
  </div>

  <!-- <p class="chart-source">Source: WFP VAM Food Price Monitoring</p> -->
</section>

<style>
  .legend {
    margin-bottom: 1rem;
  }
</style>
