<script>
	import { tweened } from 'svelte/motion';
	import { cubicOut } from 'svelte/easing';

	let aControl = 1; 
	const a = tweened(1, { duration: 1000, easing: cubicOut }); 

	const generateData = (aValue) => {
		const data = [];
		const numPoints = 1000; 
		for (let i = 0; i <= numPoints; i++) {
			let theta = (i / numPoints) * 2 * Math.PI;
			let r = Math.sin((aValue / 5) * theta);
			data.push({
				x: r * Math.cos(theta),
				y: r * Math.sin(theta)
			});
		}
		return data;
	};

	const generatePathData = (data) => {
		return data.reduce((acc, point, index) => {
			const command = index === 0 ? 'M' : 'L';
			return `${acc} ${command} ${point.x},${point.y}`;
		}, '');
	};

	let data = [];
	let pathData = '';

	$: data = generateData($a);
	$: pathData = generatePathData(data);
	$: a.set(aControl);
</script>

<input type="range" min="1" max="20" step="0.1" bind:value={aControl} />

<svg width="400" height="400" viewBox="-2 -2 4 4">
	<path d={pathData} fill="none" stroke="black" stroke-width="0.01" />
</svg>

<style>
	input {
		width: 100%;
		margin-bottom: 1rem;
	}
</style>
