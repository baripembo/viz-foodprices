<script>
  import { select, scaleTime, scaleLinear, extent, line,
           rollup, mean, timeFormat, timeYear, pointer } from 'd3'

  let { data, adminField = 'admin1' } = $props()

  const CUTOFF = new Date('2016-01-01')
  const EXCLUDE_COMMODITY = 'Exchange rate (unofficial)'
  const EXCLUDE_CATEGORY = 'non-food'

  let chartEl = $state(null)

  let baseData = $derived(
    data.filter(d =>
      d.usdprice > 0 &&
      d.date >= CUTOFF &&
      d.category !== EXCLUDE_CATEGORY &&
      d.commodity !== EXCLUDE_COMMODITY
    )
  )

  let admins = $derived(
    [...new Set(baseData.map(d => d[adminField]))].sort()
  )

  let commodities = $derived(
    [...new Set(baseData.map(d => d.commodity))].sort()
  )

  const DEFAULT_STAPLES = new Set([
    'Bread (pita)', 'Rice (imported, Egyptian)', 'Oil (sunflower)',
    'Lentils (red)', 'Eggs', 'Meat (chicken, whole, frozen)'
  ])

  let selectedAdmin = $state(null)
  let selectedCommodities = $state(new Set())
  let commoditiesInitialised = $state(false)

  // Functions set by renderChart, called from sidebar hover handlers
  let highlightFn = null
  let resetFn = null

  $effect(() => {
    if (admins.length > 0 && selectedAdmin === null) {
      selectedAdmin = admins[0]
    }
  })

  $effect(() => {
    if (commodities.length > 0 && !commoditiesInitialised) {
      const defaults = commodities.filter(c => DEFAULT_STAPLES.has(c))
      selectedCommodities = new Set(defaults.length > 0 ? defaults : commodities.slice(0, 6))
      commoditiesInitialised = true
    }
  })

  let chartData = $derived(() => {
    if (!selectedAdmin || selectedCommodities.size === 0) return new Map()
    const filtered = baseData.filter(d =>
      d[adminField] === selectedAdmin &&
      selectedCommodities.has(d.commodity)
    )
    return rollup(
      filtered,
      v => mean(v, d => d.usdprice),
      d => d.commodity,
      d => d.date
    )
  })

  let tableData = $derived(() => {
    const cd = chartData()
    if (cd.size === 0) return []
    return Array.from(cd, ([commodity, dateMap]) => {
      const sorted = Array.from(dateMap, ([date, value]) => ({ date, value }))
        .filter(d => d.value != null)
        .sort((a, b) => a.date - b.date)
      if (sorted.length === 0) return null
      const latest = sorted[sorted.length - 1]
      const prev = sorted.length > 1 ? sorted[sorted.length - 2] : null
      const peak = sorted.reduce((max, d) => d.value > max.value ? d : max, sorted[0])
      return {
        commodity,
        latestPrice: latest.value,
        prevPrice: prev?.value ?? null,
        change: prev != null ? latest.value - prev.value : null,
        peakPrice: peak.value,
        peakDate: peak.date,
      }
    }).filter(Boolean).sort((a, b) => b.latestPrice - a.latestPrice)
  })

  function toggleCommodity(c) {
    const next = new Set(selectedCommodities)
    if (next.has(c)) next.delete(c)
    else next.add(c)
    selectedCommodities = next
  }

  function selectDefault() {
    const defaults = commodities.filter(c => DEFAULT_STAPLES.has(c))
    selectedCommodities = new Set(defaults.length > 0 ? defaults : commodities.slice(0, 6))
  }

  $effect(() => {
    const cd = chartData()
    if (!chartEl) return

    // cleanup
    select(chartEl).selectAll('*').remove()
    document.querySelectorAll('.admin-tooltip').forEach(el => el.remove())

    if (cd.size === 0) return

    renderChart(chartEl, cd)

    return () => {
      select(chartEl).selectAll('*').remove()
      document.querySelectorAll('.admin-tooltip').forEach(el => el.remove())
    }
  })

  function renderChart(container, cd) {
    const margin = { top: 20, right: 140, bottom: 52, left: 60 }
    const totalWidth = container.clientWidth || 800
    const totalHeight = 420
    const iW = totalWidth - margin.left - margin.right
    const iH = totalHeight - margin.top - margin.bottom

    // Build flat series: commodity → [{date, value}]
    const series = Array.from(cd, ([commodity, dateMap]) => ({
      commodity,
      values: Array.from(dateMap, ([date, value]) => ({ date, value }))
        .sort((a, b) => a.date - b.date)
    }))

    // Scales
    const allDates = series.flatMap(s => s.values.map(v => v.date))
    const allValues = series.flatMap(s => s.values.map(v => v.value))
    const x = scaleTime().domain(extent(allDates)).range([0, iW])
    const y = scaleLinear().domain([0, Math.max(...allValues)]).nice().range([iH, 0])

    const svg = select(container)
      .append('svg')
      .attr('width', totalWidth)
      .attr('height', totalHeight)
      .style('overflow', 'visible')

    const g = svg.append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`)

    // Y grid lines
    g.append('g')
      .selectAll('line')
      .data(y.ticks(6))
      .join('line')
      .attr('stroke', '#e8e8e8')
      .attr('stroke-width', 1)
      .attr('x1', 0)
      .attr('x2', iW)
      .attr('y1', d => y(d))
      .attr('y2', d => y(d))

    // X axis
    const xAxisG = g.append('g').attr('transform', `translate(0,${iH})`)

    // base line
    xAxisG.append('line').attr('x2', iW).attr('stroke', '#ccc').attr('stroke-width', 1)

    // year labels — one per year, positioned at Jan 1
    const yearTicks = x.ticks(timeYear)
    xAxisG.selectAll('.year-label')
      .data(yearTicks)
      .join('text')
      .attr('class', 'axis-label')
      .attr('x', d => x(d))
      .attr('y', 16)
      .attr('text-anchor', 'middle')
      .style('font-size', '12px')
      .text(d => timeFormat('%Y')(d))

    // Y axis
    const yTickGs = g.selectAll('.y-tick')
      .data(y.ticks(6))
      .join('g')
      .attr('class', 'y-tick')
      .attr('transform', d => `translate(0,${y(d)})`)

    yTickGs.append('text')
      .attr('class', 'axis-label')
      .attr('x', -8)
      .attr('dy', '0.32em')
      .attr('text-anchor', 'end')
      .style('font-size', '12px')
      .text(d => `$${d >= 1000 ? (d / 1000).toFixed(1) + 'k' : d}`)

    // Y axis label
    g.append('text')
      .attr('class', 'axis-label')
      .attr('transform', `rotate(-90)`)
      .attr('x', -iH / 2)
      .attr('y', -46)
      .attr('text-anchor', 'middle')
      .style('font-size', '12px')
      .text('Price (USD)')

    // Lines
    const lineGen = line()
      .defined(d => d.value != null)
      .x(d => x(d.date))
      .y(d => y(d.value))

    const lineG = g.append('g').attr('class', 'lines')

    const LINE_GRAY  = 'var(--hdx-grey-mid)'
    const LABEL_GRAY = 'var(--hdx-grey-dark)'
    const GRAY = LABEL_GRAY
    const BLUE = 'var(--hdx-sapphire)'

    series.forEach((s) => {
      lineG.append('path')
        .datum(s.values)
        .attr('class', 'line')
        .style('stroke-width', '1.5px')
        .attr('stroke', LINE_GRAY)
        .attr('data-commodity', s.commodity)
        .attr('d', lineGen)

    })

    // Right-margin labels — all at same x, left-aligned
    const LABEL_X = iW + 10
    const MIN_LABEL_GAP = 16

    const labelData = series
      .map(s => {
        const last = s.values[s.values.length - 1]
        if (!last) return null
        return {
          commodity: s.commodity,
          anchorX: x(last.date),
          naturalY: y(last.value),
          resolvedY: y(last.value),
        }
      })
      .filter(Boolean)
      .sort((a, b) => a.naturalY - b.naturalY)

    // Forward pass: push labels down to resolve overlaps
    for (let i = 1; i < labelData.length; i++) {
      labelData[i].resolvedY = Math.max(
        labelData[i].resolvedY,
        labelData[i - 1].resolvedY + MIN_LABEL_GAP
      )
    }

    // Backward pass: pull labels up from the bottom to fix overflow
    labelData[labelData.length - 1].resolvedY = Math.min(
      labelData[labelData.length - 1].resolvedY, iH
    )
    for (let i = labelData.length - 2; i >= 0; i--) {
      labelData[i].resolvedY = Math.min(
        labelData[i].resolvedY,
        labelData[i + 1].resolvedY - MIN_LABEL_GAP
      )
    }

    const labelG = g.append('g').attr('class', 'label-group')
    labelData.forEach(({ commodity, anchorX, naturalY, resolvedY }) => {
      const displaced = Math.abs(resolvedY - naturalY) > 1
      const lineEndsEarly = anchorX < iW - 5


      if (displaced || lineEndsEarly) {
        labelG.append('line')
          .attr('class', 'annotation-line')
          .attr('data-commodity', commodity)
          .attr('x1', anchorX).attr('y1', naturalY)
          .attr('x2', iW + 2).attr('y2', resolvedY)
          .attr('stroke', LINE_GRAY)
          .attr('stroke-width', 0.75)
          .attr('stroke-dasharray', '2,2')
      }
      labelG.append('text')
        .attr('class', 'line-label')
        .attr('x', LABEL_X)
        .attr('y', resolvedY)
        .attr('dy', '0.32em')
        .attr('fill', GRAY)
        .attr('data-commodity', commodity)
        .text(commodity.length > 18 ? commodity.slice(0, 17) + '…' : commodity)
    })

    // Label + line hover: hovered turns blue, others fade
    const allLines = g.selectAll('.line')
    const allLabels = g.selectAll('.line-label')
    const allAnnotations = g.selectAll('.annotation-line')

    function highlight(commodity) {
      allLines.each(function() {
        const node = select(this)
        const match = node.attr('data-commodity') === commodity
        node.attr('stroke', match ? BLUE : LINE_GRAY)
          .style('opacity', match ? 1 : 0.3)
        if (match) node.raise()
      })
      allLabels.each(function() {
        const node = select(this)
        const match = node.attr('data-commodity') === commodity
        node.attr('fill', match ? BLUE : LABEL_GRAY)
          .style('opacity', match ? 1 : 0.3)
      })
      allAnnotations.each(function() {
        const node = select(this)
        const match = node.attr('data-commodity') === commodity
        node.attr('stroke', match ? BLUE : LINE_GRAY)
          .style('opacity', match ? 1 : 0.3)
      })
    }

    function resetHighlight() {
      allLines.attr('stroke', LINE_GRAY).style('opacity', 1)
      allLabels.attr('fill', LABEL_GRAY).style('opacity', 1)
      allAnnotations.attr('stroke', LINE_GRAY).style('opacity', 1)
    }

    allLabels.style('cursor', 'pointer')
      .on('mouseover', function() { highlight(select(this).attr('data-commodity')) })
      .on('mouseout', resetHighlight)

    allLines.style('cursor', 'pointer')
      .on('mouseover', function() { highlight(select(this).attr('data-commodity')) })
      .on('mouseout', resetHighlight)

    // Expose for sidebar hover
    highlightFn = highlight
    resetFn = resetHighlight

    // Tooltip
    const tooltip = select('body')
      .append('div')
      .attr('class', 'admin-tooltip')
      .style('position', 'fixed')
      .style('pointer-events', 'none')
      .style('background', 'white')
      .style('border', '1px solid var(--hdx-grey-mid)')
      .style('border-radius', '4px')
      .style('padding', '8px 10px')
      .style('font-size', '12px')
      .style('font-family', 'var(--font-family)')
      .style('color', 'var(--hdx-black)')
      .style('box-shadow', '0 2px 6px rgba(0,0,0,0.1)')
      .style('opacity', 0)

    // Vertical dashed line
    const vLine = g.append('line')
      .attr('class', 'v-line')
      .attr('y1', 0)
      .attr('y2', iH)
      .attr('stroke', 'var(--hdx-grey-mid)')
      .attr('stroke-width', 1)
      .attr('stroke-dasharray', '4,3')
      .style('opacity', 0)

    // Overlay rect for mouse events
    g.append('rect')
      .attr('width', iW)
      .attr('height', iH)
      .attr('fill', 'transparent')
      .on('mousemove', function(event) {
        const [mx] = pointer(event, this)
        const date = x.invert(mx)
        vLine.attr('x1', mx).attr('x2', mx).style('opacity', 1)

        // Find nearest values per series
        const rows = series.map(s => {
          const nearest = s.values.reduce((prev, curr) =>
            Math.abs(curr.date - date) < Math.abs(prev.date - date) ? curr : prev
          , s.values[0])
          return { commodity: s.commodity, value: nearest?.value }
        }).filter(r => r.value != null)
          .sort((a, b) => b.value - a.value)

        const fmt = timeFormat('%b %Y')
        const dateLabel = fmt(date)

        tooltip
          .style('opacity', 1)
          .html(
            `<div style="font-weight:700;margin-bottom:4px;color:var(--hdx-grey-dark)">${dateLabel}</div>` +
            rows.map(r =>
              `<div style="display:flex;justify-content:space-between;gap:16px">` +
              `<span>${r.commodity}</span>` +
              `<span style="font-weight:600">$${r.value.toFixed(2)}</span></div>`
            ).join('')
          )

        const [ex, ey] = pointer(event)
        const pageX = event.clientX
        const pageY = event.clientY
        tooltip
          .style('left', (pageX + 14) + 'px')
          .style('top', (pageY - 28) + 'px')
      })
      .on('mouseleave', function() {
        vLine.style('opacity', 0)
        tooltip.style('opacity', 0)
      })
  }
</script>

<div class="admin-view">
  <aside class="admin-sidebar">
    <div class="sidebar-inner">
    <h4 class="sidebar-heading">Commodities</h4>
    <div class="link-btns">
      <button class="link-btn" onclick={selectDefault}>Reset</button>
    </div>
    <div class="checkbox-list">
      {#each commodities as c}
        <label
          class="checkbox-item"
          onmouseenter={() => selectedCommodities.has(c) && highlightFn?.(c)}
          onmouseleave={() => resetFn?.()}
        >
          <input
            type="checkbox"
            checked={selectedCommodities.has(c)}
            onchange={() => toggleCommodity(c)}
          />
          {c}
        </label>
      {/each}
    </div>
    </div>
  </aside>

  <div class="admin-main">
    <div class="admin-controls">
      <span class="controls-label">Food prices for</span>
      <select
        id="admin-select"
        value={selectedAdmin}
        onchange={e => selectedAdmin = e.target.value}
      >
        {#each admins as a}
          <option value={a}>{a}</option>
        {/each}
      </select>
    </div>

    <div class="chart" bind:this={chartEl}></div>

    {#if tableData().length > 0}
    <table class="price-table">
      <thead>
        <tr>
          <th>Commodity</th>
          <th class="num-col">Latest price</th>
          <th class="num-col">Prev month</th>
          <th class="num-col">Monthly change</th>
          <th class="num-col">Peak price (since 2016)</th>
        </tr>
      </thead>
      <tbody>
        {#each tableData() as row}
        <tr>
          <td>{row.commodity}</td>
          <td class="num-col">${row.latestPrice.toFixed(2)}</td>
          <td class="num-col">{row.prevPrice != null ? '$' + row.prevPrice.toFixed(2) : '—'}</td>
          <td class="num-col change" class:up={row.change > 0} class:down={row.change < 0}>
            {#if row.change != null}
              {row.change > 0 ? '+' : ''}{row.change.toFixed(2)}
            {:else}—{/if}
          </td>
          <td class="num-col">${row.peakPrice.toFixed(2)} <span class="peak-date">({timeFormat('%b %Y')(row.peakDate)})</span></td>
        </tr>
        {/each}
      </tbody>
    </table>
    {/if}

    <p class="chart-source"><a class="data-label" href="https://data.humdata.org/dataset/wfp-food-prices-for-lebanon" target="_blank" rel="noopener">DATA</a><span class="data-source"> | Dec 2025 | HDX</span></p>
  </div>
</div>

<style>
  .admin-view {
    display: grid;
    grid-template-columns: 200px 1fr;
    gap: 1.5rem;
    align-items: stretch;
  }

  .admin-sidebar {
    position: relative;
    font-size: var(--font-size-base);
  }

  .sidebar-inner {
    position: sticky;
    top: 1rem;
  }

  .sidebar-heading {
    font-size: 15px;
    font-weight: 600;
    color: var(--hdx-black);
    margin: 0 0 0.25rem;
  }

  .link-btns {
    display: flex;
    align-items: center;
    gap: 0.25rem;
    margin-bottom: 0.5rem;
  }

  .link-btn {
    background: none;
    border: none;
    padding: 0;
    cursor: pointer;
    color: var(--hdx-sapphire);
    font-size: var(--font-size-sm);
    font-family: var(--font-family);
    text-decoration: underline;
  }

  .link-btn:hover {
    color: var(--hdx-grey-dark);
  }

  .checkbox-list {
    max-height: 60vh;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 0.2rem;
  }

  .checkbox-item {
    display: flex;
    align-items: center;
    gap: 0.35rem;
    font-size: var(--font-size-base);
    cursor: pointer;
    line-height: 1.4;
  }

  .admin-controls {
    display: flex;
    align-items: baseline;
    gap: 0.5rem;
    margin-bottom: 0.75rem;
  }

  .controls-label {
    font-size: 15px;
    font-weight: 600;
    color: var(--hdx-black);
  }

  .admin-controls select {
    font-family: var(--font-family);
    font-size: var(--font-size-base);
    font-weight: 600;
    color: var(--hdx-black);
    border: none;
    border-bottom: 2px solid var(--hdx-grey-mid);
    background: transparent;
    cursor: pointer;
    padding: 0 4px 1px;
    appearance: auto;
  }

  .admin-controls select:focus {
    outline: none;
    border-bottom-color: var(--hdx-sapphire);
  }

  .admin-main {
    min-width: 0;
  }

  :global(.line-label) {
    font-size: 13px;
  }

  .price-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 1.5rem;
    font-size: 14px;
    font-family: var(--font-family);
  }

  .price-table th {
    background-color: #EDF6FD;
    border-bottom: 1px solid var(--hdx-grey-mid);
    font-weight: 700;
    font-size: 14px;
    color: var(--hdx-black);
    padding: 12px 10px;
    text-align: left;
  }

  .price-table td {
    border-bottom: 1px solid var(--hdx-grey-light);
    font-size: 14px;
    padding: 5px 10px;
    color: var(--hdx-black);
    vertical-align: middle;
  }

  .price-table .num-col {
    text-align: right;
    font-variant-numeric: tabular-nums;
  }

  .change { font-weight: 600; }
  .change.up   { color: var(--hdx-tomato); }
  .change.down { color: var(--hdx-mint); }

  .peak-date {
    color: var(--hdx-grey-dark);
    font-weight: 400;
  }

  :global(.chart-source) {
    margin-top: 2rem;
  }
</style>
