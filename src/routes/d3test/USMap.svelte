<script>
	import * as d3 from 'd3';
	import { onMount } from 'svelte';
    import * as topojson from 'topojson-client';
	import usMapData from './counties-albers-10m.json';
	let path = d3.geoPath();
	let us = usMapData;
	let deps = [];
	let meshCounties;
    let meshStates;
	onMount(() => {
		deps = topojson.feature(us, us.objects.nation).features;
		console.log({ deps });
		meshCounties = topojson.mesh(
			us,
			us.objects.counties,
			(a, b) => a !== b && ((a.id / 1000) | 0) === ((b.id / 1000) | 0)
		);
        meshStates = topojson.mesh(us, us.objects.states, (a, b) => a !== b)
	});
</script>

<svg viewBox="0 0 1000 610">
	<g fill="none" stroke="#fff" stroke-linejoin="round" stroke-linecap="round">
		{#each deps as feature, i}
			<path d={path(feature)} />
            <path stroke="#fff" d={path(meshStates)} />
            <path stroke="#fff" stroke-width="0.25" d={path(meshCounties)}></path>
		{/each}
	</g>
</svg>
