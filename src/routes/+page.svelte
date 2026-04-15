<script lang="ts">
	import { onMount } from 'svelte';
	import { pb } from '$lib/pb';

	// 1. Types & Config
	interface MenuRecord {
		id: string;
		mess_name: 'Polytechnic' | 'Management';
		date: string;
		breakfast: string;
		lunch: string;
		snacks: string;
		dinner: string;
	}

	const expectedMesses = ['Polytechnic', 'Management'] as const;

	const mealConfig = [
		{ key: 'breakfast' as const, label: 'Breakfast', icon: '🌅', color: 'text-primary' },
		{ key: 'lunch' as const, label: 'Lunch', icon: '🍲', color: 'text-secondary' },
		{ key: 'snacks' as const, label: 'Snacks', icon: '☕', color: 'text-accent' },
		{ key: 'dinner' as const, label: 'Dinner', icon: '🌃', color: 'text-info' }
	];

	// 2. Svelte 5 Runes for State
	let displayRecords = $state<Partial<MenuRecord>[]>([]);
	let isLoading = $state(true);
	let errorMsg = $state<string | null>(null);

	// 3. Logic: Fetch & Normalize
	const loadMenu = async () => {
		isLoading = true;
		errorMsg = null;

		try {
			const today = new Date().toISOString().split('T')[0];

			// Fetch whatever is available
			const records = await pb.collection('menu').getFullList<MenuRecord>({
				filter: `date ~ "${today}"`
			});

			// Map expected messes to fetched data, or return a placeholder if missing
			displayRecords = expectedMesses.map((name) => {
				const found = records.find((r) => r.mess_name === name);
				return found || { mess_name: name, breakfast: '', lunch: '', snacks: '', dinner: '' };
			});
		} catch (err: any) {
			errorMsg = err.message || 'Failed to connect to PocketBase';
		} finally {
			isLoading = false;
		}
	};

	onMount(loadMenu);
</script>

<div class="min-h-screen bg-base-200 p-4 lg:p-12">
	<div class="mx-auto max-w-7xl">
		<header class="mb-10 flex flex-col items-end justify-between gap-4 md:flex-row">
			<div>
				<h1 class="mb-2 text-5xl font-black tracking-tighter text-base-content uppercase">
					Today's Menu
				</h1>
				<p class="text-lg font-medium opacity-60">
					{new Date().toLocaleDateString('en-GB', { dateStyle: 'full' })}
				</p>
			</div>
			<div class="flex gap-2">
				<button
					class="btn border-base-300 bg-base-100 shadow-sm btn-ghost"
					onclick={loadMenu}
					disabled={isLoading}
				>
					{#if isLoading}
						<span class="loading loading-xs loading-spinner"></span>
					{:else}
						⟳
					{/if}
					Refresh
				</button>
			</div>
		</header>

		{#if errorMsg}
			<div class="alert alert-error shadow-lg">
				<svg
					xmlns="http://www.w3.org/2000/svg"
					class="h-6 w-6 shrink-0 stroke-current"
					fill="none"
					viewBox="0 0 24 24"
					><path
						stroke-linecap="round"
						stroke-linejoin="round"
						stroke-width="2"
						d="M10 14l2-2m0 0l2-2m-2 2l-2-2m2 2l2 2m7-2a9 9 0 11-18 0 9 9 0 0118 0z"
					/></svg
				>
				<span>{errorMsg}</span>
			</div>
		{:else if isLoading}
			<div class="grid grid-cols-1 gap-8 lg:grid-cols-2">
				<div class="h-[500px] w-full skeleton rounded-3xl"></div>
				<div class="h-[500px] w-full skeleton rounded-3xl"></div>
			</div>
		{:else}
			<div class="grid grid-cols-1 gap-8 lg:grid-cols-2">
				{#each displayRecords as mess}
					<div
						class="card border-b-8 bg-base-100 shadow-xl {mess.mess_name === 'Polytechnic'
							? 'border-primary'
							: 'border-secondary'}"
					>
						<div class="card-body p-0">
							<div class="flex items-center justify-between p-8">
								<h2 class="card-title text-3xl font-black tracking-tight italic">
									{mess.mess_name}
								</h2>
								<div class="badge badge-ghost font-mono opacity-50">TODAY</div>
							</div>

							<div class="overflow-x-auto px-2 pb-4">
								<table class="table w-full table-lg">
									<thead>
										<tr class="bg-base-200/50 text-base tracking-widest uppercase">
											<th class="w-1/3">Meal</th>
											<th>Menu Item</th>
										</tr>
									</thead>
									<tbody>
										{#each mealConfig as meal}
											<tr class="transition-colors hover:bg-base-200/30">
												<td class="py-6 font-bold">
													<div class="flex items-center gap-4">
														<span class="text-2xl">{meal.icon}</span>
														<span class={meal.color}>{meal.label}</span>
													</div>
												</td>
												<td class="leading-relaxed font-medium text-base-content/80 italic">
													{mess[meal.key] || 'No items added yet'}
												</td>
											</tr>
										{/each}
									</tbody>
								</table>
							</div>
						</div>
					</div>
				{/each}
			</div>
		{/if}
	</div>
</div>

<style>
	/* Optional: Smooth transition for the refresh action */
	:global(html) {
		scroll-behavior: smooth;
	}
</style>
