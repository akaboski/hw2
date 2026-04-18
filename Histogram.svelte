<script>
  import * as d3 from 'd3';

  export let fullData = [];
  export let variable;
  export let var1;
  export let var2;
  export let filter1 = [];
  export let filter2 = [];
  export let filter = [];

  const margin = { top: 10, right: 18, bottom: 32, left: 42 };
  const width = 360;
  const height = 200;
  const chartW = width - margin.left - margin.right;
  const chartH = height - margin.top - margin.bottom;
  const BIN_COUNT = 30;

  let brushLayer;
  let xAxis;
  let yAxis;

  function hasExtent(extent) {
    return Array.isArray(extent) && extent.length === 2;
  }

  function inExtent(value, extent) {
    return value >= extent[0] && value < extent[1];
  }

  function passesFilters(row) {
    if (hasExtent(filter1) && !inExtent(row[var1], filter1)) return false;
    if (hasExtent(filter2) && !inExtent(row[var2], filter2)) return false;
    return true;
  }

  function computeHistogram(rows, key) {
    const values = [];
    for (const row of rows) {
      const value = +row[key];
      if (Number.isFinite(value)) values.push(value);
    }

    if (values.length === 0) {
      return {
        xDomain: [0, 1],
        backgroundBins: [],
        activeBins: [],
        yMax: 1
      };
    }

    let [min, max] = d3.extent(values);
    if (min === max) {
      min -= 0.5;
      max += 0.5;
    }

    const step = (max - min) / BIN_COUNT || 1;
    const backgroundCounts = new Array(BIN_COUNT).fill(0);
    const activeCounts = new Array(BIN_COUNT).fill(0);

    for (const row of rows) {
      const value = +row[key];
      if (!Number.isFinite(value)) continue;

      let index = Math.floor((value - min) / step);
      if (value === max) index = BIN_COUNT - 1;
      index = Math.max(0, Math.min(BIN_COUNT - 1, index));

      backgroundCounts[index] += 1;
      if (passesFilters(row)) {
        activeCounts[index] += 1;
      }
    }

    const backgroundBins = backgroundCounts.map((count, i) => ({
      x0: min + i * step,
      x1: min + (i + 1) * step,
      length: count
    }));

    const activeBins = activeCounts.map((count, i) => ({
      x0: min + i * step,
      x1: min + (i + 1) * step,
      length: count
    }));

    return {
      xDomain: [min, max],
      backgroundBins,
      activeBins,
      yMax: d3.max(backgroundCounts) ?? 1
    };
  }

  $: histogram = computeHistogram(fullData, variable);
  $: xScale = d3.scaleLinear().range([0, chartW]).domain(histogram.xDomain);
  $: yScale = d3.scaleLinear().range([chartH, 0]).domain([0, histogram.yMax]).nice();

  $: brush = d3
    .brushX()
    .extent([
      [0, 0],
      [chartW, chartH]
    ])
    .on('end', brushended);

  function brushended(event) {
    if (event?.selection) {
      filter = [xScale.invert(event.selection[0]), xScale.invert(event.selection[1])];
    } else {
      filter = [];
    }
  }

  $: if (brushLayer && xAxis && yAxis) {
    d3.select(brushLayer).call(brush);
    d3.select(xAxis).call(d3.axisBottom(xScale).ticks(6));
    d3.select(yAxis).call(d3.axisLeft(yScale).ticks(5));
  }
</script>

<main>
  <svg {width} {height} aria-label={`Histogram for ${variable}`}>
    <g transform={`translate(${margin.left}, ${margin.top})`}>
      {#each histogram.backgroundBins as d}
        <rect
          class="backgroundbar"
          x={xScale(d.x0)}
          y={yScale(d.length)}
          width={Math.max(0, xScale(d.x1) - xScale(d.x0) - 1)}
          height={chartH - yScale(d.length)}
        />
      {/each}

      {#each histogram.activeBins as d}
        <rect
          class="bar"
          x={xScale(d.x0)}
          y={yScale(d.length)}
          width={Math.max(0, xScale(d.x1) - xScale(d.x0) - 1)}
          height={chartH - yScale(d.length)}
        />
      {/each}
    </g>

    <g transform={`translate(${margin.left}, ${margin.top})`} bind:this={brushLayer} />
    <g transform={`translate(${margin.left}, ${chartH + margin.top})`} bind:this={xAxis} />
    <g transform={`translate(${margin.left}, ${margin.top})`} bind:this={yAxis} />
  </svg>
</main>

<style>
  .bar {
    fill: goldenrod;
    stroke: white;
    stroke-width: 1px;
  }

  .backgroundbar {
    fill: #c9c9c9;
  }
</style>
