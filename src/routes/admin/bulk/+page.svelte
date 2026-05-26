<script lang="ts">
	import { onMount } from 'svelte';
	import { pb } from '$lib/pb';
	import type { MenuResponse } from '../../../../pocketbase-types';
	import {
		ArrowLeft,
		Download,
		UploadCloud,
		FileSpreadsheet,
		AlertCircle,
		CheckCircle2,
		Trash2,
		Loader2
	} from 'lucide-svelte';

	interface ParsedRow {
		date: string;
		mess_name: 'Polytechnic' | 'Management';
		breakfast: string;
		lunch: string;
		snacks: string;
		dinner: string;
		rowNum: number;
	}

	interface ValidationError {
		rowNum: number;
		message: string;
	}

	let isLoading = $state(true);
	let isDragging = $state(false);
	let fileInput = $state<HTMLInputElement | null>(null);
	let selectedFile = $state<File | null>(null);

	let parsedRows = $state<ParsedRow[]>([]);
	let errors = $state<ValidationError[]>([]);

	let isImporting = $state(false);
	let importProgress = $state(0);
	let importStatusText = $state('');

	let errorMsg = $state<string | null>(null);
	let successMsg = $state<string | null>(null);

	const messes = ['Polytechnic', 'Management'] as const;

	onMount(() => {
		if (!pb.authStore.isValid) {
			window.location.href = '/admin/login';
			return;
		}
		isLoading = false;
	});

	// CSV Template generator
	const templateCSV = [
		'date,mess_name,breakfast,lunch,snacks,dinner',
		'2026-06-01,Polytechnic,"Idli, Sambar, Coconut Chutney","Veg Biryani, Raita, Pickle, Papad","Samosa, Hot Chai","Roti, Paneer Butter Masala, Rice, Dal"',
		'2026-06-01,Management,"Masala Dosa, Tomato Chutney, Sambar","Jeera Rice, Dal Fry, Roti, Aloo Gobi","Veg Cutlet, Coffee","Roti, Butter Chicken, Rice, Dal"'
	].join('\n');

	const downloadTemplate = () => {
		const blob = new Blob([templateCSV], { type: 'text/csv;charset=utf-8;' });
		const url = URL.createObjectURL(blob);
		const link = document.createElement('a');
		link.href = url;
		link.setAttribute('download', 'mess_menu_template.csv');
		document.body.appendChild(link);
		link.click();
		document.body.removeChild(link);
	};

	// Parse custom CSV line handling quotes
	function parseCSVLine(text: string): string[] {
		const result: string[] = [];
		let cell = '';
		let inQuotes = false;
		for (let i = 0; i < text.length; i++) {
			const char = text[i];
			if (char === '"') {
				inQuotes = !inQuotes;
			} else if (char === ',' && !inQuotes) {
				result.push(cell.trim());
				cell = '';
			} else {
				cell += char;
			}
		}
		result.push(cell.trim());
		return result;
	}

	const handleFile = (file: File) => {
		if (!file.name.endsWith('.csv')) {
			errorMsg = 'Please select a valid CSV file.';
			setTimeout(() => (errorMsg = null), 4000);
			return;
		}

		selectedFile = file;
		parsedRows = [];
		errors = [];

		const reader = new FileReader();
		reader.onload = (e) => {
			const text = e.target?.result as string;
			if (!text) return;

			const lines = text.split(/\r?\n/).filter((line) => line.trim() !== '');
			if (lines.length <= 1) {
				errors.push({ rowNum: 1, message: 'CSV file is empty or missing headers.' });
				return;
			}

			// Parse headers
			const headers = parseCSVLine(lines[0]);
			const headerIndex = {
				date: -1,
				mess_name: -1,
				breakfast: -1,
				lunch: -1,
				snacks: -1,
				dinner: -1
			};

			headers.forEach((h, idx) => {
				const key = h.toLowerCase().trim() as keyof typeof headerIndex;
				if (key in headerIndex) {
					headerIndex[key] = idx;
				}
			});

			// Validate header existence
			const requiredKeys: (keyof typeof headerIndex)[] = ['date', 'mess_name'];
			const missingHeaders: string[] = [];
			requiredKeys.forEach((k) => {
				if (headerIndex[k] === -1) missingHeaders.push(k);
			});

			if (missingHeaders.length > 0) {
				errors.push({
					rowNum: 1,
					message: `Missing required header columns: ${missingHeaders.join(', ')}`
				});
				return;
			}

			// Parse data rows
			const tempRows: ParsedRow[] = [];
			for (let i = 1; i < lines.length; i++) {
				const line = lines[i];
				const cells = parseCSVLine(line);
				const rowNum = i + 1;

				const dateVal = headerIndex.date !== -1 ? cells[headerIndex.date] : '';
				const messNameRaw = headerIndex.mess_name !== -1 ? cells[headerIndex.mess_name] : '';
				const breakfast =
					headerIndex.breakfast !== -1 && headerIndex.breakfast < cells.length
						? cells[headerIndex.breakfast]
						: '';
				const lunch =
					headerIndex.lunch !== -1 && headerIndex.lunch < cells.length
						? cells[headerIndex.lunch]
						: '';
				const snacks =
					headerIndex.snacks !== -1 && headerIndex.snacks < cells.length
						? cells[headerIndex.snacks]
						: '';
				const dinner =
					headerIndex.dinner !== -1 && headerIndex.dinner < cells.length
						? cells[headerIndex.dinner]
						: '';

				// Validations
				if (!dateVal) {
					errors.push({ rowNum, message: 'Date field is empty.' });
					continue;
				}

				// Validate Date format YYYY-MM-DD
				const dateRegex = /^\d{4}-\d{2}-\d{2}$/;
				if (!dateRegex.test(dateVal)) {
					errors.push({ rowNum, message: `Invalid date format "${dateVal}". Must be YYYY-MM-DD.` });
					continue;
				}

				const parsedTimestamp = Date.parse(dateVal);
				if (isNaN(parsedTimestamp)) {
					errors.push({ rowNum, message: `Invalid date value "${dateVal}".` });
					continue;
				}

				// Validate Mess Name
				if (!messNameRaw) {
					errors.push({ rowNum, message: 'Mess name is empty.' });
					continue;
				}

				const lowerMess = messNameRaw.toLowerCase().trim();
				let mess_name: 'Polytechnic' | 'Management' | null = null;
				if (lowerMess === 'polytechnic') mess_name = 'Polytechnic';
				else if (lowerMess === 'management') mess_name = 'Management';

				if (!mess_name) {
					errors.push({
						rowNum,
						message: `Invalid mess name "${messNameRaw}". Must be "Polytechnic" or "Management".`
					});
					continue;
				}

				tempRows.push({
					date: dateVal,
					mess_name,
					breakfast: breakfast || '',
					lunch: lunch || '',
					snacks: snacks || '',
					dinner: dinner || '',
					rowNum
				});
			}

			parsedRows = tempRows;
		};
		reader.readAsText(file);
	};

	const handleDrop = (e: DragEvent) => {
		e.preventDefault();
		isDragging = false;
		if (e.dataTransfer?.files && e.dataTransfer.files.length > 0) {
			handleFile(e.dataTransfer.files[0]);
		}
	};

	const handleDragOver = (e: DragEvent) => {
		e.preventDefault();
		isDragging = true;
	};

	const handleDragLeave = () => {
		isDragging = false;
	};

	const handleFileInputChange = (e: Event) => {
		const target = e.target as HTMLInputElement;
		if (target.files && target.files.length > 0) {
			handleFile(target.files[0]);
		}
	};

	const clearFile = () => {
		selectedFile = null;
		parsedRows = [];
		errors = [];
		if (fileInput) fileInput.value = '';
	};

	const handleImport = async () => {
		if (parsedRows.length === 0 || errors.length > 0) return;

		isImporting = true;
		importProgress = 0;
		errorMsg = null;
		successMsg = null;

		try {
			// Extract all unique dates
			const dates = Array.from(new Set(parsedRows.map((r) => r.date)));

			// Build filter query for existing records
			importStatusText = 'Checking database for existing menus...';
			const filterStr = dates.map((d) => `date = "${d}"`).join(' || ');
			const existingRecords = await pb.collection('menu').getFullList<MenuResponse>({
				filter: filterStr
			});

			const existingMap = new Map<string, string>(); // Key: date_messName -> ID
			existingRecords.forEach((r) => {
				existingMap.set(`${r.date}_${r.mess_name}`, r.id);
			});

			// Loop through and update or create
			for (let i = 0; i < parsedRows.length; i++) {
				const row = parsedRows[i];
				const key = `${row.date}_${row.mess_name}`;
				const existingId = existingMap.get(key);

				importStatusText = `Importing row ${i + 1} of ${parsedRows.length}: ${row.mess_name} (${row.date})...`;
				importProgress = Math.round(((i + 1) / parsedRows.length) * 100);

				if (existingId) {
					// Update
					await pb.collection('menu').update(existingId, {
						breakfast: row.breakfast,
						lunch: row.lunch,
						snacks: row.snacks,
						dinner: row.dinner
					});
				} else {
					// Create
					await pb.collection('menu').create({
						date: row.date,
						mess_name: row.mess_name,
						breakfast: row.breakfast,
						lunch: row.lunch,
						snacks: row.snacks,
						dinner: row.dinner
					});
				}
			}

			successMsg = `Awesome! Successfully imported ${parsedRows.length} menus.`;
			clearFile();
			setTimeout(() => (successMsg = null), 4000);
		} catch (err) {
			errorMsg = err instanceof Error ? err.message : 'Failed to import menus.';
		} finally {
			isImporting = false;
		}
	};
</script>

<svelte:head>
	<title>Bulk Upload - Mess Menu</title>
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

		<header class="mb-8 flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
			<div>
				<p class="mb-1 text-xs font-bold tracking-wider text-primary uppercase sm:text-sm">
					Admin Tools
				</p>
				<h1
					class="font-display text-3xl font-extrabold tracking-tight text-base-content sm:text-4xl"
				>
					Bulk Menu Upload 🗓️
				</h1>
				<p class="mt-1 text-sm text-base-content/60">
					Upload a CSV file containing menus for multiple days at once.
				</p>
			</div>

			<button
				type="button"
				class="inline-flex items-center gap-2 self-start rounded-2xl border-2 border-primary/20 bg-primary/5 px-5 py-3 text-sm font-bold text-primary transition-all hover:bg-primary hover:text-primary-content hover:shadow-md active:scale-95 sm:self-auto"
				onclick={downloadTemplate}
			>
				<Download size={16} />
				Template CSV
			</button>
		</header>

		{#if isLoading}
			<div class="flex flex-col items-center justify-center py-20">
				<Loader2 size={40} class="animate-spin text-primary" />
				<p class="mt-4 text-sm font-semibold text-base-content/60">Loading session...</p>
			</div>
		{:else}
			<div class="space-y-8">
				<!-- Upload Card Container -->
				<div
					class="mx-auto w-full max-w-2xl rounded-[24px] border border-base-300 bg-base-100 p-6 shadow-xl"
				>
					{#if !selectedFile}
						<div
							role="button"
							tabindex="0"
							class="group flex cursor-pointer flex-col items-center justify-center rounded-2xl border-2 border-dashed border-base-300 bg-base-200/30 px-6 py-12 text-center transition-all duration-300 hover:border-primary/50 hover:bg-base-200/60"
							ondragover={handleDragOver}
							ondragleave={handleDragLeave}
							ondrop={handleDrop}
							onclick={() => fileInput?.click()}
							onkeydown={(e) => e.key === 'Enter' && fileInput?.click()}
						>
							<input
								type="file"
								accept=".csv"
								class="hidden"
								bind:this={fileInput}
								onchange={handleFileInputChange}
							/>
							<div
								class="mb-4 flex h-14 w-14 items-center justify-center rounded-full bg-primary/10 text-primary transition-transform group-hover:scale-110"
							>
								<UploadCloud size={26} />
							</div>
							<h3 class="font-display text-lg font-bold text-base-content">
								Drag & drop your CSV file here
							</h3>
							<p class="mt-1 text-sm text-base-content/50">or click to browse your computer</p>
							<span
								class="mt-4 inline-flex items-center gap-1.5 rounded-full bg-primary/10 px-3 py-1 text-xs font-bold text-primary"
							>
								Supports CSV up to 500 rows
							</span>
						</div>
					{:else}
						<!-- Selected File Sheet -->
						<div
							class="flex items-center justify-between rounded-xl border border-primary/20 bg-primary/5 p-4 shadow-inner"
						>
							<div class="flex items-center gap-3">
								<div
									class="flex h-12 w-12 items-center justify-center rounded-xl bg-primary/10 text-primary"
								>
									<FileSpreadsheet size={22} />
								</div>
								<div>
									<p class="leading-snug font-bold text-base-content">{selectedFile.name}</p>
									<p class="text-xs font-medium text-base-content/50">
										{(selectedFile.size / 1024).toFixed(1)} KB &bull; Parsed successfully
									</p>
								</div>
							</div>
							<button
								type="button"
								class="flex h-10 w-10 items-center justify-center rounded-xl border border-error/15 bg-error/5 text-error transition-all hover:bg-error hover:text-white hover:shadow-lg hover:shadow-error/15 active:scale-95"
								onclick={clearFile}
								disabled={isImporting}
							>
								<Trash2 size={18} />
							</button>
						</div>
					{/if}
				</div>

				<!-- Import Progress Bar -->
				{#if isImporting}
					<div
						class="mx-auto w-full max-w-2xl rounded-[22px] border border-primary/20 bg-primary/5 p-6 shadow-xl"
					>
						<div class="mb-3 flex items-center justify-between">
							<div class="flex items-center gap-2">
								<Loader2 size={16} class="animate-spin text-primary" />
								<span class="text-sm font-bold text-primary">{importStatusText}</span>
							</div>
							<span class="text-sm font-black text-primary">{importProgress}%</span>
						</div>
						<div class="h-3 w-full overflow-hidden rounded-full bg-primary/15">
							<div
								class="h-full bg-primary transition-all duration-300"
								style="width: {importProgress}%"
							></div>
						</div>
					</div>
				{/if}

				<!-- Errors List -->
				{#if errors.length > 0}
					<div
						class="mx-auto w-full max-w-2xl rounded-[24px] border border-error/30 bg-error/5 p-6 text-error"
					>
						<div class="font-display mb-4 flex items-center gap-2 text-lg font-extrabold">
							<AlertCircle size={22} class="shrink-0 text-error" />
							<h3>CSV Review: {errors.length} Correction{errors.length > 1 ? 's' : ''} Needed</h3>
						</div>
						<div class="max-h-60 space-y-3 overflow-y-auto pr-2">
							{#each errors as err (err.rowNum + '-' + err.message)}
								<div
									class="flex items-start gap-3 rounded-xl bg-error/10 p-3 text-xs font-semibold"
								>
									<span class="shrink-0 rounded bg-error/20 px-2 py-0.5 font-bold text-error"
										>Row {err.rowNum}</span
									>
									<span class="mt-0.5 text-error-content/90">{err.message}</span>
								</div>
							{/each}
						</div>
						<p class="mt-4 border-t border-error/10 pt-4 text-xs font-semibold text-error/80">
							Please review the errors above, correct your CSV file, and try uploading again.
						</p>
					</div>
				{/if}

				<!-- Parsed Rows Preview Tickets -->
				{#if parsedRows.length > 0 && errors.length === 0}
					<div class="space-y-6">
						<div class="flex flex-col gap-4 sm:flex-row sm:items-center sm:justify-between">
							<div>
								<h2 class="font-display text-2xl font-black text-base-content">
									Meal Ticket Preview ({parsedRows.length})
								</h2>
								<p class="text-xs font-semibold text-base-content/50">
									Confirm your menus below. Click 'Import' to write items to the database.
								</p>
							</div>

							<button
								type="button"
								class="flex items-center justify-center gap-2 rounded-2xl bg-success px-6 py-4 font-bold text-success-content shadow-lg shadow-success/15 transition-all hover:bg-success/90 hover:shadow-xl active:scale-95 disabled:opacity-50"
								onclick={handleImport}
								disabled={isImporting}
							>
								{#if isImporting}
									<Loader2 size={18} class="animate-spin" />
									Importing...
								{:else}
									Import to Database
								{/if}
							</button>
						</div>

						<div
							class="grid max-h-[520px] gap-6 overflow-y-auto rounded-3xl border border-base-200 bg-base-300/10 p-1 sm:grid-cols-2 lg:grid-cols-3"
						>
							{#each parsedRows as row (row.rowNum + '-' + row.date + '-' + row.mess_name)}
								<div
									class="mess-{row.mess_name.toLowerCase()} relative flex flex-col overflow-hidden rounded-2xl border border-[var(--mess-border)] bg-[var(--mess-card-bg)] text-[var(--mess-text-main)] shadow-md"
								>
									<!-- Ticket notches relative to absolute anchor -->
									<div class="relative h-0">
										<div
											class="absolute -top-2.5 -left-2.5 h-5 w-5 rounded-full border border-[var(--mess-border)] bg-base-200"
										></div>
										<div
											class="absolute -top-2.5 -right-2.5 h-5 w-5 rounded-full border border-[var(--mess-border)] bg-base-200"
										></div>
									</div>

									<!-- Ticket Header -->
									<div
										class="flex items-center justify-between border-b border-dashed border-[var(--mess-border)] bg-[var(--mess-bg)] px-5 py-4"
									>
										<div>
											<span class="text-[10px] font-bold text-[var(--mess-text-muted)]/60"
												>Row {row.rowNum}</span
											>
											<p class="font-display text-sm font-extrabold">{row.date}</p>
										</div>
										<span
											class="badge border-none bg-[var(--mess-badge-bg)] badge-sm font-bold text-[var(--mess-badge-text)]"
										>
											{row.mess_name}
										</span>
									</div>

									<!-- Ticket Body -->
									<div class="flex-grow space-y-3 p-5 text-xs font-medium">
										<div class="space-y-1">
											<span
												class="text-[9px] font-bold tracking-wider text-[var(--mess-text-muted)]/60 uppercase"
												>Breakfast</span
											>
											<p class="truncate text-base-content" title={row.breakfast}>
												{row.breakfast || '-'}
											</p>
										</div>
										<div class="space-y-1">
											<span
												class="text-[9px] font-bold tracking-wider text-[var(--mess-text-muted)]/60 uppercase"
												>Lunch</span
											>
											<p class="truncate text-base-content" title={row.lunch}>{row.lunch || '-'}</p>
										</div>
										<div class="space-y-1">
											<span
												class="text-[9px] font-bold tracking-wider text-[var(--mess-text-muted)]/60 uppercase"
												>Snacks</span
											>
											<p class="truncate text-base-content" title={row.snacks}>
												{row.snacks || '-'}
											</p>
										</div>
										<div class="space-y-1">
											<span
												class="text-[9px] font-bold tracking-wider text-[var(--mess-text-muted)]/60 uppercase"
												>Dinner</span
											>
											<p class="truncate text-base-content" title={row.dinner}>
												{row.dinner || '-'}
											</p>
										</div>
									</div>
								</div>
							{/each}
						</div>
					</div>
				{/if}
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
