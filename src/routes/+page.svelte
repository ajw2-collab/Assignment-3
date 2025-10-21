<script lang="ts">
	// Imports copied from the template.
	import AQIChart from '$lib/AQIChart.svelte';
	import * as d3 from 'd3';

	// Datasets copied from the template.
	const datasets = {
		avalon: 'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BAvalon%5D_daily-avg.csv',
		glassport_high_street:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BGlassport%20High%20Street%5D_daily-avg.csv',
		lawrenceville:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLawrenceville%5D_daily-avg.csv',
		liberty_sahs:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BLiberty%20(SAHS)%5D_daily-avg.csv',
		manchester:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BManchester%5D_daily-avg.csv',
		north_braddock:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BNorth%20Braddock%5D_daily-avg.csv',
		parkway_east_near_road:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BParkway%20East%20(Near%20Road)%5D_daily-avg.csv',
		usa_pennsylvania_pittsburgh:
			'https://dig.cmu.edu/datavis-fall-2025/assignments/data/%5BUSA-Pennsylvania-Pittsburgh%5D_daily-avg.csv'
	};

	// Changed to let due to const doesn't allow bind.
	let selectedDataset: keyof typeof datasets = $state('avalon');

	// Make boolean for checkbox.
	let showRawPoints = $state(true);

	// Seperate original data constant function into simple transformation function that takes any type and transform into this object type.
	const transformFn = (d: any) => ({
		city: d.City,
		country: d.Country,
		mainPollutant: d['Main pollutant'],
		pm25: +d['PM2.5'],
		state: d.State,
		stationName: d['Station name'],
		timestamp: new Date(d['Timestamp(UTC)']),
		usAqi: +d['US AQI']
	});

	const data = $derived.by(() => {
		if (selectedDataset) {
			// If a station is selected, load just that one.
			return d3.csv(datasets[selectedDataset], transformFn);
		} else {
			// If no station is selected (meaning '' is selected), load all stations and flatten.
			return Promise.all(Object.values(datasets).map(url => d3.csv(url, transformFn))).then(arrays => arrays.flat());
		}
	});

	// Creates constant stationNames that reads the csv file, thus require async function to wait till it finishes reading
	const stationNames = $derived.by(async () => {

		// Creates an object with string key and string Actual name for better visualization of course.
		const names: Record<string, string> = {};
		
		//counter
		let total: number = 0;

		// For each key from datasets
		for (const key of Object.keys(datasets)) {
			
			// Get the dataset URL (and kept TypeScript Happy).
			const url = datasets[key as keyof typeof datasets];

			// Extract the part between /%5B (a.k.a. [ ) and /%5D (a.k.a. ] ).
			const match = url.match(/%5B(.*?)%5D/);

			// If match is null return key, if not decode match just to change %20 to space.
			const name = match ? decodeURIComponent(match[1]) : key;

			// Load CSV to get record count and add to total count for all station.
			const data = await d3.csv(url);
			const count = data.length;
			total += count;

			// Shove name back in the object stationNames.
			names[key] = `${name} (${count})`;
		}

		names[''] = "All Station (" + total + ")";

		return names;
	});

	// Creates constant recordNumbers that reads the csv file, thus require async function to wait till it finishes reading
	const recordNumbers = $derived.by(async () => {

		// Creates an object with string key and number data length.
		const numbers: Record<string, number> = {};

		//counter
		let total: number = 0;

		// For each key from datasets.
		for (const key of Object.keys(datasets)) {
			
			// Load CSV to get record count and add to total count for all station.
			const data = await d3.csv(datasets[key as keyof typeof datasets]);
			const count = data.length;
			total += count;

			// Shove datalength back in the object recordNumbers.
			numbers[key] = count;
		}

		numbers[''] = total;
		return numbers;
	});

	// Palette
	const palette = [
		'red',
		'orange',
		'gold',
		'green',
		'blue',
		'indigo',
		'violet',
		'black'
	];

	// Decode the display name again and return it as decoded string.
	const nameFromUrl = (url: string) => {
			// Extract the part between /%5B (a.k.a. [ ) and /%5D (a.k.a. ] ).
			const match = url.match(/%5B(.*?)%5D/);

			// If match is null return key, if not decode match just to change %20 to space.
			return match ? decodeURIComponent(match[1]) : url;
	};

	// Stores station name with color.
	const colorMap: Record<string, string> = {};

	// For key grabs the url, change the name, and use the index of it then shove color from palette to colorMap.
	Object.entries(datasets).forEach(([key, url], i) => {
		const display = nameFromUrl(url);
		colorMap[display] = palette[i];
	});

</script>

<!--My title -->
<h1>Andrew's AQI Chart</h1>

<!-- Dropdown label -->
<label for="station">Choose station:</label>

<!-- Toggle Functions -->

{#await Promise.all([stationNames, recordNumbers, data])}
	<p>Loading stationNames ...</p>
{:then [stationNames, recordNumbers, data]}

<!-- Dropdown function -->
<select id="station" bind:value={selectedDataset}>


	{#each Object.keys(stationNames)
		//Sort from highest to lowest, for example, all station contains the most, thus being the first.
		.sort((a, b) => (recordNumbers[b] ?? 0) - (recordNumbers[a] ?? 0)) as key}
		<option value={key}>{stationNames[key]}</option>
	{/each}
	
</select>

<!-- Number of records for this chart -->
<h2>{"Number of records: " + recordNumbers[selectedDataset]}</h2>

<br>

<!-- Checkbox function -->
<input type="checkbox" bind:checked={showRawPoints} /> Show Raw Data

<br>

<!-- Load AQIChart -->
<AQIChart {data} {showRawPoints} {colorMap}/>

{:catch error}
	<p style="color: red;">Something went wrong: {error.message}</p>
{/await}


<style>
	* {
		font-family: sans-serif;
	}

	select {
		margin: 1em 0;
		padding: 0.5em;
		font-size: 1em;
	}

	svg {
		max-width: 100%;
		overflow: visible;
		margin-block: 2em;
	}
</style>