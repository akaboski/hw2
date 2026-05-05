<script>
  import * as d3 from 'd3';
	import {onMount} from 'svelte';
  import Map from './Map.svelte';
  import Histogram from './Histogram.svelte';

	let data = [];
  let fullData = [];
  let filter1 = [];
  let filter2 = [];
  let filter3 = [];
  let filterMap = [];

  let var1 = 'soviScore';
  let var2 = 'reslScore';
  let var3 = 'ealScore';

	onMount(async function() {
    // load data from csv (source: https://chicagohealthatlas.org/download)
      const nriColumn = 'RISK_SCORE'; // change this to any numeric NRI column you want to map

  let table = d3.csv('NRI_Table_Counties.csv', (d) => ({
    ...d,
    STCOFIPS: String(d.STCOFIPS).padStart(5, '0'),
    POPULATION: +d.POPULATION,
    RISK_SCORE: +d.RISK_SCORE,
    RISK_VALUE: +d.RISK_VALUE,
    SOVI_SCORE: +d.SOVI_SCORE,
    RESL_SCORE: +d.RESL_SCORE,
    EAL_SCORE: +d.EAL_SCORE
  }));

  let geocoord = d3.json('counties.geojson')
    .then((d) => d.features);

  await Promise.all([table, geocoord]).then((values) => {
    let table = values[0];
    let geocoord = values[1];

    const nriByFips = new globalThis.Map(
      table.map((row) => [row.STCOFIPS, row])
    );

    data = geocoord
      .map((county) => {
        const fips = county.properties.GEOID;
        const row = nriByFips.get(fips);

        if (!row || isNaN(row[nriColumn])) {
          return null;
        }

        return {
          ...county,
          properties: {
            ...county.properties,
            nriValue: row[nriColumn],
            population: row.POPULATION,
            riskScore: row.RISK_SCORE,
            riskValue: row.RISK_VALUE,
            soviScore: row.SOVI_SCORE,
            reslScore: row.RESL_SCORE,
            ealScore: row.EAL_SCORE,
            countyName: row.COUNTY,
            state: row.STATE,
            stateAbbr: row.STATEABBRV
          }
        };
      })
      .filter(Boolean);

    fullData = [...data];
  });
	});

  function updateData() {
    data = fullData.filter((d) => {
      const passesFilter1 =
        filter1.length === 0 ||
        (d.properties[var1] >= filter1[0] && d.properties[var1] < filter1[1]);

      const passesFilter2 =
        filter2.length === 0 ||
        (d.properties[var2] >= filter2[0] && d.properties[var2] < filter2[1]);

      const passesFilter3 =
        filter3.length === 0 ||
        (d.properties[var3] >= filter3[0] && d.properties[var3] < filter3[1]);

      const passesMapFilter =
        filterMap.length === 0 ||
        filterMap.includes(d.properties.GEOID);

      return passesFilter1 && passesFilter2 && passesFilter3 && passesMapFilter;
    });
  }
</script>

<main>
  <h1>Chicago Census Data</h1>

  <div class="flex-container row">
    <div class="map"><Map data={data} fullData={fullData} bind:filter={filterMap} update={updateData}></Map></div>
    <div class="flex-container col">
      <div class="hist"><Histogram title="Social Vulnerability Score" data={data} fullData={fullData} variable={var1} bind:filter={filter1} update={updateData}></Histogram></div>
      <div class="hist"><Histogram title="National Risk Index" data={data} fullData={fullData} variable={var2} bind:filter={filter2} update={updateData}></Histogram></div>
      <div class="hist"><Histogram title="Expected Annual Loss Score" data={data} fullData={fullData} variable={var3} bind:filter={filter3} update={updateData}></Histogram></div>
    </div>
  </div>
</main>

<style>
  .flex-container {

    display: flex;
    
    justify-content: center;  
    /* flex-flow: row; */ 
    

    height: 100%;
    padding: 15px;
    gap: 5px;

  }

  .flex-container > div{
    padding: 8px;
  }

  .flex-container .row {
    flex-direction: row;  
  }

  .flex-container .col {
    flex-direction: column;  
  }

  .map { 
    /* flex:1 1 auto; */
    flex-grow:1;
  }
			
  .hist { 
    /* flex:0 1 auto; */
    flex-grow:0;
  }
			

</style>