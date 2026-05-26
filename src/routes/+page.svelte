<script lang="ts">
	import { onMount } from 'svelte';
	import { pb } from '$lib/pb';
	import { Sunrise, UtensilsCrossed, Coffee, Moon, RefreshCw, User } from 'lucide-svelte';

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
		{
			key: 'breakfast' as const,
			label: 'Breakfast',
			icon: Sunrise,
			time: '7:30 – 9:30 am',
			badge: 'badge-warning',
			iconBtn: 'btn-warning'
		},
		{
			key: 'lunch' as const,
			label: 'Lunch',
			icon: UtensilsCrossed,
			time: '12:30 – 2:30 pm',
			badge: 'badge-success',
			iconBtn: 'btn-success'
		},
		{
			key: 'snacks' as const,
			label: 'Snacks',
			icon: Coffee,
			time: '4:30 – 5:30 pm',
			badge: 'badge-accent',
			iconBtn: 'btn-accent'
		},
		{
			key: 'dinner' as const,
			label: 'Dinner',
			icon: Moon,
			time: '7:30 – 9:30 pm',
			badge: 'badge-info',
			iconBtn: 'btn-info'
		}
	];

	let displayRecords = $state<Partial<MenuRecord>[]>([]);
	let isLoading = $state(true);
	let errorMsg = $state<string | null>(null);
	let spinning = $state(false);
	let isAdminLoggedIn = $state(false);

	const loadMenu = async () => {
		isLoading = true;
		spinning = true;
		errorMsg = null;
		try {
			const today = new Date().toISOString().split('T')[0];
			const records = await pb.collection('menu').getFullList<MenuRecord>({
				filter: `date ~ "${today}"`
			});
			displayRecords = expectedMesses.map((name) => {
				const found = records.find((r) => r.mess_name === name);
				return found || { mess_name: name, breakfast: '', lunch: '', snacks: '', dinner: '' };
			});
		} catch (err) {
			errorMsg = err instanceof Error ? err.message : 'Failed to connect to PocketBase';
		} finally {
			isLoading = false;
			setTimeout(() => (spinning = false), 600);
		}
	};

	onMount(() => {
		isAdminLoggedIn = pb.authStore.isValid;
		loadMenu();
	});
</script>

<div class="p-4 lg:p-10">
	<div class="mx-auto max-w-4xl">
		<header class="mb-10 flex items-end justify-between">
			<div>
				<p class="mb-1 text-sm font-medium tracking-widest text-base-content/40 uppercase">
					{new Date().toLocaleDateString('en-GB', { weekday: 'long' })}
				</p>
				<h1 class="text-4xl font-black tracking-tight text-base-content">What's cooking 🍽️</h1>
			</div>
			<div class="flex items-center gap-2">
				{#if isAdminLoggedIn}
					<a
						href="/admin"
						class="animate-in fade-in btn gap-2 rounded-full font-bold btn-sm btn-primary"
					>
						<User size={14} />
						Admin Portal
					</a>
				{/if}
				<button
					type="button"
					class="btn gap-2 rounded-full btn-ghost btn-sm"
					onclick={loadMenu}
					disabled={isLoading}
				>
					<RefreshCw size={14} class={spinning ? 'animate-spin' : ''} />
					Refresh
				</button>
			</div>
		</header>

		{#if errorMsg}
			<div class="alert rounded-2xl alert-error">
				<span>{errorMsg}</span>
			</div>
		{:else if isLoading}
			<div class="space-y-4">
				<div class="h-72 w-full skeleton rounded-3xl"></div>
				<div class="h-72 w-full skeleton rounded-3xl"></div>
			</div>
		{:else}
			<div class="space-y-6">
				{#each displayRecords as mess, i (mess.mess_name)}
					<div class="card bg-base-100 shadow-md">
						<div class="card-body p-0">
							<div class="flex items-center justify-between border-b border-base-200 px-6 py-4">
								<div>
									<h2 class="text-xl font-black tracking-tight">{mess.mess_name} Mess</h2>
									<p class="text-xs text-base-content/40">
										{new Date().toLocaleDateString('en-GB', { dateStyle: 'long' })}
									</p>
								</div>
								<div class="badge {i === 0 ? 'badge-primary' : 'badge-secondary'} badge-outline">
									Today
								</div>
							</div>

							<div class="grid grid-cols-2 divide-x divide-y divide-base-200 lg:grid-cols-4">
								{#each mealConfig as meal (meal.key)}
									{@const items = mess[meal.key]}
									{@const Icon = meal.icon}
									<div class="flex flex-col gap-4 p-5">
										<div class="flex items-center gap-3">
											<div
												class="btn {meal.iconBtn} no-animation pointer-events-none btn-square rounded-xl btn-sm"
											>
												<Icon size={15} />
											</div>
											<div>
												<p class="text-sm leading-none font-bold">{meal.label}</p>
												<p class="mt-0.5 text-xs text-base-content/40">{meal.time}</p>
											</div>
										</div>

										{#if items}
											<div class="flex flex-wrap gap-1.5">
												{#each items.split(',') as item, idx (item + '-' + idx)}
													<span class="badge {meal.badge} badge-soft badge-sm">
														{item.trim()}
													</span>
												{/each}
											</div>
										{:else}
											<p class="text-xs text-base-content/30 italic">Not updated yet</p>
										{/if}
									</div>
								{/each}
							</div>
						</div>
					</div>
				{/each}
			</div>
		{/if}
	</div>
</div>

<style>
	:global(.animate-spin) {
		animation: spin 0.6s linear infinite;
	}
	@keyframes spin {
		from {
			transform: rotate(0deg);
		}
		to {
			transform: rotate(360deg);
		}
	}
</style>
