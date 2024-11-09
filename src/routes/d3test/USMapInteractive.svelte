<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';
	import * as topojson from 'topojson-client';
	import usMapData from './counties-albers-10m.json';

	let svg;
	let container;
	let path = d3.geoPath();
	let us = usMapData;
	let states = [];
	let meshStates;
	let electoralVotes = {};
	let totalDemocraticVotes = 226;
	let totalRepublicanVotes = 219;
	let totalTossUpVotes = 93;
	let totalUncommittedVotes = 0;
	let width = 600; // Default width, updated by ResizeObserver

	const targetVotes = 270;

	// Calculate total votes and update widths
	let sections = [
		{ label: 'Democrats', votes: totalDemocraticVotes, color: '#1d4ed8' }, // Dark blue
		{ label: 'Toss-Up', votes: totalTossUpVotes, color: '#facc15' }, // Yellow
		{ label: 'Republicans', votes: totalRepublicanVotes, color: '#b91c1c' }, // Dark red
		{ name: 'Uncommitted', votes: totalUncommittedVotes, color: 'gray' },

	];

	function updateSections() {
		totalUncommittedVotes = 538 - (totalDemocraticVotes + totalRepublicanVotes + totalTossUpVotes);
		sections = [
			{ name: 'Democrats', votes: totalDemocraticVotes, color: 'blue' },
			{ name: 'Toss-Ups', votes: totalTossUpVotes, color: 'yellow' },
			{ name: 'Uncommitted', votes: totalUncommittedVotes, color: 'gray' },
			{ name: 'Republicans', votes: totalRepublicanVotes, color: 'red' }
		];
	}


	// Function to update chart when data changes
	function updateChart() {
		if (!svg) return;
		updateSections();
		const totalVotes = d3.sum(sections, (d) => d.votes);

		const widthScale = d3.scaleLinear().domain([0, totalVotes]).range([0, width]);

		// Select all bars and bind data
		const bars = d3.select(svg).selectAll('.bar').data(sections, d => d.name);

		// Enter and update bars
		bars
			.enter()
			.append('rect')
			.attr('class', 'bar')
			.merge(bars)
			.attr('x', (d, i) => d3.sum(sections.slice(0, i), (d) => widthScale(d.votes)))
			.attr('y', 0)
			.attr('height', 60)
			.attr('width', (d) => widthScale(d.votes))
			.attr('fill', (d) => d.color);

		bars.exit().remove();
	}
	// State abbreviations for each FIPS code
	const stateAbbreviations = {
		'01': 'AL',
		'02': 'AK',
		'04': 'AZ',
		'05': 'AR',
		'06': 'CA',
		'08': 'CO',
		'09': 'CT',
		'10': 'DE',
		'11': 'DC',
		'12': 'FL',
		'13': 'GA',
		'15': 'HI',
		'16': 'ID',
		'17': 'IL',
		'18': 'IN',
		'19': 'IA',
		'20': 'KS',
		'21': 'KY',
		'22': 'LA',
		'23': 'ME',
		'24': 'MD',
		'25': 'MA',
		'26': 'MI',
		'27': 'MN',
		'28': 'MS',
		'29': 'MO',
		'30': 'MT',
		'31': 'NE',
		'32': 'NV',
		'33': 'NH',
		'34': 'NJ',
		'35': 'NM',
		'36': 'NY',
		'37': 'NC',
		'38': 'ND',
		'39': 'OH',
		'40': 'OK',
		'41': 'OR',
		'42': 'PA',
		'44': 'RI',
		'45': 'SC',
		'46': 'SD',
		'47': 'TN',
		'48': 'TX',
		'49': 'UT',
		'50': 'VT',
		'51': 'VA',
		'53': 'WA',
		'54': 'WV',
		'55': 'WI',
		'56': 'WY'
	};

	// List of small states to exclude from the map
	const smallStates = ['09', '10', '11', '24', '25', '33', '34', '44'];

	// Define initial colours for each state
	let stateColours = {};

	// To track the selected state for tooltip
	let selectedStateId = null;

	let tooltipPosition = { x: 0, y: 0 }; // Tooltip position

	$: updateChart();

	onMount(() => {
		states = topojson.feature(us, us.objects.states).features;
		meshStates = topojson.mesh(us, us.objects.states, (a, b) => a !== b);
		// Initialise electoral votes and colours for each state

		console.log({ states, stateColours });
		// Close tooltip when clicking outside of it
		document.addEventListener('click', handleClickOutside);
		// Observe the container's width
		const resizeObserver = new ResizeObserver((entries) => {
			for (let entry of entries) {
				width = entry.contentRect.width; // Update width with container's current width
				console.log({ width });
				updateChart(); // Re-render the chart with new width
			}
		});
		resizeObserver.observe(container);
		states.forEach((state) => {
			const stateId = state.id;
			electoralVotes[stateId] = getElectoralVotes(stateId);
			stateColours[stateId] = getStateColours(stateId); // Initial color for undecided
			setStateColour(stateId, getStateColours(stateId))
			updateChart();
		});
		updateChart();
	});

	function openTooltip(stateId, event) {
		selectedStateId = stateId;
		tooltipPosition = { x: event.pageX, y: event.pageY };
		event.stopPropagation(); // Prevent closing the tooltip immediately on open
	}

	function closeTooltip() {
		selectedStateId = null;
	}

	function handleClickOutside(event) {
		// Close tooltip if click is outside the tooltip element
		if (selectedStateId && !event.target.closest('.tooltip')) {
			closeTooltip();
		}
	}

	function setStateColour(stateId, color) {
		// Reset the vote counts for the previous color
		if (stateColours[stateId] === 'blue') {
			totalDemocraticVotes -= electoralVotes[stateId];
		} else if (stateColours[stateId] === 'red') {
			totalRepublicanVotes -= electoralVotes[stateId];
		} else if (stateColours[stateId] === '#f5ad42') {
			totalTossUpVotes -= electoralVotes[stateId];
		} else if (stateColours[stateId] === 'grey') {
			totalUncommittedVotes -= electoralVotes[stateId];
		}

		// Set the new color
		stateColours[stateId] = color;

		// Update the vote counts based on the selected color
		if (color === 'blue') {
			totalDemocraticVotes += electoralVotes[stateId];
		} else if (color === 'red') {
			totalRepublicanVotes += electoralVotes[stateId];
		} else if (color === '#f5ad42') {
			totalTossUpVotes += electoralVotes[stateId];
		} else if (color === 'grey') {
			totalUncommittedVotes += electoralVotes[stateId];
		}

		// Close the tooltip
		selectedStateId = null;
	}

	function getStateColours(stateId) {
		const colourByState = {
			'01': 'red', // Alabama
			'02': 'red', // Alaska
			'04': '#f5ad42', // Arizona
			'05': 'red', // Arkansas
			'06': 'blue', // California
			'08': 'blue', // Colorado
			'09': 'blue', // Connecticut
			'10': 'blue', // Delaware
			'11': 'blue', // District of Columbia
			'12': 'red', // Florida
			'13': '#f5ad42', // Georgia
			'15': 'blue', // Hawaii
			'16': 'red', // Idaho
			'17': 'blue', // Illinois
			'18': 'red', // Indiana
			'19': 'red', // Iowa
			'20': 'red', // Kansas
			'21': 'red', // Kentucky
			'22': 'red', // Louisiana
			'23': 'blue', // Maine
			'24': 'blue', // Maryland
			'25': 'blue', // Massachusetts
			'26': '#f5ad42', // Michigan
			'27': 'blue', // Minnesota
			'28': 'red', // Mississippi
			'29': 'red', // Missouri
			'30': 'red', // Montana
			'31': 'red', // Nebraska
			'32': '#f5ad42', // Nevada
			'33': 'blue', // New Hampshire
			'34': 'blue', // New Jersey
			'35': 'blue', // New Mexico
			'36': 'blue', // New York
			'37': '#f5ad42', // North Carolina
			'38': 'red', // North Dakota
			'39': 'red', // Ohio
			'40': 'red', // Oklahoma
			'41': 'blue', // Oregon
			'42': '#f5ad42', // Pennsylvania
			'44': 'blue', // Rhode Island
			'45': 'red', // South Carolina
			'46': 'red', // South Dakota
			'47': 'red', // Tennessee
			'48': 'red', // Texas
			'49': 'red', // Utah
			'50': 'blue', // Vermont
			'51': 'blue', // Virginia
			'53': 'blue', // Washington
			'54': 'red', // West Virginia
			'55': '#f5ad42', // Wisconsin
			'56': 'red' // Wyoming
		}

		return colourByState[stateId] || 'gray';
	}

	// Placeholder function for each state’s electoral votes
	function getElectoralVotes(stateId) {
		// This would return electoral votes for each state
		const electoralVotesByState = {
			'01': 9, // Alabama
			'02': 3, // Alaska
			'04': 11, // Arizona
			'05': 6, // Arkansas
			'06': 54, // California
			'08': 10, // Colorado
			'09': 7, // Connecticut
			'10': 3, // Delaware
			'11': 3, // District of Columbia
			'12': 30, // Florida
			'13': 16, // Georgia
			'15': 4, // Hawaii
			'16': 4, // Idaho
			'17': 19, // Illinois
			'18': 11, // Indiana
			'19': 6, // Iowa
			'20': 6, // Kansas
			'21': 8, // Kentucky
			'22': 8, // Louisiana
			'23': 4, // Maine
			'24': 10, // Maryland
			'25': 11, // Massachusetts
			'26': 15, // Michigan
			'27': 10, // Minnesota
			'28': 6, // Mississippi
			'29': 10, // Missouri
			'30': 4, // Montana
			'31': 5, // Nebraska
			'32': 6, // Nevada
			'33': 4, // New Hampshire
			'34': 14, // New Jersey
			'35': 5, // New Mexico
			'36': 28, // New York
			'37': 16, // North Carolina
			'38': 3, // North Dakota
			'39': 17, // Ohio
			'40': 7, // Oklahoma
			'41': 8, // Oregon
			'42': 19, // Pennsylvania
			'44': 4, // Rhode Island
			'45': 9, // South Carolina
			'46': 3, // South Dakota
			'47': 11, // Tennessee
			'48': 40, // Texas
			'49': 6, // Utah
			'50': 3, // Vermont
			'51': 13, // Virginia
			'53': 12, // Washington
			'54': 4, // West Virginia
			'55': 10, // Wisconsin
			'56': 3 // Wyoming
		};

		return electoralVotesByState[stateId] || 0;
	}
</script>

<div class="flex justify-center space-x-4 p-4">
	<div class="bg-red-500 text-white shadow-lg p-6 w-48 text-center">
		<h2 class="text-lg font-bold mb-2">Republican Vote</h2>
		<p class="text-3xl font-semibold">{totalRepublicanVotes}</p>
	</div>
	<div class="bg-blue-500 text-white shadow-lg p-6 w-48 text-center">
		<h2 class="text-lg font-bold mb-2">Democratic Vote</h2>
		<p class="text-3xl font-semibold">{totalDemocraticVotes}</p>
	</div>
	<div style="background-color: #f5ad42" class="text-white shadow-lg p-6 w-48 text-center">
		<h2 class="text-lg font-bold mb-2">Toss-Up Vote</h2>
		<p class="text-3xl font-semibold">{totalTossUpVotes}</p>
	</div>
	<div class="bg-gray-500 text-white shadow-lg p-6 w-48 text-center">
		<h2 class="text-lg font-bold mb-2">Uncommitted Vote</h2>
		<p class="text-3xl font-semibold">{totalUncommittedVotes}</p>
	</div>
</div>

<!-- SVG container for the bar chart -->
<div class="flex items-center space-x-2 p-4 w-full">
	<div class="text-blue-600 font-bold text-lg flex items-center label">
		Democrats <span class="ml-2 text-xl font-semibold">{totalDemocraticVotes}</span>
	</div>
	<div bind:this={container} class="chart-container w-full">
		<svg bind:this={svg} class="bar w-full" height="60"></svg>
	</div>
	<div class="text-red-600 font-bold text-lg flex items-center label">
		<span class="mr-2 text-xl font-semibold">{totalRepublicanVotes}</span> Republicans
	</div>
</div>

<div class="flex flex-row justify-center items-start space-x-8">
	<!-- Small states interactive boxes -->
	<div class="flex-grow">
		<svg viewBox="0 0 1000 610" class="w-full h-auto">
			<g fill="none" stroke="#fff" stroke-linejoin="round" stroke-linecap="round">
				{#each states as feature}
					<path
						fill={stateColours[feature.id]}
						on:click={(e) => openTooltip(feature.id, e)}
						on:keydown={(e) => (e.key === 'Enter' || e.key === ' ') && openTooltip(feature.id, e)}
						d={path(feature)}
						tabindex="0"
						role="button"
						style="cursor: pointer;"
						aria-label="Toggle state color"
					></path>
					{#if !smallStates.includes(feature.id)}
						{#if feature.id === '12'}
							<!-- Example: Special formatting for Florida (FIPS code 12) -->
							<text
								x={path.centroid(feature)[0] + 8}
								y={path.centroid(feature)[1] - 10}
								text-anchor="middle"
								class="text-xs fill-white"
								style="font-size: 10px;"
							>
								{stateAbbreviations[feature.id]}
							</text>
							<text
								x={path.centroid(feature)[0] + 8}
								y={path.centroid(feature)[1] + 5}
								text-anchor="middle"
								class="text-xs fill-white"
								style="font-size: 10px;"
							>
								{electoralVotes[feature.id]}
							</text>
						{:else}
							<!-- Default positioning for other states -->
							<text
								x={path.centroid(feature)[0]}
								y={path.centroid(feature)[1]}
								text-anchor="middle"
								class="text-xs fill-white"
								style="font-size: 10px;"
							>
								{stateAbbreviations[feature.id]}
							</text>
							<text
								x={path.centroid(feature)[0]}
								y={path.centroid(feature)[1] + 12}
								text-anchor="middle"
								class="text-xs fill-white"
								style="font-size: 10px;"
							>
								{electoralVotes[feature.id]}
							</text>
						{/if}
					{/if}
				{/each}
			</g>
		</svg>
	</div>
	<div class="w-1/8 flex flex-col space-y-2 p-4">
		{#each smallStates as stateId}
			<div
				class="w-24 p-4 rounded-lg shadow-lg text-white text-center cursor-pointer"
				style="background-color: {stateColours[stateId]}; cursor: pointer;"
				on:click={(e) => {
					openTooltip(stateId, e), console.log(e);
				}}
				on:keydown={(e) =>
					(e.key === 'Enter' || e.key === ' ') && openTooltip(stateId, e) && console.log(e)}
				tabindex="0"
				role="button"
				aria-label="Toggle state color"
			>
				<p class="text-lg font-bold">{stateAbbreviations[stateId]}</p>
				<p class="text-sm font-semibold">{electoralVotes[stateId]} votes</p>
			</div>
		{/each}
	</div>
</div>

<!-- Tooltip for color selection (rendered outside of SVG for layering control) -->
{#if selectedStateId}
	<div
		class="absolute bg-white shadow-lg p-4 rounded z-50 text-center tooltip"
		style="top: {tooltipPosition.y}px; left: {tooltipPosition.x}px; transform: translate(-50%, -110%); width: 200px;"
	>
		<button
			on:click={closeTooltip}
			class="absolute top-1 right-1 text-gray-600 hover:text-gray-800"
		>
			&times;
		</button>
		<p class="text-sm font-bold mb-2">Select Party</p>
		<div class="flex flex-col space-y-2">
			<button
				class="btn colour-blue w-full"
				on:click={() => setStateColour(selectedStateId, 'blue')}>Democrat</button
			>
			<button class="btn colour-red w-full" on:click={() => setStateColour(selectedStateId, 'red')}
				>Republican</button
			>
			<button
				class="btn colour-yellow w-full"
				on:click={() => setStateColour(selectedStateId, '#f5ad42')}>Toss-Up</button
			>
			<button
				class="btn colour-gray w-full"
				on:click={() => setStateColour(selectedStateId, 'grey')}>Uncommitted</button
			>
		</div>
	</div>
{/if}

<style>
	path[role='button']:focus {
		outline: none;
	}
	.tooltip {
		position: absolute;
	}
	/* Additional styling for bar chart */
	.bar {
		transition: width 0.3s;
	}
	.label {
		font-size: 12px;
	}
</style>
