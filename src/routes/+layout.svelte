<script lang="ts">
	import { onMount } from 'svelte';
	import { pb } from '$lib/pb';
	import './layout.css';

	let { children } = $props();

	let isAdminLoggedIn = $state(false);

	onMount(() => {
		isAdminLoggedIn = pb.authStore.isValid;
	});
</script>

<svelte:head>
	<link rel="icon" href="https://fav.farm/🍽️" />
	<title>Mess Menu</title>
</svelte:head>

<div class="flex min-h-screen flex-col bg-base-200">
	<main class="grow">
		{@render children()}
	</main>
	<footer
		class="footer-center footer border-t border-base-300/10 bg-base-300 px-4 py-6 text-base-content"
	>
		<aside class="flex flex-col items-center gap-2">
			{#if !isAdminLoggedIn}
				<a
					href="/admin/login"
					class="text-xs font-semibold text-base-content/40 transition-colors hover:text-primary hover:underline"
				>
					Admin Login
				</a>
			{/if}
			<p class="text-xs font-medium text-base-content/60 sm:text-sm">
				&copy; {new Date().getFullYear()} &bull; Built by
				<span
					class="font-semibold text-base-content transition-colors duration-200 hover:text-primary"
					>KH Systems Private Limited</span
				>
			</p>
		</aside>
	</footer>
</div>
