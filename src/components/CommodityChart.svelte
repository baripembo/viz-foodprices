<script>
  import { scaleTime, scaleLinear, line, min, max, extent } from 'd3'

  let { title, unit, pricetype, data, xDomain } = $props()

  const W = 280, H = 170
  const m = { top: 34, right: 20, bottom: 28, left: 52 }
  const iW = W - m.left - m.right
  const iH = H - m.top - m.bottom

  const USD_COLOR = '#007CE0'   /* hdx-sapphire */

  function fmtUSD(v) {
    if (v >= 100) return '$' + Math.round(v)
    if (v >= 10) return '$' + v.toFixed(1)
    return '$' + v.toFixed(2)
  }

  let usdData = $derived(data.filter(d => d.usdprice > 0).sort((a, b) => a.date - b.date))

  let x = $derived(
    scaleTime()
      .domain(xDomain ?? extent(usdData, d => d.date))
      .range([0, iW])
  )

  let y = $derived(
    scaleLinear()
      .domain([min(usdData, d => d.usdprice) ?? 0, max(usdData, d => d.usdprice) || 1])
      .nice()
      .range([iH, 0])
  )

  let usdPath = $derived(
    line().x(d => x(d.date)).y(d => y(d.usdprice))(usdData)
  )

  // One tick per year, deduplicated
  let xTicks = $derived(
    x.ticks(4).filter((tick, i, arr) =>
      i === 0 || tick.getFullYear() !== arr[i - 1].getFullYear()
    )
  )
  let yTicks = $derived(y.ticks(4))

  let lastUSD = $derived(usdData.at(-1))
</script>

<svg viewBox="0 0 {W} {H}" width="100%" style="display:block; overflow:visible">
  <rect width={W} height={H} fill="#fff" />

  <!-- title -->
  <text x={m.left} y={20} font-size="13" font-weight="700"
    font-family="sans-serif" text-anchor="start">
    <tspan fill="#111">{title}</tspan> <tspan fill="#7a7a7a" font-weight="400"> - {pricetype} per {unit}</tspan>
  </text>

  <g transform="translate({m.left},{m.top})">

    <!-- grid lines -->
    {#each yTicks as tick}
      <line x1={0} x2={iW} y1={y(tick)} y2={y(tick)} stroke="#ebebeb" />
    {/each}

    <!-- x axis -->
    {#each xTicks as tick}
      <g transform="translate({x(tick)},{iH})">
        <line y2="4" stroke="#ccc" />
        <text y="16" text-anchor="middle" font-size="11" fill="#888"
          font-family="sans-serif">{tick.getFullYear()}</text>
      </g>
    {/each}
    <line x2={iW} y1={iH} y2={iH} stroke="#ccc" />

    <!-- y axis -->
    {#each yTicks as tick}
      <text x="-6" y={y(tick)} dy="0.32em" text-anchor="end" font-size="11"
        fill="#888" font-family="sans-serif">{fmtUSD(tick)}</text>
    {/each}

    <!-- line -->
    {#if usdPath}
      <path d={usdPath} fill="none" stroke={USD_COLOR} stroke-width="1.5" />
    {/if}

    <!-- end label -->
    {#if lastUSD}
      {@const uy = y(lastUSD.usdprice)}
      {@const ux = x(lastUSD.date)}
      <rect x={ux - 26} y={uy - 8} width={26} height={16} fill={USD_COLOR} rx="1" />
      <text x={ux - 13} y={uy + 4} text-anchor="middle" font-size="10" fill="#fff"
        font-family="sans-serif" font-weight="600">{fmtUSD(lastUSD.usdprice)}</text>
    {/if}

  </g>
</svg>
