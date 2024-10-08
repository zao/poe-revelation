<script lang="ts">
	import type { Header } from '$lib/dat-viewer/headers';
	import type { BundleIndex } from '$lib/patchcdn/index-store';
	import type { DatFile } from 'pathofexile-dat/dat.js';
	import { onMount } from 'svelte';
	import TableCell from './TableCell.svelte';
	import { Grid, List } from 'svelte-virtual';
	import DataTable from './Table/DataTable.svelte';

	let loader;
	let index: BundleIndex;
	let db: any;
	let dirs: string[] = [];
	let dirContents: { files: string[]; dirs: string[] } = { files: [], dirs: [] };
	let fileContent: Uint8Array | null = null;
	let headers: Header[] = [];
	let currentPath: string = '';
	let datFile: DatFile | null = null;
	let rows: { [key: string]: any }[] = [];
	let searchTerm: string = '';
	let loading: boolean = false;

	let windowHeight: number = 0;

	let showBytesByColumn: boolean[] = []; // Track column toggle states

	// calculate window height and return a number in pixels
	$: if (typeof window !== 'undefined') {
		windowHeight = window.innerHeight;
	}

	function toggleColumn(index: number) {
		showBytesByColumn[index] = !showBytesByColumn[index];
	}

	$: filteredFiles = dirContents.files.filter((file) => {
		if (!searchTerm) return true;
		return file.includes(searchTerm);
	});

	onMount(async () => {
		loading = false;
		const { BundleLoader } = await import('$lib/patchcdn/cache');
		const { BundleIndex } = await import('$lib/patchcdn/index-store');
		const { DatSchemasDatabase } = await import('$lib/dat-viewer/db');

		loader = new BundleLoader();
		await loader.setPatch('3.25.1.1.3'); // Set the patch version

		index = new BundleIndex(loader);
		await index.loadIndex(); // Load index

		db = new DatSchemasDatabase(index);
		await db.fetchSchema(); // Fetch schemas

		// Load root directories
		currentPath = '';
		dirs = index.getRootDirs() || [];
		await loadDirContents(currentPath); // Initialize root directory
	});

	// Load directory contents
	async function loadDirContents(dirPath: string) {
		currentPath = dirPath; // Update current path

		const contents = index.getDirContent(dirPath) || { files: [], dirs: [] };
		// filter out the files that are not .dat64
		contents.files = contents.files.filter((file) => file.endsWith('.dat64'));
		dirContents = { dirs: contents.dirs, files: contents.files };
	}

	// Load file content
	async function loadFileContent(filePath: string) {
		loading = true;
		fileContent = await index.loadFileContent(filePath); // Load file content

		// Extract schema name
		const schemaName = filePath.replace('data/', '').replace('.dat64', '');

		// Dynamically import necessary functions
		const { readDatFile } = await import('pathofexile-dat/dat.js');
		const { analyzeDatFile } = await import('$lib/worker/interface');
		const { fromSerializedHeaders } = await import('$lib/dat-viewer/headers');

		// Parse `.dat64` file
		datFile = readDatFile(filePath, fileContent);
		const columnStats = await analyzeDatFile(datFile);

		// Find and apply headers
		const serializedHeaders = await db.findByName(schemaName);
		const headersResult = fromSerializedHeaders(serializedHeaders, columnStats, datFile);

		if (headersResult) {
			// Assign default names to unnamed headers
			headers = headersResult.headers.map((header, index) => ({
				...header,
				name: header.name ? header.name : `Unnamed ${index}`
			}));
			// console.log('Loaded file headers:', headers);

			// Time the operation of extracting rows
			// const startTime = performance.now();

			// Extract rows from the datFile using the headers
			rows = (await extractRows(datFile, headers)) as { [key: string]: any }[];
			// console.log('Extracted rows:', rows);

			// const endTime = performance.now();
			// console.log(`Extracting rows took ${endTime - startTime} milliseconds`);

			loading = false;
		} else {
			console.warn(`Invalid headers for file: ${schemaName}`);
		}
	}

	// Function to extract rows from datFile based on the headers
	async function extractRows(datFile: DatFile, headers: Header[]) {
		// Dynamically import necessary functions
		const { readColumn } = await import('pathofexile-dat/dat.js');
		const columns = await Promise.all(headers.map((header) => readColumn(header, datFile)));

		const startTime = performance.now();
		const rows: { [key: string]: any }[] = [];
		for (let i = 0; i < datFile.rowCount; i++) {
			const row: { [key: string]: any } = {};
			for (const [index, header] of headers.entries()) {
				const value = columns[index][i];
				if (header.name) {
					row[header.name] = value;
				}
			}
			rows.push(row);
		}
		const endTime = performance.now();
		console.log(`Extracting rows took ${endTime - startTime} milliseconds`);
		return rows;
	}

	// Navigate up one directory level
	function goUp() {
		if (currentPath) {
			const parts = currentPath.split('/');
			parts.pop();
			loadDirContents(parts.join('/'));
		}
	}
</script>

<section class="w-1/4">
	<input
		class="input"
		bind:value={searchTerm}
		title="Input (search)"
		type="search"
		placeholder="Filter files..."
	/>
	<hr class="mb-2" />
	{#if currentPath}
		<button type="button" class="btn variant-outline-primary w-full my-2" on:click={goUp}
			>Go Up</button
		>
	{/if}
	<div class="file-list-container" style="position: relative; max-height: 400px; overflow: auto;">
		<List
			itemCount={dirContents.dirs.length}
			itemSize={40}
			height={'30vh'}
			scrollBehavior="smooth"
			overScan={1}
		>
			<div slot="item" let:index let:style {style} class="list-item">
				<button
					type="button"
					class="btn variant-ghost-primary w-full truncate"
					on:click={() => loadDirContents(dirContents.dirs[index])}
				>
					{dirContents.dirs[index]}
				</button>
			</div>
		</List>
	</div>

	<div class=" flex-col">
		<div style="gap: 1rem;">
			<!-- Directory Contents and Files -->
			<div style="flex: 3;">
				<hr class="mb-2" />
				<!-- Adjust the outer container to hold the virtual list properly -->
				<div class="file-list-container" style="position: relative; overflow: auto;">
					<List
						itemCount={filteredFiles.length}
						itemSize={40}
						height={'59vh'}
						scrollBehavior="smooth"
						overScan={1}
					>
						<div slot="item" let:index let:style {style} class="list-item">
							<button
								type="button"
								class="btn variant-ghost-primary w-full"
								on:click={() => loadFileContent(`${filteredFiles[index]}`)}
							>
								<p class="truncate">
									{filteredFiles[index].split('/').pop()}
								</p>
							</button>
						</div>
					</List>
				</div>
			</div>
		</div>
	</div>
</section>

<section class=" w-3/4 max-h-screen">
	<!-- Display the data rows -->
	<!-- 
      We needed to fix the table being painted in the DOM twice.
      And we did fix it. 
      
      "Just dont paint the table twice 4head" - OneRobotBoii 2024
  -->
	<!-- Virtualized Grid replacing the table -->
	{#if !loading}
		<!-- <Grid
			height={windowHeight}
			itemCount={rows.length * (headers.length + 1)}
			itemHeight={50}
			itemWidth={250}
			columnCount={headers.length + 1}
			overScan={10}
		>
			<div slot="header" class="grid grid-flow-col text-center">
				{#if headers.length > 0}
					<div class="p-2" style=" height: 50px; width: 250px;">#</div>
				{/if}
				{#each headers as header, index}

					<div
						class="p-2"
						style=" height: 50px; width: 250px;"
						on:click={() => toggleColumn(index)}
					>
						<TableCell
							value={header.name}
							showBytes={showBytesByColumn[index]}
							bytes={header.name}
						/>
					</div>
				{/each}
			</div>
			<div
				slot="item"
				let:rowIndex
				let:columnIndex
				let:style
				{style}
				class="overflow-hidden text-ellipsis my-12"
			>
				{#if columnIndex === 0}
					<div class="p-2 text-center" style=" height: 50px; width: 250px;">
						{rowIndex + 1}
					</div>
				{:else}
					<TableCell
						value={rows[rowIndex][headers[columnIndex - 1]?.name ?? '']}
						showBytes={showBytesByColumn[columnIndex - 1]}
						bytes={rows[rowIndex][headers[columnIndex - 1]?.name ?? '']}
					/>
				{/if}
			</div>
		</Grid> -->

   		<DataTable {rows} {headers} />
	{:else}
		<!-- Loading spinner -->
		<img
			src="/newkekclose.png"
			alt="Loading..."
			class="mx-auto rounded-full object-cover animate-spin h-24 w-24"
		/>
		<h1 class="text-center">Loading...</h1>
	{/if}
</section>

<style>
</style>