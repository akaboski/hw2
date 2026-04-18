<script>
  import * as d3 from 'd3';
  import { onMount } from 'svelte';

  export let fullData = [];
  export let filter1 = [];
  export let filter2 = [];
  export let var1;
  export let var2;

  const basemapUrl = new URL('./chicago-community-areas.geojson', import.meta.url).href;
  const width = 760;
  const height = 760;
  const padding = 24;
  const swatchWidth = 42;
  const colorRange = ['#eff3ff', '#bdd7e7', '#6baed6', '#3182bd', '#08519c'];

  let basemap = null;
  let projection = d3.geoMercator();
  let path = d3.geoPath(projection);

  function hasExtent(extent) {
    return Array.isArray(extent) && extent.length === 2;
  }

  function inExtent(value, extent) {
    return Number.isFinite(value) && value >= extent[0] && value < extent[1];
  }

  function passesFilters(row) {
    if (hasExtent(filter1) && !inExtent(row[var1], filter1)) return false;
    if (hasExtent(filter2) && !inExtent(row[var2], filter2)) return false;
    return true;
  }

  function normalizeArea(value) {
    const match = String(value ?? '').match(/\d+/);
    if (!match) return null;
    const area = +match[0];
    return Number.isFinite(area) && area >= 1 && area <= 77 ? area : null;
  }

  function formatCommunityName(name) {
    return String(name ?? 'Unknown')
      .toLowerCase()
      .replace(/\b\w/g, (letter) => letter.toUpperCase());
  }

  function summarizeAreas(rows) {
    const byArea = new Map();
    let filteredMappedRows = 0;
    let filteredUnmappedRows = 0;

    for (const row of rows) {
      const isMatch = passesFilters(row);
      const area = normalizeArea(row.communityArea);

      if (area == null) {
        if (isMatch) filteredUnmappedRows += 1;
        continue;
      }

      let stats = byArea.get(area);
      if (!stats) {
        stats = {
          area,
          totalCount: 0,
          filteredCount: 0,
          violationSum: 0,
          riskSum: 0,
          riskCount: 0
        };
        byArea.set(area, stats);
      }

      stats.totalCount += 1;

      if (isMatch) {
        filteredMappedRows += 1;
        stats.filteredCount += 1;
        stats.violationSum += Number.isFinite(row.violationCount) ? row.violationCount : 0;

        if (Number.isFinite(row.riskScore)) {
          stats.riskSum += row.riskScore;
          stats.riskCount += 1;
        }
      }
    }

    return { byArea, filteredMappedRows, filteredUnmappedRows };
  }

  function formatLegendLabel(start, end) {
    const a = Math.ceil(start);
    const b = Math.floor(end);
    return a >= b ? `${b}` : `${a}–${b}`;
  }

  onMount(async () => {
    basemap = await d3.json(basemapUrl);
  });

  $: if (basemap) {
    projection = d3.geoMercator().fitExtent(
      [
        [padding, padding],
        [width - padding, height - padding]
      ],
      basemap
    );
    path = d3.geoPath(projection);
  }

  $: summary = summarizeAreas(fullData);

  $: communityAreas = basemap
    ? basemap.features.map((feature) => {
        const props = feature.properties ?? {};
        const area = normalizeArea(props.area_num_1 ?? props.area_numbe ?? props.area ?? props.comarea);
        const stats = summary.byArea.get(area) ?? {
          totalCount: 0,
          filteredCount: 0,
          violationSum: 0,
          riskSum: 0,
          riskCount: 0
        };

        return {
          feature,
          area,
          name: formatCommunityName(props.community),
          totalCount: stats.totalCount,
          filteredCount: stats.filteredCount,
          averageViolations: stats.filteredCount ? stats.violationSum / stats.filteredCount : 0,
          averageRisk: stats.riskCount ? stats.riskSum / stats.riskCount : null,
          tooltip: `${formatCommunityName(props.community)} (Area ${area})\nMatching inspections: ${d3.format(',d')(stats.filteredCount)}\nAll inspections: ${d3.format(',d')(stats.totalCount)}\nAvg. violations: ${d3.format('.1f')(stats.filteredCount ? stats.violationSum / stats.filteredCount : 0)}${stats.riskCount ? `\nAvg. risk score: ${d3.format('.1f')(stats.riskSum / stats.riskCount)}` : ''}`
        };
      })
    : [];

  $: maxCount = d3.max(communityAreas, (d) => d.filteredCount) ?? 0;
  $: colorBins = colorRange.slice(0, Math.max(1, Math.min(colorRange.length, maxCount || 1)));
  $: colorScale =
    maxCount > 0
      ? d3.scaleQuantize().domain([1, maxCount]).range(colorBins)
      : null;

  $: legendItems =
    colorScale && maxCount > 0
      ? colorScale.range().map((color) => {
          const [start, end] = colorScale.invertExtent(color);
          return {
            color,
            label: formatLegendLabel(start, end)
          };
        })
      : [];

  $: visibleAreaCount = communityAreas.filter((d) => d.filteredCount > 0).length;

  function fillFor(area) {
    if (area.filteredCount <= 0 || !colorScale) return '#f4f4f4';
    return colorScale(area.filteredCount);
  }
</script>

<main>
  <svg {width} {height} viewBox={`0 0 ${width} ${height}`} aria-label="Chicago community area choropleth">
    <g>
      {#each communityAreas as area}
        <path
          d={path(area.feature)}
          class="community-area"
          fill={fillFor(area)}
          stroke="#ffffff"
          stroke-width="1"
>
          <title>{area.tooltip}</title>
        </path>
      {/each}
    </g>

    <g transform={`translate(${width - 270}, ${height - 112})`}>
      <rect class="legend-box" width="230" height="88" rx="8" ry="8" />
      <text class="legend-title" x="14" y="22">Matching inspections</text>

      {#if legendItems.length > 0}
        {#each legendItems as item, index}
          <rect
            x={14 + index * swatchWidth}
            y="34"
            width={swatchWidth - 6}
            height="16"
            fill={item.color}
          />
          <text class="legend-label" x={14 + index * swatchWidth} y="68">{item.label}</text>
        {/each}
      {:else}
        <text class="legend-label" x="14" y="48">No matching inspections</text>
      {/if}
    </g>
  </svg>

  <p class="caption">
    Showing <strong>{d3.format(',d')(summary.filteredMappedRows)}</strong> matching inspections across
    <strong>{visibleAreaCount}</strong> community areas.
    {#if summary.filteredUnmappedRows > 0}
      <span>
        {d3.format(',d')(summary.filteredUnmappedRows)} matching row{summary.filteredUnmappedRows === 1 ? '' : 's'}
        could not be assigned to a community area.
      </span>
    {/if}
  </p>
</main>

<style>
  main {
    max-width: 760px;
  }

  svg {
    display: block;
    width: 100%;
    height: auto;
    background: #fbfbfb;
    border: 1px solid #e1e1e1;
  }

  .community-area {
    transition: fill 120ms ease, stroke 120ms ease;
  }

  .community-area:hover {
    stroke: #111111;
    stroke-width: 1.4;
  }

  .legend-box {
    fill: rgba(255, 255, 255, 0.92);
    stroke: #d0d0d0;
  }

  .legend-title {
    font-size: 13px;
    font-weight: 600;
  }

  .legend-label {
    font-size: 11px;
    fill: #444444;
  }

  .caption {
    margin: 0.6rem 0 0 0;
    line-height: 1.4;
    color: #444444;
  }
</style>
