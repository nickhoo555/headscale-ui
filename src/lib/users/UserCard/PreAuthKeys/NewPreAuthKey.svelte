<script lang="ts">
	import { newPreAuthKey, getPreauthKeys } from '$lib/common/apiFunctions.svelte';
	import { PreAuthKey, User } from '$lib/common/classes';
	import { fade } from 'svelte/transition';
	import { alertStore } from '$lib/common/stores';
	let currentTime = new Date();
	let minDate = new Date(currentTime.setMinutes(currentTime.getMinutes() + 60 - currentTime.getTimezoneOffset())).toISOString().slice(0, 16);

	export let newPreAuthKeyShow = false;
	export let user = new User();
	export let keyList = [new PreAuthKey()];
	let expiry = minDate;
	let reusable = false;
	let ephemeral = false;
	let createdKey = '';
	let createdKeyInput: HTMLInputElement;
	let copySucceeded = false;

	function NewPreAuthKeyAction() {
		let formattedDate = new Date(expiry).toISOString();
		newPreAuthKey(user.id, formattedDate, reusable, ephemeral)
			.then((preAuthKey) => {
				// The full key is only returned once by the create endpoint. Keep it
				// visible until the user has had a chance to copy it.
				createdKey = preAuthKey.key;
				getPreauthKeysAction();
			})
			.catch((error) => {
				$alertStore = error;
			});
	}

	async function copyCreatedKey() {
		copySucceeded = false;

		try {
			if (!navigator.clipboard?.writeText) {
				throw new Error('Clipboard API unavailable');
			}
			await navigator.clipboard.writeText(createdKey);
		} catch (_) {
			// Clipboard API requires a secure context and may be denied by browser
			// policy. Fall back to copying the selected, visible input value.
			createdKeyInput.focus();
			createdKeyInput.select();
			if (!document.execCommand('copy')) {
				$alertStore = 'Unable to copy automatically. Select the key and copy it manually.';
				return;
			}
		}

		copySucceeded = true;
	}

	function closeCreatedKey() {
		createdKey = '';
		copySucceeded = false;
		newPreAuthKeyShow = false;
	}

	function getPreauthKeysAction() {
		getPreauthKeys(user.id)
			.then((keys) => {
				keyList = keys;
			})
			.catch((error) => {
				$alertStore = error;
			});
	}
</script>

<div in:fade|global class="card-pending">
	{#if createdKey}
		<div class="p-3">
			<div class="alert alert-warning mb-3">
				<span>This pre-auth key is shown only once. Copy it before closing.</span>
			</div>
			<label class="block text-sm font-bold mb-2" for="created-preauth-key">Created Preauth Key</label>
			<div class="flex flex-wrap gap-2">
				<input bind:this={createdKeyInput} id="created-preauth-key" aria-label="Created pre-auth key" class="input input-bordered font-mono grow min-w-0" readonly value={createdKey} on:focus={(event) => event.currentTarget.select()} on:click={(event) => event.currentTarget.select()} />
				<button type="button" class="btn btn-sm btn-primary capitalize" on:click={copyCreatedKey}>
					{copySucceeded ? 'Copied!' : 'Copy Key'}
				</button>
				<button type="button" class="btn btn-sm btn-secondary capitalize" on:click={closeCreatedKey}>Done</button>
			</div>
		</div>
	{:else}
		<form on:submit|preventDefault={NewPreAuthKeyAction}>
			<table class="table table-compact w-full">
				<tbody>
					<tr>
						<th>Expiry:</th>
						<td><input bind:value={expiry} class="border rounded px-2" type="datetime-local" required min={minDate} /><br /></td>
					</tr>
					<tr>
						<th>Reusable:</th>
						<td>
							<input type="checkbox" bind:checked={reusable} class="checkbox checkbox-sm text-base-content" />
						</td>
					</tr>
					<tr>
						<th>Ephemeral:</th>
						<td>
							<input type="checkbox" bind:checked={ephemeral} class="checkbox checkbox-sm text-base-content" />
						</td>
					</tr>
				</tbody>
			</table>
			<button class="btn btn-sm m-3 btn-primary capitalize">Create Preauth Key</button>
			<button
				on:click={() => {
					newPreAuthKeyShow = false;
				}}
				type="button"
				class="btn btn-sm m-1 btn-secondary capitalize">Cancel</button
			>
		</form>
	{/if}
</div>
