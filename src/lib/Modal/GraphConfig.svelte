<script lang="ts">
	import {
		dashboard,
		states,
		connection,
		lang,
		history,
		historyIndex,
		record,
		ripple
	} from '$lib/Stores';
	import { onDestroy } from 'svelte';
	import Graph from '$lib/Sidebar/Graph.svelte';
	import Select from '$lib/Components/Select.svelte';
	import ConfigButtons from '$lib/Modal/ConfigButtons.svelte';
	import InputClear from '$lib/Components/InputClear.svelte';
	import Modal from '$lib/Modal/Index.svelte';
	import { updateObj, getName } from '$lib/Utils';
	import type { GraphItem } from '$lib/Types';
	import Ripple from 'svelte-ripple';
	import Icon from '@iconify/svelte';

	export let isOpen: boolean;
	export let sel: GraphItem;

	export let demo: string | undefined = undefined;

	if (sel?.entity_id && (!sel?.entities || sel.entities.length === 0)) {
		set('entities', [{ entity_id: sel.entity_id }]);
		set('entity_id');
	}

	if (demo) {
		$history.splice($historyIndex, 1);
		set('entities', [{ entity_id: demo }]);
	}

	let name = sel?.name;
	let options: { id: string; label: string; hint?: string }[] = [];
	let stroke = sel?.stroke;
	let numberElement: HTMLInputElement;
	let entitiesInitialized = false;

	const range = {
		min: 0,
		max: 10
	};

	const periodOptions = [
		{ id: '5minute', label: $lang('period_5minute') },
		{ id: 'hour', label: $lang('period_hour') },
		{ id: 'day', label: $lang('period_day') },
		{ id: 'week', label: $lang('period_week') },
		{ id: 'month', label: $lang('period_month') }
	];

	const scaleModeOptions = [
		{ id: 'auto', label: $lang('scale_mode_auto') },
		{ id: 'zero', label: $lang('scale_mode_zero') },
		{ id: 'custom', label: $lang('scale_mode_custom') }
	];

	function minMax(key: string | number | undefined) {
		return Math.min(Math.max(parseInt(key as string), range.min), range.max);
	}

	function handleNumberRange(event: any) {
		const value = minMax(event?.target?.value);
		set('stroke', value);
		if (numberElement) numberElement.value = String(value);
	}

	function set(key: string, event?: any) {
		sel = updateObj(sel, key, event);
		$dashboard = $dashboard;
	}

	onDestroy(() => $record());

	$: entitySelections = sel?.entities ?? [];

	connection.subscribe(async (conn) => {
		if (!conn) return;

		try {
			const [res1, res2]: [any, any] = await Promise.all([
				conn.sendMessagePromise({ type: 'recorder/list_statistic_ids' }),
				conn.sendMessagePromise({ type: 'recorder/validate_statistics' })
			]);

			const list_statistic_ids = Object.values(res1)
				.map((entry: any) =>
					entry?.statistic_id?.startsWith('sensor.') ? entry.statistic_id : null
				)
				.filter(Boolean);

			const validate_statistics_set = new Set(
				Object.values(res2)
					.map((entry: any) => entry[0]?.data?.statistic_id)
					.filter(Boolean)
			);

			options = list_statistic_ids
				.filter((id) => !validate_statistics_set.has(id))
				.map((item) => {
					const entity = $states?.[item];
					const friendly = getName(undefined, entity);
					return {
						id: item,
						label: friendly || item,
						hint: friendly && friendly !== item ? item : undefined
					};
				})
				.sort((a, b) => a.label.localeCompare(b.label));
		} catch (err) {
			console.error(err);
		}
	});

	function ensureEntities() {
		if (entitiesInitialized) return;
		if (!sel?.entities || sel.entities.length === 0) {
			const fallback = demo || options?.[0]?.id;
			if (fallback) {
				set('entities', [{ entity_id: fallback }]);
			} else {
				set('entities', [{}]);
			}
		}
		entitiesInitialized = true;
	}

	$: if (options?.length && !entitiesInitialized) ensureEntities();

	function addEntity() {
		if (!options?.length) return;
		const entries = sel?.entities ? [...sel.entities] : [];
		const fallback = options[0]?.id;
		entries.push({ entity_id: fallback });
		set('entities', entries);
	}

	function updateEntity(index: number, event: CustomEvent<string | undefined>) {
		const entries = sel?.entities ? [...sel.entities] : [];
		entries[index] = { ...(entries[index] || {}), entity_id: event.detail };
		set('entities', entries);
	}

	function removeEntity(index: number) {
		const entries = sel?.entities ? [...sel.entities] : [];
		entries.splice(index, 1);
		if (!entries.length) entries.push({});
		set('entities', entries);
	}

	function handleScaleChange(key: 'scale_min' | 'scale_max', value: string) {
		if (!value) {
			set(key);
			return;
		}

		const numeric = Number(value);
		if (!Number.isNaN(numeric)) {
			set(key, numeric);
		}
	}
</script>

{#if isOpen}
	<Modal>
		<h1 slot="title">{$lang('graph')}</h1>

		<h2>{$lang('preview')}</h2>

		<div class="preview">
			<Graph
				entities={sel?.entities}
				entity_id={sel?.entity_id}
				name={sel?.name}
				period={sel?.period}
				stroke={minMax(stroke)}
				scale_mode={sel?.scale_mode}
				scale_min={sel?.scale_min}
				scale_max={sel?.scale_max}
				variant="preview"
			/>
		</div>

		<h2>{$lang('name')}</h2>

		<InputClear
			condition={name}
			on:clear={() => {
				name = undefined;
				set('name');
			}}
			let:padding
		>
			<input
				name={$lang('name')}
				class="input"
				type="text"
				placeholder={getName(sel, (sel.entity_id && $states[sel.entity_id]) || undefined) ||
					$lang('name')}
				autocomplete="off"
				spellcheck="false"
				bind:value={name}
				on:change={(event) => set('name', event)}
				style:padding
			/>
		</InputClear>

		<h2>{$lang('entities')}</h2>

		{#if options?.length}
			<div class="entities">
				{#each entitySelections as entity, index (index)}
					<div class="entity-row">
						<Select
							computeIcons={true}
							{options}
							placeholder={$lang('sensor')}
							value={entity?.entity_id}
							on:change={(event) => updateEntity(index, event)}
						/>

						<button
							class="icon-button"
							on:click={() => removeEntity(index)}
							disabled={entitySelections.length <= 1}
							use:Ripple={$ripple}
							aria-label={$lang('remove')}
						>
							<Icon icon="mingcute:close-fill" height="18" />
						</button>
					</div>
				{/each}

				<button class="add-entity" on:click={addEntity} use:Ripple={$ripple}>
					<Icon icon="gridicons:add-outline" height="16" />
					{$lang('add_entity')}
				</button>
			</div>
		{:else}
			<p class="hint">{$lang('loading')}</p>
		{/if}

		<h2>{$lang('period')} (data_points)</h2>

		{#if periodOptions}
			<Select
				options={periodOptions}
				placeholder={$lang('period')}
				value={sel?.period}
				on:change={(event) => set('period', event)}
			/>
		{/if}

		<h2>{$lang('scaling')}</h2>

		<Select
			options={scaleModeOptions}
			placeholder={$lang('scaling')}
			value={sel?.scale_mode || 'auto'}
			on:change={(event) => set('scale_mode', event)}
		/>

		{#if (sel?.scale_mode || 'auto') === 'custom'}
			<div class="scale-inputs">
				<label>
					<span>{$lang('scale_min')}</span>
					<input
						class="input"
						type="number"
						step="0.1"
						placeholder="0"
						value={sel?.scale_min ?? ''}
						on:input={(event) =>
							handleScaleChange(
								'scale_min',
								(event.target as HTMLInputElement | null)?.value || ''
							)
						}
					/>
				</label>

				<label>
					<span>{$lang('scale_max')}</span>
					<input
						class="input"
						type="number"
						step="0.1"
						placeholder="100"
						value={sel?.scale_max ?? ''}
						on:input={(event) =>
							handleScaleChange(
								'scale_max',
								(event.target as HTMLInputElement | null)?.value || ''
							)
						}
					/>
				</label>
			</div>
		{/if}

		<h2>{$lang('size')}</h2>

		<input
			class="input"
			type="number"
			placeholder="2"
			on:input={handleNumberRange}
			bind:value={stroke}
			bind:this={numberElement}
			min={range.min}
			max={range.max}
		/>

		<h2>{$lang('mobile')}</h2>

		<div class="button-container">
			<button
				class:selected={sel?.hide_mobile !== true}
				on:click={() => set('hide_mobile')}
				use:Ripple={$ripple}
			>
				{$lang('visible')}
			</button>

			<button
				class:selected={sel?.hide_mobile === true}
				on:click={() => set('hide_mobile', true)}
				use:Ripple={$ripple}
			>
				{$lang('hidden')}
			</button>
		</div>

		<ConfigButtons {sel} />
	</Modal>
{/if}

<style>
	.preview {
		height: 11rem;
	}

	.entities {
		display: flex;
		flex-direction: column;
		gap: 0.75rem;
	}

	.entity-row {
		display: flex;
		gap: 0.5rem;
		align-items: center;
	}

	.icon-button {
		display: flex;
		align-items: center;
		justify-content: center;
		width: 2.75rem;
		height: 3.3rem;
		border-radius: 0.5rem;
		border: 1px solid rgba(255, 255, 255, 0.12);
		background: rgba(255, 255, 255, 0.05);
		color: inherit;
		cursor: pointer;
	}

	.icon-button:disabled {
		opacity: 0.4;
		cursor: not-allowed;
	}

	.add-entity {
		display: inline-flex;
		align-items: center;
		gap: 0.35rem;
		padding: 0.65rem 0.9rem;
		border-radius: 0.5rem;
		border: 1px dashed rgba(255, 255, 255, 0.25);
		background: transparent;
		color: inherit;
		cursor: pointer;
		width: fit-content;
		font-size: 0.9rem;
	}

	.hint {
		opacity: 0.6;
	}

	.scale-inputs {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(8rem, 1fr));
		gap: 0.75rem;
		margin-top: 0.5rem;
	}

	.scale-inputs label {
		display: flex;
		flex-direction: column;
		gap: 0.35rem;
		font-size: 0.85rem;
	}
</style>
