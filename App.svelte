<script>
  import * as d3 from 'd3';
  import { onMount } from 'svelte';
  import Map from './Map.svelte';
  import Histogram from './Histogram.svelte';

  const csvUrl = new URL('./example.csv', import.meta.url).href;

  let data = [];
  let fullData = [];
  let filter1 = [];
  let filter2 = [];

  let var1 = 'resultScore';
  let var2 = 'violationCount';

  const resultScoreMap = {
    'Pass': 0,
    'Pass w/ Conditions': 1,
    'Fail': 2,
    'Out of Business': 3,
    'Business Not Located': 4,
    'No Entry': 5,
    'Not Ready': 6
  };

  function parseRiskScore(risk) {
    const match = String(risk ?? '').match(/Risk\s+(\d+)/i);
    return match ? +match[1] : null;
  }

  function countViolations(violations) {
    if (!violations) return 0;
    return violations
      .split('|')
      .map((d) => d.trim())
      .filter(Boolean).length;
  }

  onMount(async function () {
    const table = await d3.csv(csvUrl, (d) => {
      const latitude = +d['Latitude'];
      const longitude = +d['Longitude'];
      const violationCount = countViolations(d['Violations']);
      const resultScore = resultScoreMap[d['Results']] ?? -1;
      const riskScore = parseRiskScore(d['Risk']);

      if (!Number.isFinite(latitude) || !Number.isFinite(longitude)) {
        return null;
      }

      return {
        type: 'Feature',
        geometry: {
          type: 'Point',
          coordinates: [longitude, latitude]
        },
        properties: {
          name10: d['DBA Name'] || d['AKA Name'] || 'Unknown location',
          population: violationCount,
          walkability: riskScore,
          percentWhite: resultScore,
          percentVacant: violationCount,
          inspectionId: +d['Inspection ID'],
          license: +d['License #'],
          zip: +d['Zip'],
          facilityType: d['Facility Type'] || 'Unknown',
          risk: d['Risk'] || 'Unknown',
          riskScore,
          results: d['Results'] || 'Unknown',
          resultScore,
          violationCount,
          inspectionType: d['Inspection Type'] || 'Unknown',
          inspectionDate: d['Inspection Date'],
          address: d['Address'] || '',
          city: d['City'] || '',
          state: d['State'] || ''
        }
      };
    });

    data = table.filter(Boolean);
    fullData = [...data];
  });

  function updateData() {
    if (filter1.length > 0 && filter2.length > 0) {
      data = fullData.filter(
        (d) =>
          d.properties[var1] >= filter1[0] &&
          d.properties[var1] < filter1[1] &&
          d.properties[var2] >= filter2[0] &&
          d.properties[var2] < filter2[1]
      );
    } else if (filter1.length > 0 && filter2.length == 0) {
      data = fullData.filter(
        (d) => d.properties[var1] >= filter1[0] && d.properties[var1] < filter1[1]
      );
    } else if (filter1.length == 0 && filter2.length > 0) {
      data = fullData.filter(
        (d) => d.properties[var2] >= filter2[0] && d.properties[var2] < filter2[1]
      );
    } else {
      data = [...fullData];
    }
  }
</script>

<main>
  <h1>Chicago Food Inspection Sample</h1>
  <p>
    The map now uses points from <code>example.csv</code>. Point color reflects violation count,
    and the histograms filter by inspection result score and violation count.
  </p>

  <div class="flex-container row">
    <div class="map">
      <h3>Locations (colored by violation count)</h3>
      <Map data={data} fullData={fullData}></Map>
    </div>
    <div class="flex-container col">
      <div class="hist">
        <h3>Inspection result score</h3>
        <Histogram
          data={data}
          fullData={fullData}
          variable={var1}
          bind:filter={filter1}
          update={updateData}
        ></Histogram>
      </div>
      <div class="hist">
        <h3>Violation count</h3>
        <Histogram
          data={data}
          fullData={fullData}
          variable={var2}
          bind:filter={filter2}
          update={updateData}
        ></Histogram>
      </div>
    </div>
  </div>
</main>

<style>
  .flex-container {
    display: flex;
    justify-content: center;
    height: 100%;
    padding: 15px;
    gap: 5px;
  }

  .flex-container > div {
    padding: 8px;
  }

  .flex-container.row {
    flex-direction: row;
  }

  .flex-container.col {
    flex-direction: column;
  }

  .map {
    flex-grow: 1;
  }

  .hist {
    flex-grow: 0;
  }

  h1,
  h3,
  p {
    margin-left: 1rem;
  }

  p {
    max-width: 70rem;
  }
</style>
