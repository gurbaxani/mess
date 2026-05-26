<script lang="ts">
	import { onMount } from 'svelte';
	import { pb } from '$lib/pb';
	import {
		ArrowLeft,
		User,
		Lock,
		Save,
		LogOut,
		Loader2,
		AlertCircle,
		CheckCircle2
	} from 'lucide-svelte';

	let isLoading = $state(true);

	let displayName = $state('');
	let email = $state('');

	let oldPassword = $state('');
	let newPassword = $state('');
	let confirmPassword = $state('');

	let isSavingName = $state(false);
	let isSavingPassword = $state(false);

	let errorMsg = $state<string | null>(null);
	let successMsg = $state<string | null>(null);

	onMount(() => {
		if (!pb.authStore.isValid || !pb.authStore.model) {
			window.location.href = '/admin/login';
			return;
		}

		displayName = pb.authStore.model.name || '';
		email = pb.authStore.model.email || '';
		isLoading = false;
	});

	const handleUpdateName = async (e: Event) => {
		e.preventDefault();
		isSavingName = true;
		errorMsg = null;
		successMsg = null;

		try {
			if (!pb.authStore.model?.id) throw new Error('Session has expired. Please log in.');
			const record = await pb.collection('users').update(pb.authStore.model.id, {
				name: displayName
			});
			pb.authStore.model.name = record.name;
			successMsg = 'Profile name updated successfully!';
			setTimeout(() => (successMsg = null), 4000);
		} catch (err) {
			errorMsg = err instanceof Error ? err.message : 'Failed to update name.';
		} finally {
			isSavingName = false;
		}
	};

	const handleUpdatePassword = async (e: Event) => {
		e.preventDefault();
		if (newPassword !== confirmPassword) {
			errorMsg = 'New passwords do not match.';
			return;
		}

		isSavingPassword = true;
		errorMsg = null;
		successMsg = null;

		try {
			if (!pb.authStore.model?.id) throw new Error('Session has expired. Please log in.');
			await pb.collection('users').update(pb.authStore.model.id, {
				password: newPassword,
				passwordConfirm: confirmPassword,
				oldPassword: oldPassword
			});

			successMsg = 'Password updated successfully!';
			oldPassword = '';
			newPassword = '';
			confirmPassword = '';
			setTimeout(() => (successMsg = null), 4000);
		} catch (err) {
			errorMsg = err instanceof Error ? err.message : 'Failed to update password.';
		} finally {
			isSavingPassword = false;
		}
	};

	const handleLogout = () => {
		pb.authStore.clear();
		window.location.href = '/admin/login';
	};
</script>

<svelte:head>
	<title>Profile Settings - Mess Menu</title>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:opsz,wght@12..96,200..800&family=Lexend:wght@300..700&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="px-4 py-8 sm:px-6 lg:px-8 lg:py-12">
	<div class="mx-auto max-w-4xl">
		<!-- Back link -->
		<div class="mb-6">
			<a
				href="/admin"
				class="group inline-flex items-center gap-1.5 font-bold text-primary transition-colors hover:text-primary/85"
			>
				<ArrowLeft size={16} class="transition-transform group-hover:-translate-x-1" />
				Back to Dashboard
			</a>
		</div>

		<header class="mb-10">
			<p class="mb-1.5 text-xs font-bold tracking-wider text-primary uppercase sm:text-sm">
				Admin settings
			</p>
			<h1 class="font-display text-4xl font-extrabold tracking-tight text-base-content sm:text-5xl">
				Profile Settings ⚙️
			</h1>
			<p class="mt-2 text-sm text-base-content/60">
				Manage your display details and account security settings.
			</p>
		</header>

		{#if isLoading}
			<div class="flex flex-col items-center justify-center py-20">
				<Loader2 size={40} class="animate-spin text-primary" />
				<p class="mt-4 text-sm font-semibold text-base-content/60">Loading profile...</p>
			</div>
		{:else}
			<div class="grid gap-8 md:grid-cols-3">
				<!-- Left Column: Admin ID Lanyard Badge -->
				<div class="md:col-span-1">
					<div
						class="relative flex flex-col items-center rounded-3xl border border-base-300 bg-base-100 p-6 pt-10 text-center shadow-xl"
					>
						<!-- Lanyard Slot -->
						<div
							class="absolute -top-3 left-1/2 h-4 w-12 -translate-x-1/2 rounded-full border border-base-300 bg-base-200"
						>
							<div class="mx-auto mt-0.5 h-1.5 w-6 rounded-full bg-base-300 shadow-inner"></div>
						</div>

						<!-- Avatar Circle -->
						<div
							class="mb-4 flex h-24 w-24 items-center justify-center rounded-full border-2 border-primary/20 bg-primary/10 text-4xl shadow-inner select-none"
						>
							🧑‍🍳
						</div>

						<!-- Name & Email Details -->
						<h3 class="font-display text-2xl leading-tight font-black text-base-content">
							{displayName || 'Admin'}
						</h3>
						<p class="mt-1 text-sm font-semibold break-all text-base-content/50">
							{email}
						</p>

						<!-- Role Badge -->
						<span
							class="mt-4 inline-flex items-center gap-1.5 rounded-full border border-success/10 bg-success/15 px-3 py-1 text-xs font-bold text-success"
						>
							<span class="h-1.5 w-1.5 rounded-full bg-success"></span>
							Authorized Admin
						</span>
					</div>
				</div>

				<!-- Right Column: Settings Forms Panel -->
				<div class="space-y-8 md:col-span-2">
					<div
						class="space-y-8 rounded-3xl border border-base-300 bg-base-100 p-6 shadow-xl sm:p-8"
					>
						<!-- Section 1: General settings -->
						<section class="space-y-5">
							<div class="flex items-center gap-2 border-b border-base-200 pb-2">
								<div
									class="flex h-7 w-7 items-center justify-center rounded-lg bg-primary/10 text-primary"
								>
									<User size={14} />
								</div>
								<h2 class="font-display text-lg font-bold text-base-content">General Settings</h2>
							</div>

							<form onsubmit={handleUpdateName} class="space-y-4">
								<div class="form-control">
									<label class="label pb-1.5" for="emailInput">
										<span class="label-text font-bold text-base-content/50">Email Address</span>
									</label>
									<input
										id="emailInput"
										type="email"
										class="input-bordered input w-full cursor-not-allowed rounded-2xl border-2 border-base-200 bg-base-200/50 px-4 py-3 font-medium text-base-content/50 focus:outline-none"
										value={email}
										readonly
									/>
								</div>

								<div class="form-control">
									<label class="label pb-1.5" for="nameInput">
										<span class="label-text font-bold text-base-content/75">Display Name</span>
									</label>
									<input
										id="nameInput"
										type="text"
										class="input-bordered input w-full rounded-2xl border-2 border-base-300 bg-base-100 px-4 py-3 font-medium text-base-content transition-all focus:border-primary focus:ring-4 focus:ring-primary/10 focus:outline-none"
										bind:value={displayName}
										placeholder="Your Name"
										required
									/>
								</div>

								<div class="pt-2">
									<button
										type="submit"
										class="hover:bg-primary-focus flex inline-flex w-full items-center justify-center gap-2 rounded-2xl bg-primary px-6 py-4 font-bold text-white transition-all hover:shadow-lg hover:shadow-primary/15 active:scale-98 disabled:opacity-50 sm:w-auto"
										disabled={isSavingName}
									>
										{#if isSavingName}
											<Loader2 size={16} class="animate-spin" />
											Saving...
										{:else}
											<Save size={16} />
											Save Name
										{/if}
									</button>
								</div>
							</form>
						</section>

						<!-- Dashed divider -->
						<div class="border-t-2 border-dashed border-base-200"></div>

						<!-- Section 2: Password Change -->
						<section class="space-y-5">
							<div class="flex items-center gap-2 border-b border-base-200 pb-2">
								<div
									class="flex h-7 w-7 items-center justify-center rounded-lg bg-primary/10 text-primary"
								>
									<Lock size={14} />
								</div>
								<h2 class="font-display text-lg font-bold text-base-content">Change Password</h2>
							</div>

							<form onsubmit={handleUpdatePassword} class="space-y-4">
								<div class="form-control">
									<label class="label pb-1.5" for="oldPasswordInput">
										<span class="label-text font-bold text-base-content/75">Current Password</span>
									</label>
									<input
										id="oldPasswordInput"
										type="password"
										class="input-bordered input w-full rounded-2xl border-2 border-base-300 bg-base-100 px-4 py-3 font-medium text-base-content transition-all focus:border-primary focus:ring-4 focus:ring-primary/10 focus:outline-none"
										bind:value={oldPassword}
										placeholder="••••••••"
										required
									/>
								</div>

								<div class="form-control">
									<label class="label pb-1.5" for="newPasswordInput">
										<span class="label-text font-bold text-base-content/75">New Password</span>
									</label>
									<input
										id="newPasswordInput"
										type="password"
										class="input-bordered input w-full rounded-2xl border-2 border-base-300 bg-base-100 px-4 py-3 font-medium text-base-content transition-all focus:border-primary focus:ring-4 focus:ring-primary/10 focus:outline-none"
										bind:value={newPassword}
										placeholder="••••••••"
										required
									/>
								</div>

								<div class="form-control">
									<label class="label pb-1.5" for="confirmPasswordInput">
										<span class="label-text font-bold text-base-content/75">Confirm Password</span>
									</label>
									<input
										id="confirmPasswordInput"
										type="password"
										class="input-bordered input w-full rounded-2xl border-2 border-base-300 bg-base-100 px-4 py-3 font-medium text-base-content transition-all focus:border-primary focus:ring-4 focus:ring-primary/10 focus:outline-none"
										bind:value={confirmPassword}
										placeholder="••••••••"
										required
									/>
								</div>

								<div class="pt-2">
									<button
										type="submit"
										class="hover:bg-primary-focus flex inline-flex w-full items-center justify-center gap-2 rounded-2xl bg-primary px-6 py-4 font-bold text-white transition-all hover:shadow-lg hover:shadow-primary/15 active:scale-98 disabled:opacity-50 sm:w-auto"
										disabled={isSavingPassword}
									>
										{#if isSavingPassword}
											<Loader2 size={16} class="animate-spin" />
											Saving...
										{:else}
											<Lock size={16} />
											Update Password
										{/if}
									</button>
								</div>
							</form>
						</section>
					</div>

					<!-- Danger zone logout card -->
					<div class="rounded-3xl border border-error/20 bg-error/5 p-6 shadow-md">
						<div class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
							<div>
								<h3 class="font-display text-lg font-extrabold text-error">Danger Zone</h3>
								<p class="font-sans text-xs font-medium text-error/70">
									Securely sign out of your current administration session.
								</p>
							</div>
							<button
								type="button"
								class="flex items-center gap-2 self-start rounded-2xl bg-error px-5 py-3 text-sm font-bold text-error-content shadow-lg shadow-error/15 transition-all hover:bg-error/90 hover:shadow-xl active:scale-95 sm:self-auto"
								onclick={handleLogout}
							>
								<LogOut size={16} />
								Log Out Session
							</button>
						</div>
					</div>
				</div>
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
</style>
