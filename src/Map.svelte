<script>
    import * as d3 from 'd3';
    import {legendColor} from 'd3-svg-legend';

    export let data;
    export let fullData;
    export let filter;
    export let update;

    let width = 1000;
    let height = 500;

    let proj = d3.geoMercator()
        .scale(800)
        .center([-55, 24])
        .translate([width, height]);
    let path = d3.geoPath().projection(proj);

    const brush = d3.brush().extent([[0, 0], [width, height]]).on("brush", brushed).on("end", brushended);

    function brushed(event) {
        if (!event.selection) return;

        const [[x0, y0], [x1, y1]] = event.selection;

        filter = fullData
            .filter((d) => {
            const [x, y] = path.centroid(d);

            return (
                x >= x0 &&
                x <= x1 &&
                y >= y0 &&
                y <= y1
            );
            })
            .map((d) => d.properties.GEOID);

        update();
    }

    function brushended(event) {
        if (!event.selection) {
            filter = [];
            update();
        }
    }

    // $: scale = d3.scaleOrdinal(d3.schemeDark2)
    //   .domain(d3.extent(data.map((d) => +d.properties.name10)));
    // $: scale = d3.scaleSequential(d3.interpolateGreens)
    //   .domain(d3.extent(data.map((d) => +d.properties.walkability)));
    $: scale = d3.scaleSequential(d3.interpolateYlOrRd)
        .domain([d3.min(data.map((d) => +d.properties.riskScore)), d3.median(data.map((d) => +d.properties.riskScore)), d3.max(data.map((d) => +d.properties.riskScore))]);

    let legend;
    let brushLayer;
    $: {
    if (legend) {
            const colorLegend = legendColor()
            .shape('rect')
            .cells(9)
            .scale(scale);

            d3.select(legend)
            .call(colorLegend);
        }

        if (brushLayer) {
            d3.select(brushLayer)
            .call(brush);
        }
    }
</script>

<main>
    <svg {width} {height}>
        <text x={width / 2} y={25} text-anchor="middle" class="map-title">
            National Risk Index by County
        </text>

        {#each data as d}
          <path style = "fill: {scale(+d.properties.riskScore)};"
            d={path(d)}/>
        {/each}
        {#each fullData as d}
          <path class = "geooverlay"
            d={path(d)}/>
        {/each}

        <g transform="translate({width - 100}, 50)"
              bind:this={legend} />
        
        <g bind:this={brushLayer} />
      </svg>
</main>

<style>
    .geooverlay {
      stroke: grey;
      stroke-width: 1px;
      fill: none;
    }
  </style>