<script lang="ts">
	import { onMount } from 'svelte';
	import { pb } from '$lib/pb';
	import type { MenuResponse } from '../../../pocketbase-types';
	import {
		LogOut,
		Save,
		Coffee,
		Sunrise,
		UtensilsCrossed,
		Moon,
		AlertCircle,
		CheckCircle2
	} from 'lucide-svelte';

	let isLoading = $state(true);
	let isSaving = $state<string | null>(null);
	let errorMsg = $state<string | null>(null);
	let successMsg = $state<string | null>(null);

	const messes = ['Polytechnic', 'Management'] as const;
	type MessName = (typeof messes)[number];

	let activeMessTab = $state<MessName>('Polytechnic');

	const mealConfig = [
		{
			key: 'breakfast',
			label: 'Breakfast',
			icon: Sunrise,
			placeholder: 'Idli, Sambar, Chutney...'
		},
		{
			key: 'lunch',
			label: 'Lunch',
			icon: UtensilsCrossed,
			placeholder: 'Rice, Dal, Roti, Sabzi...'
		},
		{ key: 'snacks', label: 'Snacks', icon: Coffee, placeholder: 'Samosa, Tea...' },
		{ key: 'dinner', label: 'Dinner', icon: Moon, placeholder: 'Fried Rice, Manchurian...' }
	] as const;

	// We store the data in an object keyed by mess name
	let menuData = $state<Record<MessName, Partial<MenuResponse>>>({
		Polytechnic: { mess_name: 'Polytechnic', breakfast: '', lunch: '', snacks: '', dinner: '' },
		Management: { mess_name: 'Management', breakfast: '', lunch: '', snacks: '', dinner: '' }
	});

	onMount(async () => {
		if (!pb.authStore.isValid) {
			window.location.href = '/admin/login';
			return;
		}

		isLoading = true;
		try {
			const today = new Date().toISOString().split('T')[0];
			const records = await pb.collection('menu').getFullList<MenuResponse>({
				filter: `date ~ "${today}"`
			});

			for (const mess of messes) {
				const existing = records.find((r) => r.mess_name === mess);
				if (existing) {
					menuData[mess] = existing;
				} else {
					menuData[mess] = {
						mess_name: mess,
						breakfast: '',
						lunch: '',
						snacks: '',
						dinner: '',
						date: today
					};
				}
			}
		} catch (err) {
			errorMsg = err instanceof Error ? err.message : 'Failed to load menu data';
		} finally {
			isLoading = false;
		}
	});

	const handleSave = async (mess: MessName) => {
		isSaving = mess;
		errorMsg = null;
		successMsg = null;
		try {
			const data = menuData[mess];
			const today = new Date().toISOString().split('T')[0];

			if (data.id) {
				await pb.collection('menu').update(data.id, {
					breakfast: data.breakfast,
					lunch: data.lunch,
					snacks: data.snacks,
					dinner: data.dinner
				});
			} else {
				const created = await pb.collection('menu').create({
					mess_name: mess,
					date: today,
					breakfast: data.breakfast,
					lunch: data.lunch,
					snacks: data.snacks,
					dinner: data.dinner
				});
				menuData[mess] = created as unknown as MenuResponse;
			}

			successMsg = `Awesome! ${mess} menu updated.`;
			setTimeout(() => {
				successMsg = null;
			}, 3000);
		} catch (err) {
			errorMsg = err instanceof Error ? err.message : `Failed to save ${mess} menu`;
		} finally {
			isSaving = null;
		}
	};

	const handleLogout = () => {
		pb.authStore.clear();
		window.location.href = '/admin/login';
	};
</script>

<svelte:head>
	<title>Manage Menu - Mess Menu</title>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,200..800&family=Lexend:wght@300..700&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="px-4 py-8 sm:px-6 lg:px-8 lg:py-12">
	<div class="mx-auto max-w-6xl">
		<header
			class="mb-10 flex flex-col items-start justify-between gap-6 sm:flex-row sm:items-center"
		>
			<div>
				<p class="mb-1.5 text-xs font-bold tracking-wider text-primary uppercase sm:text-sm">
					Admin Portal
				</p>
				<h1
					class="font-display text-4xl font-extrabold tracking-tight text-base-content sm:text-5xl"
				>
					Today's Menu <span class="inline-block animate-bounce select-none">👩‍🍳</span>
				</h1>
			</div>
			<button
				type="button"
				class="flex items-center gap-2 rounded-2xl border-2 border-error/20 bg-error/5 px-5 py-3 font-bold text-error transition-all hover:bg-error hover:text-error-content hover:shadow-lg hover:shadow-error/15 active:scale-95"
				onclick={handleLogout}
			>
				<LogOut size={18} />
				Log Out
			</button>
		</header>

		<!-- Mobile/Tablet Tab Switcher -->
		<div class="mb-8 flex rounded-2xl bg-base-300/50 p-1.5 lg:hidden">
			{#each messes as mess (mess)}
				<button
					type="button"
					class="flex-1 rounded-xl py-3 text-sm font-bold transition-all duration-200 {activeMessTab ===
					mess
						? 'bg-base-100 text-primary shadow-sm'
						: 'text-base-content/60 hover:text-base-content'}"
					onclick={() => (activeMessTab = mess)}
				>
					{mess} Mess
				</button>
			{/each}
		</div>

		{#if isLoading}
			<div class="grid gap-8 lg:grid-cols-2">
				{#each messes as mess (mess)}
					<div
						class="mess-{mess.toLowerCase()} flex flex-col rounded-[24px] border-2 border-[var(--mess-border)] bg-[var(--mess-card-bg)] opacity-60 {activeMessTab ===
						mess
							? 'flex'
							: 'hidden lg:flex'}"
					>
						<!-- Punched holes header -->
						<div
							class="flex items-center justify-between rounded-t-[22px] border-b-2 border-dashed border-[var(--mess-border)] bg-[var(--mess-bg)] px-6 py-4"
						>
							<div class="flex items-center gap-2">
								<div class="h-3 w-3 rounded-full bg-[var(--mess-border)]"></div>
								<div class="h-3 w-3 rounded-full bg-[var(--mess-border)]"></div>
								<div class="h-3 w-3 rounded-full bg-[var(--mess-border)]"></div>
							</div>
							<div class="h-5 w-16 skeleton rounded-full bg-[var(--mess-border)]/50"></div>
						</div>
						<div class="space-y-6 p-6 sm:p-8">
							<div>
								<div class="h-8 w-48 skeleton rounded-xl bg-[var(--mess-border)]/50"></div>
								<div class="mt-2 h-4 w-32 skeleton rounded-lg bg-[var(--mess-border)]/30"></div>
							</div>
							<div class="grid gap-6 sm:grid-cols-2">
								{#each Array(4) as _, i (i)}
									<div class="space-y-2">
										<div class="h-5 w-24 skeleton rounded-lg bg-[var(--mess-border)]/40"></div>
										<div class="h-28 w-full skeleton rounded-2xl bg-[var(--mess-border)]/20"></div>
									</div>
								{/each}
							</div>
							<div class="h-14 w-full skeleton rounded-2xl bg-[var(--mess-border)]/40"></div>
						</div>
					</div>
				{/each}
			</div>
		{:else}
			<div class="grid gap-8 lg:grid-cols-2">
				{#each messes as mess (mess)}
					<div
						class="mess-{mess.toLowerCase()} flex flex-col rounded-[24px] border-2 border-[var(--mess-border)] bg-[var(--mess-card-bg)] shadow-[var(--mess-accent)]/5 shadow-2xl transition-all duration-300 hover:shadow-[var(--mess-accent)]/10 hover:shadow-xl {activeMessTab ===
						mess
							? 'flex'
							: 'hidden lg:flex'}"
					>
						<!-- Notepad punched holes header -->
						<div
							class="flex items-center justify-between rounded-t-[22px] border-b-2 border-dashed border-[var(--mess-border)] bg-[var(--mess-bg)] px-6 py-4"
						>
							<div class="flex items-center gap-2">
								<div class="h-3 w-3 rounded-full bg-[var(--mess-border)]"></div>
								<div class="h-3 w-3 rounded-full bg-[var(--mess-border)]"></div>
								<div class="h-3 w-3 rounded-full bg-[var(--mess-border)]"></div>
							</div>
							<div
								class="badge border-none bg-[var(--mess-badge-bg)] px-3 py-1 text-xs font-bold text-[var(--mess-badge-text)]"
							>
								Active Today
							</div>
						</div>

						<div class="flex flex-grow flex-col justify-between p-6 sm:p-8">
							<div>
								<div class="mb-6">
									<h2 class="font-display text-3xl font-extrabold text-[var(--mess-text-main)]">
										{mess} Mess
									</h2>
									<p class="text-xs font-semibold text-[var(--mess-text-muted)]/70">
										Set the menu for {new Date().toLocaleDateString('en-US', {
											weekday: 'long',
											month: 'short',
											day: 'numeric'
										})}
									</p>
								</div>

								<div class="grid gap-6 sm:grid-cols-2">
									{#each mealConfig as meal (meal.key)}
										{@const Icon = meal.icon}
										<div class="flex flex-col gap-2">
											<label
												for="{mess}-{meal.key}"
												class="flex items-center gap-2 text-sm font-bold text-[var(--mess-text-muted)]"
											>
												<div
													class="flex h-7 w-7 items-center justify-center rounded-lg bg-[var(--mess-badge-bg)] text-[var(--mess-badge-text)]"
												>
													<Icon size={14} />
												</div>
												{meal.label}
											</label>
											<textarea
												id="{mess}-{meal.key}"
												class="textarea h-28 w-full resize-none rounded-2xl border-2 border-[var(--mess-border)] bg-[var(--mess-input-bg)] px-4 py-3 font-medium text-[var(--mess-text-main)] transition-all placeholder:text-[var(--mess-text-muted)]/45 focus:border-[var(--mess-accent)] focus:ring-4 focus:ring-[var(--mess-accent-ring)] focus:outline-none"
												placeholder={meal.placeholder}
												bind:value={menuData[mess][meal.key]}
											></textarea>
										</div>
									{/each}
								</div>
							</div>

							<div class="mt-8">
								<button
									type="button"
									class="relative flex w-full items-center justify-center gap-2 overflow-hidden rounded-2xl bg-[var(--mess-accent)] py-4 font-bold text-white transition-all hover:bg-[var(--mess-accent-hover)] hover:shadow-[var(--mess-accent)]/20 hover:shadow-lg focus:ring-4 focus:ring-[var(--mess-accent-ring)] active:scale-98 disabled:opacity-50"
									onclick={() => handleSave(mess)}
									disabled={isSaving !== null}
								>
									{#if isSaving === mess}
										<span class="loading loading-sm loading-spinner"></span>
										Saving Menu...
									{:else}
										<Save size={18} />
										Save {mess} Menu
									{/if}
								</button>
							</div>
						</div>
					</div>
				{/each}
			</div>
		{/if}
	</div>
</div>

<!-- Toast Notifications -->
<div class="fixed bottom-6 left-1/2 z-50 w-full max-w-sm -translate-x-1/2 px-4">
	{#if errorMsg}
		<div
			class="animate-in fade-in slide-in-from-bottom-5 flex items-center gap-3 rounded-2xl border-2 border-error bg-base-100 p-4 text-error shadow-2xl transition-all duration-300"
		>
			<AlertCircle size={20} class="shrink-0" />
			<p class="text-sm font-semibold">{errorMsg}</p>
		</div>
	{/if}

	{#if successMsg}
		<div
			class="animate-in fade-in slide-in-from-bottom-5 flex items-center gap-3 rounded-2xl border-2 border-success bg-base-100 p-4 text-success shadow-2xl transition-all duration-300"
		>
			<CheckCircle2 size={20} class="shrink-0" />
			<p class="text-sm font-semibold">{successMsg}</p>
		</div>
	{/if}
</div>

<style>
	:global(body) {
		font-family: 'Lexend', sans-serif;
	}
	.font-display {
		font-family: 'Bricolage Grotesque', sans-serif;
	}

	.mess-polytechnic {
		--mess-bg: #fffbeb;
		--mess-card-bg: #ffffff;
		--mess-border: #fde68a;
		--mess-accent: #d97706;
		--mess-accent-hover: #b45309;
		--mess-accent-ring: rgba(217, 119, 6, 0.15);
		--mess-input-bg: #fffdf5;
		--mess-text-main: #78350f;
		--mess-text-muted: #92400e;
		--mess-badge-bg: #fef3c7;
		--mess-badge-text: #b45309;
	}

	.mess-management {
		--mess-bg: #f0fdf4;
		--mess-card-bg: #ffffff;
		--mess-border: #a7f3d0;
		--mess-accent: #059669;
		--mess-accent-hover: #047857;
		--mess-accent-ring: rgba(5, 150, 105, 0.15);
		--mess-input-bg: #f9fefb;
		--mess-text-main: #064e3b;
		--mess-text-muted: #065f46;
		--mess-badge-bg: #d1fae5;
		--mess-badge-text: #047857;
	}

	@media (prefers-color-scheme: dark) {
		.mess-polytechnic {
			--mess-bg: #1c1917;
			--mess-card-bg: #292524;
			--mess-border: #44403c;
			--mess-accent: #f59e0b;
			--mess-accent-hover: #fbbf24;
			--mess-accent-ring: rgba(245, 158, 11, 0.2);
			--mess-input-bg: #1c1917;
			--mess-text-main: #fef3c7;
			--mess-text-muted: #fcd34d;
			--mess-badge-bg: #44403c;
			--mess-badge-text: #fbbf24;
		}

		.mess-management {
			--mess-bg: #0b1512;
			--mess-card-bg: #111c18;
			--mess-border: #1e2e28;
			--mess-accent: #10b981;
			--mess-accent-hover: #34d399;
			--mess-accent-ring: rgba(16, 185, 129, 0.2);
			--mess-input-bg: #0b1512;
			--mess-text-main: #d1fae5;
			--mess-text-muted: #6ee7b7;
			--mess-badge-bg: #1e2e28;
			--mess-badge-text: #34d399;
		}
	}
</style>
