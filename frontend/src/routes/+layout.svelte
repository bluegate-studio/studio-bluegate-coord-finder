<script>
	import '../app.css';

	let { children } = $props();
</script>

<svelte:head>
	<title>Coord Finder</title>
</svelte:head>

{@render children()}
