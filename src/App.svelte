<script lang="ts">
import { writable, derived } from 'svelte/store';
import {csvParse} from 'd3-dsv';
import { csv } from './assets/data';

const sortFunction = (sortBy: string, sortOrder: string) => {
  return (a: any, b: any) => {
    if (typeof a[sortBy] == 'string') {
      if (sortOrder === 'asc') return a[sortBy].localeCompare(b[sortBy]);
      else return b[sortBy].localeCompare(a[sortBy]);
    }
    else {
      if (sortOrder === 'asc') return a[sortBy] - b[sortBy];
      else return b[sortBy] - a[sortBy];
    }
  }
}

const data = csvParse(csv, (d: any) => {
  console.log('parsing');
  return {
    fylke: d['Fylke'],
    knr: +d['KNR'],
    kommune: d['Kommune'],
    skatt: +d['Kr skatt per innbygger etter inntektsutjevning'],
    prosentSkatt: +d['Prosent skatt av landsgjennomsnittet etter inntektsutjevning (fraksjon)'],
    ufoere: +d['Andel uføretrygda 18-66 år (prosent)'],
    nettoDrift: +d['Netto driftsresultat (beløp i 1000)'],
    omsorgssektor: +d['Andel utgifter til omsorgssektor av netto driftsutgifter (prosent)'],
    barnefattigdom: +d['Andel personer i husholdninger i vedvarende barnefattigdom 0-17 år (2020-2022)'],
    kommunaleAvgifter: d['Kommunale avgifter (gj.snitt)'],
    eiendomsskatt: d['Eiendomsskatt for bolig 2023 (Gj.snitt)']
  }
});

const config = writable({
  length: 10,
  offset: 0,
  sortBy: 'fylke',
  sortOrder: 'asc',
  filter: '',
});

const listItems = derived(config, $config => {
  const filteredItems = data
    .filter((d: any) => d.kommune.toLowerCase().includes($config.filter.toLowerCase()))
    .sort(sortFunction($config.sortBy, $config.sortOrder));
  if ($config.offset >= filteredItems.length) $config.offset = Math.min(filteredItems.length - $config.length, 0);
  return filteredItems;
});

const columns = [
  {
    key: 'kommune', 
    label: 'Kommune', 
    sortable: true
  },
  {
    key: 'fylke', 
    label: 'Fylke', 
    sortable: true
  },
  {
    key: 'skatt', 
    label: 'Skatt', 
    sortable: true,
    align: 'right',
    render: (value: number) => value.toLocaleString('nb-NO')
  },
  {
    key: 'prosentSkatt', 
    label: 'Skatt av landsgjennomsnittet', 
    sortable: true,
    render: (value: number) => value.toLocaleString('nb-NO', {style: 'percent', maximumFractionDigits: 1})
  },
  {
    key: 'ufoere', 
    label: 'Andel uføretrygda 18-66 år',
    sortable: true,
    render: (value: number) => (value / 100).toLocaleString('nb-NO', {style: 'percent', maximumFractionDigits: 1})
  },
  {
    key: 'nettoDrift', 
    label: 'Netto driftsresultat', 
    sortable: true,
    align: 'right',
    render: (value: number) => value.toLocaleString('nb-NO')
  },
  {
    key: 'omsorgssektor', 
    label: 'Utgifter til omsorgssektor', 
    description: 'Andel av netto driftsutgifter',
    sortable: true,
    render: (value: number) => (value / 100).toLocaleString('nb-NO', {style: 'percent', maximumFractionDigits: 1})
  },
  {
    key: 'barnefattigdom', 
    label: 'Barnefattigdom',
    description: 'Andel personer i husholdninger i vedvarende barnefattigdom 0-17 år', 
    sortable: true,
    render: (value: number) => (value / 100).toLocaleString('nb-NO', {style: 'percent', maximumFractionDigits: 1})
  },
  {
    key: 'kommunaleAvgifter', 
    label: 'Kommunale avgifter', 
    sortable: true,
    align: 'right',
    render: (value: number) => value
  },
  {
    key: 'eiendomsskatt', 
    label: 'Eiendomsskatt for bolig 2023', 
    sortable: true,
    align: 'right',
  }
]

const sort = (key: string) => {
  if ($config.sortBy == key)
    $config.sortOrder = $config.sortOrder === 'asc' ? 'desc' : 'asc';
  else {
    $config.sortOrder = 'asc';
    $config.sortBy = key;
  }
}

$: console.log($config);
$: console.log($listItems);

</script>

<div class="kostra-app">
  <div>
    <input type="text" bind:value={$config.filter} placeholder="Søk på kommune" on:change={() => $config.offset = 0}>
  </div>
  <div class="list">
    <table>
    <tr class="list-header">
      <th class="column"></th>
      {#each columns as column}
        <th class="column" class:right={column?.align == 'right'}>
          <div>
            <button on:click={() => sort(column.key)}>{#if $config.sortBy == column.key}{@html $config.sortOrder == 'asc' ? '&darr;' : '&uarr;'}{/if}{column.label}</button>
          </div>
          {#if column.description}<div class="header-description">{column.description}</div>{/if}
        </th>
      {/each}
    </tr>
    {#each $listItems.slice($config.offset, $config.offset + $config.length) as municipality}
      <tr class="list-item">
        <td class="shield">
          <img src="https://vaapenskiold.vercel.app/svg/{String(municipality.knr).padStart(4, '0')}.svg" alt="">
        </td>
        {#each columns as column}
          <td class:right={column?.align == 'right'}>
            {column.render ? column.render(municipality[column.key]) : municipality[column.key]}
          </td>
        {/each}
      </tr>
    {/each}
    </table>
  </div>
  <div class="pagination">
    <div>
      <button on:click={() => $config.offset -= $config.length} disabled={$config.offset <= 10}>Første</button>
      <button on:click={() => $config.offset -= $config.length} disabled={$config.offset == 0}>Forrige</button>
    </div>
    <button on:click={() => $config.offset += $config.length} disabled={$config.offset + $config.length >= $listItems.length}>Neste
  </div>
</div>

<style>
  .kostra-app {
    display: flex;
    flex-direction: column;
    gap: 1rem;
    font-family: "National 2";
  }
  .list {
    overflow-x: scroll;
  }
  tr {
    height: 3rem;
  }
  th, td {
    padding: 0.5rem;
  }
  th {
    font-size: 0.9rem;
    font-weight: 700;
    vertical-align: bottom;
    text-align: left;
  }
  .right {
    text-align: right;
    justify-content: flex-end;
  }
  th > div {
    display: flex;
  }
  th button {
    all: unset;
    cursor: pointer;
    display: inline-flex;
    align-items: flex-end;
    height: 100%;
    width: 100%;
    text-align: inherit;
    font-family: inherit;
  }
  .header-description {
    font-size: 0.8rem;
    font-weight: normal;
    display: block;
    text-align: left;
  }
  .shield img {
    height: 2rem;
    width: auto;
  }
  .pagination {
    display: flex;
    justify-content: space-between;
  }
  .pagination button {
    cursor: pointer;
    font-family: inherit;
  }
</style>
