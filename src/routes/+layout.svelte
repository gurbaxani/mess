<script lang="ts">
	import './layout.css';

	let { children } = $props();
</script>

<svelte:head>
	<link rel="icon" href="https://fav.farm/🍽️" />
	<title>Mess Menu</title>
</svelte:head>
{@render children()}
