# viz-foodprices — Lebanon Food Prices

Data visualization of WFP retail food prices in Lebanon

## Stack

- **Svelte 5** — UI framework. Uses runes syntax: `$state`, `$derived`, `$props`, `$effect`
- **D3 v7** — scales, data aggregation, and chart rendering
- **Vite** — dev server and build tool

## Running locally

```bash
npm install
npm run dev      # http://localhost:5173
npm run build    # outputs to dist/
```

## Data

**File:** `public/data/wfp_food_prices_lbn.csv`
**Source:** WFP VAM Food Price Monitoring
**Coverage:** Aug 2012 – Dec 2025 (~25,862 rows)

Each row represents a food price observation with these key fields:

| Field | Description |
|---|---|
| `date` | Observation date |
| `admin1` | Governorate (e.g. Beirut, Akkar) |
| `admin2` | District |
| `market` | Market name |
| `commodity` | Food item (e.g. Bread (pita)) |
| `category` | Food category (e.g. cereals, vegetables) |
| `unit` | Unit of measurement (e.g. kg) |
| `price` | Price in Lebanese Pounds (LBP) |
| `usdprice` | Price in USD |

## File structure

```
src/
  App.svelte              # Root component
  main.js                 # Entry point — mounts App, imports global CSS
  style.css               # CSS classes
  components/
    AdminView.svelte       # "By Admin" view — sidebar + multi-line chart + table
public/data/
  wfp_food_prices_lbn.csv
```

## Components

### `App.svelte`
The root. Fetches the CSV on mount using `d3.csvParse` with `autoType` (which parses dates and numbers automatically). Passes the full dataset down as a `data` prop. Reads `?view=` from the URL to decide which view to render.

### `AdminView.svelte`
The main view. Two-column layout: sidebar on the left, chart + table on the right.

**Sidebar:** Checkbox list of all commodities (currently only showing food commodities but could also include non-food like fuels and exchange rate). Checking/unchecking re-renders the chart. "Reset" button restores the 6 default staples (hardcoded in `AdminView.svelte`).

**Chart:** Multi-line D3 chart of mean USD prices over time for the selected admin1 area and commodities. Rendered imperatively inside a `$effect` using D3. Lines are grey by default; hovering a line or label highlights it in blue and fades others. A tooltip shows all commodity prices at the hovered date.

**Table:** Summary table below the chart showing latest price, previous month price, monthly change, and peak price for each selected commodity.

**Key props:**
- `data` — full dataset from App
- `adminField` — which field to use as the region selector (`"admin1"` or `"admin2"`). Change this one prop in `App.svelte` to switch between adm1 and adm2 level.
