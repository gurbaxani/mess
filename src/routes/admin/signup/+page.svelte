<script lang="ts">
	import { ArrowLeft, Mail } from 'lucide-svelte';
	import { onMount } from 'svelte';
	let email = $state('');

	onMount(() => {
		const handle = 'mess';
		const domain = 'ashwinig.com';
		email = `${handle}@${domain}`;
	});
</script>

<svelte:head>
	<title>Sign Up - Mess Menu</title>
</svelte:head>

<div class="flex min-h-[70vh] flex-col items-center justify-center p-4">
	<div class="w-full max-w-md rounded-3xl bg-base-100 p-10 text-center shadow-xl">
		<div
			class="mx-auto mb-6 flex h-24 w-24 items-center justify-center rounded-3xl bg-primary/10 text-primary transition-transform duration-300 hover:scale-105"
		>
			<Mail size={40} />
		</div>

		<h1 class="mb-4 text-3xl font-black tracking-tight text-base-content">Need an account?</h1>
		<p class="mb-8 text-base leading-relaxed font-medium text-base-content/60">
			To get an administrator account and manage the mess menu, please contact Ashwin directly.
		</p>

		<div
			class="mb-10 rounded-2xl border border-base-300 bg-base-200/50 p-6 shadow-inner transition-colors hover:border-primary/30 hover:bg-base-200"
		>
			<span class="mb-1 block text-sm font-bold tracking-wider text-base-content/50 uppercase"
				>Email</span
			>
			<a href="mailto:{email}" class="text-xl font-bold text-primary hover:underline">
				{email}
			</a>
		</div>

		<a href="/admin/login" class="btn w-full gap-2 rounded-2xl btn-ghost btn-lg">
			<ArrowLeft size={20} />
			Back to Login
		</a>
	</div>
</div>
