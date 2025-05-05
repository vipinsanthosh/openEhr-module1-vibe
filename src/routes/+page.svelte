<script>
	import { onMount } from "svelte";
	import webtemplate from "$lib/utils/vitals_webtemplate.json";

	const BASE_URL = "https://openehr-bootcamp.medblocks.com/ehrbase";
	const OPENEHR_BASE_URL = `${BASE_URL}/rest/openehr/v1`;
	const TEMPLATE_ID = "Basic_Vitals_Dominique.v2";
	const EHR_ID = "059c35b6-c6ab-4d30-be30-d62c33c1cdd9";

	let form;
	let compositions = [];
	let selectedComposition = null;
	let error = null;
	let success = null;

	onMount(() => {
		form = document.getElementById("form");
		form.webTemplate = webtemplate;
		loadCompositions();
	});

	async function loadCompositions() {
		try {
			const response = await fetch(`${OPENEHR_BASE_URL}/query/aql`, {
				method: "POST",
				headers: {
					Accept: "application/json",
					"Content-Type": "application/json",
				},
				body: JSON.stringify({
					q: `SELECT c/uid/value from EHR e CONTAINS COMPOSITION c WHERE c/archetype_details/template_id/value ='${TEMPLATE_ID}' AND e/ehr_id/value='${EHR_ID}'`,
				}),
			});

			if (!response.ok) throw new Error("Failed to load compositions");

			const data = await response.json();
			compositions = data.rows || [];
		} catch (err) {
			error = "Failed to load compositions: " + err.message;
		}
	}

	async function handleSubmit() {
		try {
			const composition = form.getComposition();
			const url = selectedComposition
				? `${OPENEHR_BASE_URL}/ehr/${EHR_ID}/composition/${selectedComposition}?templateId=${TEMPLATE_ID}`
				: `${OPENEHR_BASE_URL}/ehr/${EHR_ID}/composition?templateId=${TEMPLATE_ID}`;

			const response = await fetch(url, {
				method: selectedComposition ? "PUT" : "POST",
				headers: {
					Accept: "application/openehr.wt.flat.schema+json",
					"Content-Type": "application/openehr.wt.flat.schema+json",
					Prefer: "return=representation",
				},
				body: JSON.stringify(composition),
			});

			if (!response.ok) throw new Error("Failed to save composition");

			success = "Composition saved successfully";
			form.clear();
			selectedComposition = null;
			await loadCompositions();
		} catch (err) {
			error = "Failed to save composition: " + err.message;
		}
	}

	async function handleDelete(compositionId) {
		if (!confirm("Are you sure you want to delete this composition?")) return;

		try {
			const response = await fetch(
				`${OPENEHR_BASE_URL}/ehr/${EHR_ID}/composition/${compositionId}`,
				{
					method: "DELETE",
					headers: {
						Accept: "application/json",
						"Content-Type": "application/json",
						Prefer: "return=representation",
					},
				}
			);

			if (!response.ok) throw new Error("Failed to delete composition");

			success = "Composition deleted successfully";
			await loadCompositions();
		} catch (err) {
			error = "Failed to delete composition: " + err.message;
		}
	}

	function handleEdit(compositionId) {
		selectedComposition = compositionId;
		// TODO: Load composition data into form
	}
</script>

<div class="container mx-auto p-4">
	<h1 class="text-2xl font-bold mb-4">Vital Signs</h1>

	{#if error}
		<div
			class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4"
		>
			{error}
		</div>
	{/if}

	{#if success}
		<div
			class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded mb-4"
		>
			{success}
		</div>
	{/if}

	<div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
		<mb-auto-form id="form"></mb-auto-form>
		<button
			class="bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded mt-4"
			on:click={handleSubmit}
		>
			{selectedComposition ? "Update" : "Save"} Vital Signs
		</button>
	</div>

	<div class="mt-8">
		<h2 class="text-xl font-bold mb-4">Vital Signs History</h2>
		<div class="grid gap-4">
			{#each compositions as composition}
				<div
					class="bg-white shadow-md rounded p-4 flex justify-between items-center"
				>
					<span>Composition ID: {composition[0]}</span>
					<div class="space-x-2">
						<button
							class="bg-yellow-500 hover:bg-yellow-700 text-white font-bold py-2 px-4 rounded"
							on:click={() => handleEdit(composition[0])}
						>
							Edit
						</button>
						<button
							class="bg-red-500 hover:bg-red-700 text-white font-bold py-2 px-4 rounded"
							on:click={() => handleDelete(composition[0])}
						>
							Delete
						</button>
					</div>
				</div>
			{/each}
		</div>
	</div>
</div>

<style>
	:global(mb-auto-form) {
		width: 100%;
	}
</style>
