<script lang="ts">
import {
	dashboard,
	states,
	connection,
	lang,
	history,
	historyIndex,
	record,
	ripple,
	entityList
} from '$lib/Stores';
	import { onDestroy } from 'svelte';
	import Graph from '$lib/Sidebar/Graph.svelte';
	import Select from '$lib/Components/Select.svelte';
	import ConfigButtons from '$lib/Modal/ConfigButtons.svelte';
	import InputClear from '$lib/Components/InputClear.svelte';
	import Modal from '$lib/Modal/Index.svelte';
import { updateObj, getName } from '$lib/Utils';
import type { GraphEntityConfig, GraphItem } from '$lib/Types';
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
type Option = { id: string; label: string; hint?: string };

let statsOptions: Option[] = [];
let entityOptions: Option[] = [];
let stroke = sel?.stroke;
let numberElement: HTMLInputElement;
let entitiesInitialized = false;
let statsLoaded = false;

$: entityOptions = statsOptions.length
	? statsOptions
	: $entityList
		? $entityList('sensor')
		: [];

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

	const scaleTypeOptions = [
		{ id: 'linear', label: $lang('scale_type_linear') },
		{ id: 'log', label: $lang('scale_type_log') }
	];
	const decimalRange = { min: 0, max: 6 };

	const columnSpanOptions = [
		{ id: '1', label: '1 (compact)' },
		{ id: '2', label: '2 (default)' },
		{ id: '3', label: '3 (wide)' }
	];

	const rowSpanOptions = [
		{ id: '1', label: '1 (short)' },
		{ id: '2', label: '2' },
		{ id: '3', label: '3' },
		{ id: '4', label: '4 (tall)' }
	];

	function handleDecimalsInput(event: Event) {
		const input = event.target as HTMLInputElement | null;
		const raw = input?.value ?? '';
		if (raw === '') {
			set('decimals');
			return;
		}

		const value = Number(raw);
		if (Number.isNaN(value)) {
			return;
		}

		const clamped = Math.max(decimalRange.min, Math.min(decimalRange.max, value));
		set('decimals', clamped);
	}

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
		if (!conn) {
			statsOptions = [];
			return;
		}

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

			const mapped = list_statistic_ids
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

			statsOptions = mapped.length ? mapped : [];
		} catch (err) {
			console.error(err);
			statsOptions = [];
		}
	});

function ensureEntities() {
	if (entitiesInitialized) return;
	if (!sel?.entities || sel.entities.length === 0) {
		const fallback = demo || entityOptions?.[0]?.id || sel?.entity_id;
		if (fallback) {
			set('entities', [{ entity_id: fallback }]);
		} else {
			set('entities', [{}]);
		}
	}
	entitiesInitialized = true;
}

$: if (!entitiesInitialized) ensureEntities();

	function addEntity() {
		if (!entityOptions?.length) return;
		const entries = sel?.entities ? [...sel.entities] : [];
		const fallback = entityOptions[0]?.id;
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

function getInputValue(event: Event) {
	const target = event.target as HTMLInputElement | null;
	return target?.value ?? '';
}

function updateEntityField<K extends keyof GraphEntityConfig>(
	index: number,
	key: K,
	value?: GraphEntityConfig[K]
) {
	const entries = sel?.entities ? [...sel.entities] : [];
	const current = { ...(entries[index] || {}) };

	if (value === undefined || value === '') {
		delete current[key];
	} else {
		current[key] = value;
	}

	entries[index] = current;
	set('entities', entries);
}

function updateEntityNumberField<K extends 'scale_min' | 'scale_max' | 'decimals'>(
	index: number,
	key: K,
	value: string
) {
	if (!value) {
		updateEntityField(index, key);
		return;
	}

	const numeric = Number(value);
	if (Number.isNaN(numeric)) return;

	if (key === 'decimals') {
		const clamped = Math.max(decimalRange.min, Math.min(decimalRange.max, Math.round(numeric)));
		updateEntityField(index, key, clamped as GraphEntityConfig[K]);
	} else {
		updateEntityField(index, key, numeric as GraphEntityConfig[K]);
	}
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
				scale_type={sel?.scale_type}
				decimals={sel?.decimals}
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

		<div class="entities">
			{#each entitySelections as entity, index (index)}
				<div class="entity-group">
					<div class="entity-row">
						<div class="full-width">
							<Select
								computeIcons={true}
								options={entityOptions}
								placeholder={$lang('sensor')}
								value={entity?.entity_id}
								on:change={(event) => updateEntity(index, event)}
							/>
						</div>

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

					{#if !entityOptions?.length}
						<p class="hint small">{$lang('loading')}</p>
					{/if}

					<label class="entity-alias">
						<span>{$lang('alias')}</span>
						<InputClear
							condition={entity?.alias}
							on:clear={() => updateEntityField(index, 'alias')}
							let:padding
						>
							<input
								class="input"
								type="text"
								placeholder={entity?.entity_id}
								value={entity?.alias ?? ''}
								on:input={(event) =>
									updateEntityField(index, 'alias', getInputValue(event) || undefined)
								}
								style:padding
							/>
						</InputClear>
					</label>

					<div class="entity-scaling">
						<Select
							options={scaleModeOptions}
							placeholder={$lang('scaling')}
							value={entity?.scale_mode || sel?.scale_mode || 'auto'}
							on:change={(event) => updateEntityField(index, 'scale_mode', event.detail)}
						/>

						<Select
							options={scaleTypeOptions}
							placeholder={$lang('scale_type')}
							value={entity?.scale_type || sel?.scale_type || 'linear'}
							on:change={(event) => updateEntityField(index, 'scale_type', event.detail)}
						/>

						<div class="entity-scale-inputs">
							<label>
								<span>{$lang('scale_min')}</span>
								<input
									class="input"
									type="number"
									step="0.1"
									placeholder="0"
									value={entity?.scale_min ?? ''}
									on:input={(event) =>
										updateEntityNumberField(index, 'scale_min', getInputValue(event))
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
									value={entity?.scale_max ?? ''}
									on:input={(event) =>
										updateEntityNumberField(index, 'scale_max', getInputValue(event))
									}
								/>
							</label>
						</div>

						<div class="entity-scale-inputs">
							<label>
								<span>{$lang('decimals')}</span>
								<input
									class="input"
									type="number"
									min={decimalRange.min}
									max={decimalRange.max}
									placeholder="1"
									value={entity?.decimals ?? ''}
									on:input={(event) =>
										updateEntityNumberField(index, 'decimals', getInputValue(event))
									}
								/>
							</label>
						</div>
					</div>
				</div>
			{/each}

			<button class="add-entity" on:click={addEntity} use:Ripple={$ripple}>
				<Icon icon="gridicons:add-outline" height="16" />
				{$lang('add_entity')}
			</button>
		</div>

		<h2>Layout (main view)</h2>

		<div class="entity-scale-inputs">
			<div class="layout-control">
				<span>Width (columns)</span>
				<Select
					options={columnSpanOptions}
					placeholder="Columns"
					value={(sel?.width_span ?? 2).toString()}
					on:change={(event) => set('width_span', Number(event.detail))}
				/>
			</div>

			<div class="layout-control">
				<span>Height (rows)</span>
				<Select
					options={rowSpanOptions}
					placeholder="Rows"
					value={(sel?.height_span ?? 4).toString()}
					on:change={(event) => set('height_span', Number(event.detail))}
				/>
			</div>
		</div>

		<p class="hint small">Applies to graph cards placed in the main dashboard grid (not the sidebar).</p>

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

		<h2>{$lang('scale_type')}</h2>

		<Select
			options={scaleTypeOptions}
			placeholder={$lang('scale_type')}
			value={sel?.scale_type || 'linear'}
			on:change={(event) => set('scale_type', event)}
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
							handleScaleChange('scale_min', getInputValue(event))
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
							handleScaleChange('scale_max', getInputValue(event))
						}
					/>
				</label>
			</div>
		{/if}

		<h2>{$lang('decimals')}</h2>

		<input
			class="input"
			type="number"
			min={decimalRange.min}
			max={decimalRange.max}
			placeholder="1"
			value={sel?.decimals ?? ''}
			on:input={handleDecimalsInput}
		/>

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
		margin-bottom: 1.25rem;
		padding: 0 0.5rem;
	}

	.preview :global(.graph-container) {
		min-height: 11rem;
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

	.full-width {
		flex: 1;
		min-width: 0;
	}

	.entity-group {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		padding: 0.85rem 1rem;
		border-radius: 0.6rem;
		border: 1px solid rgba(255, 255, 255, 0.12);
		background: rgba(255, 255, 255, 0.03);
	}

	.entity-scaling {
		display: grid;
		gap: 0.5rem;
		grid-template-columns: repeat(auto-fit, minmax(10rem, 1fr));
	}

	.entity-scale-inputs {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(8rem, 1fr));
		gap: 0.5rem;
	}

	.entity-scale-inputs label {
		display: flex;
		flex-direction: column;
		gap: 0.25rem;
		font-size: 0.8rem;
	}

	.entity-alias {
		display: flex;
		flex-direction: column;
		gap: 0.35rem;
		font-size: 0.85rem;
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

	.hint.small {
		margin: 0.2rem 0 0;
		font-size: 0.8rem;
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
