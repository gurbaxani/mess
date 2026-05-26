<script lang="ts">
	import { pb } from '$lib/pb';
	import { User, Lock, LogIn, ArrowRight } from 'lucide-svelte';

	let email = $state('');
	let password = $state('');
	let errorMsg = $state<string | null>(null);
	let isLoading = $state(false);

	const handleLogin = async (e: Event) => {
		e.preventDefault();
		isLoading = true;
		errorMsg = null;

		try {
			await pb.collection('users').authWithPassword(email, password);
			window.location.href = '/admin'; // Redirect to admin dashboard
		} catch (err) {
			errorMsg = err instanceof Error ? err.message : 'Failed to login';
		} finally {
			isLoading = false;
		}
	};
</script>

<svelte:head>
	<title>Admin Login - Mess Menu</title>
</svelte:head>

<div class="flex flex-col items-center justify-center px-4 py-16">
	<div class="w-full max-w-md rounded-3xl bg-base-100 p-8 shadow-xl">
		<div class="mb-8 text-center">
			<h1 class="text-3xl font-black tracking-tight text-base-content">Welcome Back</h1>
			<p class="mt-2 text-sm text-base-content/60">
				Enter your credentials to access the dashboard
			</p>
		</div>

		{#if errorMsg}
			<div class="mb-6 alert rounded-xl text-sm alert-error">
				<span>{errorMsg}</span>
			</div>
		{/if}

		<form onsubmit={handleLogin} class="space-y-5">
			<div class="form-control">
				<label class="label" for="email">
					<span class="label-text font-semibold">Email</span>
				</label>
				<div class="relative flex items-center">
					<span class="pointer-events-none absolute left-4 text-base-content/40">
						<User size={18} />
					</span>
					<input
						id="email"
						type="email"
						class="input-bordered input w-full rounded-xl pl-12 focus-within:outline-primary"
						placeholder="admin@example.com"
						bind:value={email}
						required
					/>
				</div>
			</div>

			<div class="form-control">
				<label class="label" for="password">
					<span class="label-text font-semibold">Password</span>
				</label>
				<div class="relative flex items-center">
					<span class="pointer-events-none absolute left-4 text-base-content/40">
						<Lock size={18} />
					</span>
					<input
						id="password"
						type="password"
						class="input-bordered input w-full rounded-xl pl-12 focus-within:outline-primary"
						placeholder="••••••••"
						bind:value={password}
						required
					/>
				</div>
			</div>

			<div class="pt-2">
				<button type="submit" class="btn w-full rounded-xl btn-primary" disabled={isLoading}>
					{#if isLoading}
						<span class="loading loading-sm loading-spinner"></span>
						Authenticating...
					{:else}
						<LogIn size={18} />
						Sign In
					{/if}
				</button>
			</div>
		</form>

		<div class="mt-8 text-center text-sm text-base-content/60">
			Don't have an account?
			<a
				href="/admin/signup"
				class="group inline-flex items-center gap-1 font-bold text-primary transition-colors hover:text-primary/80"
			>
				Create one
				<ArrowRight size={14} class="transition-transform group-hover:translate-x-1" />
			</a>
		</div>
	</div>
</div>
