<script lang="ts">
	import { states, connection, selectedLanguage, lang } from '$lib/Stores';
import { scaleTime, scaleLinear, scaleLog } from 'd3-scale';
import type { ScaleLinear, ScaleLogarithmic } from 'd3-scale';
	import { line, curveMonotoneX } from 'd3-shape';
	import { extent, bisector } from 'd3-array';
import { getName } from '$lib/Utils';
import { onDestroy } from 'svelte';

	type ScaleMode = 'auto' | 'zero' | 'custom';
	type ScaleType = 'linear' | 'log';

type GraphEntity = {
	entity_id?: string;
	color?: string;
	alias?: string;
	scale_mode?: ScaleMode;
	scale_min?: number;
	scale_max?: number;
	scale_type?: ScaleType;
	decimals?: number;
};

	type ChartPoint = {
		x: Date;
		y: number;
	};

type ChartSeries = {
	entity_id: string;
	color: string;
	unit: string;
	data: ChartPoint[];
};

type HoverPoint = {
	entity_id: string;
	color: string;
	label: string;
	unit: string;
	value: number | undefined;
	scaleLabel?: string;
	decimals?: number;
};

	export let entity_id: string | undefined;
	export let entities: GraphEntity[] | undefined;
	export let name: string | undefined = undefined;
	export let period = 'day';
	export let stroke = 2;
	export let scale_mode: ScaleMode = 'auto';
	export let scale_min: number | undefined;
	export let scale_max: number | undefined;
	export let variant: 'sidebar' | 'card' | 'preview' = 'sidebar';
	export let editable = false;
export let scale_type: ScaleType = 'linear';
export let decimals: number | undefined;

	const palette = ['#7dd3fc', '#f472b6', '#c084fc', '#facc15', '#fb923c', '#34d399', '#f87171'];

	const padding = 8;
	const defaultWindow = 2629800 * 1000;

	let start_time = new Date(Date.now() - defaultWindow).toISOString();
	let end_time = new Date().toISOString();

	let width = 0;
	let height = 0;
	let chartSeries: ChartSeries[] = [];
let hoverPoints: HoverPoint[] = [];
let hoverTimestamp: Date | undefined;
let hovering = false;
let resizeTimeout: ReturnType<typeof setTimeout> | undefined;
let isResizing = false;
let seriesScales: Map<
	string,
	{
		scale: ScaleLinear<number, number> | ScaleLogarithmic<number, number>;
		settings: ScaleSettings;
	}
> = new Map();
let hoverCoords: { x: number; y: number } | undefined;
let tooltipPosition: { left: number; top: number } | undefined;
let chartElement: HTMLDivElement | undefined;

$: tooltipPosition =
	hoverCoords && width
		? {
				left: Math.min(Math.max(hoverCoords.x + 12, 8), width - 8),
				top: Math.max(hoverCoords.y - 56, 8)
		  }
		: undefined;

	$: resolvedEntities = (entities && entities.length ? entities : entity_id ? [{ entity_id }] : [])
		.filter((entry): entry is GraphEntity & { entity_id: string } => Boolean(entry?.entity_id));

	function getResolvedEntry(entity_id: string) {
		return resolvedEntities.find((entry) => entry.entity_id === entity_id);
	}

	function getEntityLabel(entity_id: string) {
		const resolved = getResolvedEntry(entity_id);
		if (resolved?.alias) return resolved.alias;
		return getName(undefined, $states?.[entity_id]) || entity_id;
	}

	$: title =
		name ?? (resolvedEntities.length === 1 ? getEntityLabel(resolvedEntities[0].entity_id) : $lang('graph'));

	$: fetchKey = `${period}-${resolvedEntities.map((entry) => entry.entity_id).join(',')}`;

	$: if (fetchKey) fetchData();

	$: xDomain =
		chartSeries.length && chartSeries.some((series) => series.data.length)
			? (extent(
					chartSeries.flatMap((series) => series.data),
					(point) => point.x
				) as [Date, Date])
			: [new Date(Date.now() - defaultWindow), new Date()];

	function refreshWindow() {
		end_time = new Date().toISOString();
		start_time = new Date(Date.now() - defaultWindow).toISOString();
	}

type ScaleSettings = {
	mode: ScaleMode;
	min?: number;
	max?: number;
	type: ScaleType;
	decimals?: number;
};

function getSeriesSettings(entity_id: string): ScaleSettings {
	const entry = resolvedEntities.find((item) => item.entity_id === entity_id);
	return {
		mode: entry?.scale_mode || scale_mode,
		min: entry?.scale_min ?? scale_min,
		max: entry?.scale_max ?? scale_max,
		type: entry?.scale_type || scale_type,
		decimals: entry?.decimals ?? decimals
	};
}

	function getSeriesExtent(series: ChartSeries): [number, number] {
		if (!series.data.length) {
			const fallback = Number($states?.[series.entity_id]?.state);
			const value = Number.isFinite(fallback) ? Number(fallback) : 0;
			return [value, value + 1];
		}
		const domain = extent(series.data, (point) => point.y) as [number, number];
		if (domain[0] === domain[1]) {
			return [domain[0], domain[0] + 1];
		}
		return domain;
	}

	function resolveLinearDomain(series: ChartSeries, settings: ScaleSettings): [number, number] {
		const [dataMin, dataMax] = getSeriesExtent(series);

		if (settings.mode === 'custom') {
			const minValue = settings.min ?? dataMin;
			let maxValue = settings.max ?? dataMax;
			if (maxValue === minValue) {
				maxValue = minValue + 1;
			}
			return [Math.min(minValue, maxValue), Math.max(minValue, maxValue)];
		}

		if (settings.mode === 'zero') {
			const upper = Number.isFinite(dataMax) ? dataMax : 1;
			return [0, upper === 0 ? 1 : upper];
		}

		return [dataMin, dataMax];
	}

	function resolveLogDomain(
		series: ChartSeries,
		settings: ScaleSettings,
		domain: [number, number]
	): [number, number] {
		const MIN_VALUE = 0.0001;
		let [min, max] = domain;

		const positiveValues = series.data.map((point) => point.y).filter((value) => value > 0);

		if (settings.mode !== 'custom') {
			if (positiveValues.length) {
				min = Math.min(...positiveValues);
				max = Math.max(...positiveValues);
			} else {
				min = MIN_VALUE;
				max = MIN_VALUE * 10;
			}
		} else {
			if (min <= 0) min = MIN_VALUE;
			if (max <= min) max = min * 10;
		}

		min = Math.max(min, MIN_VALUE);
		max = Math.max(max, min * 1.01);

		return [min, max];
	}

	$: xScale =
		width && typeof window !== 'undefined'
			? scaleTime()
					.domain(xDomain)
					.range([padding, Math.max(width - padding, padding + 1)])
			: undefined;

	function getFractionDigits(settings?: ScaleSettings) {
		const value = settings?.decimals ?? decimals;
		return typeof value === 'number' && Number.isFinite(value) ? Math.max(0, value) : 1;
	}

	function getScaleLabel(settings?: ScaleSettings) {
		if (!settings) return undefined;
		if (settings.type === 'log') return $lang('scale_type_log_short');
		if (settings.mode === 'zero') return $lang('scale_mode_zero_short');
		if (settings.mode === 'custom') return $lang('scale_mode_custom_short');
		return undefined;
	}

	$: scaleDependencyKey = JSON.stringify({
		global: {
			scale_mode,
			scale_min,
			scale_max,
			scale_type,
			decimals
		},
		entities: resolvedEntities.map(
			({
				entity_id,
				scale_mode: mode,
				scale_min: min,
				scale_max: max,
				scale_type: type,
				decimals: entityDecimals
			}) => ({
				entity_id,
				mode,
				min,
				max,
				type,
				decimals: entityDecimals
			})
		)
	});

	$: seriesScales =
		height && typeof window !== 'undefined'
			? new Map(
					chartSeries.map((series) => {
						scaleDependencyKey;
						const settings = getSeriesSettings(series.entity_id);
						const baseDomain = resolveLinearDomain(series, settings);
						const domain =
							settings.type === 'log'
								? resolveLogDomain(series, settings, baseDomain)
								: baseDomain;
						const scale =
							settings.type === 'log' ? scaleLog().domain(domain) : scaleLinear().domain(domain);

						scale.range([Math.max(height - padding, padding + 1), padding]).nice();

						return [series.entity_id, { scale, settings }];
					})
				)
			: new Map();

	$: seriesPaths =
		xScale && seriesScales.size
			? chartSeries.map((series) => {
					const scaleEntry = seriesScales.get(series.entity_id);
					if (!scaleEntry) {
						return { entity_id: series.entity_id, color: series.color, path: null };
					}
					const lineGenerator = line<ChartPoint>()
						.x((d) => xScale(d.x))
						.y((d) => scaleEntry.scale(d.y))
						.curve(curveMonotoneX);
					return {
						entity_id: series.entity_id,
						color: series.color,
						path: lineGenerator(series.data)
					};
				})
			: [];

	$: fallbackLegend = chartSeries.map((series) => {
		const last = series.data[series.data.length - 1];
		const fallbackValue =
			last?.y ?? Number.parseFloat(String($states?.[series.entity_id]?.state));
		const scaleSettings = seriesScales.get(series.entity_id)?.settings || getSeriesSettings(series.entity_id);
		const fractionDigits = getFractionDigits(scaleSettings);

		return {
			entity_id: series.entity_id,
			color: series.color,
			label: getEntityLabel(series.entity_id),
			unit: series.unit,
			value: Number.isFinite(fallbackValue) ? fallbackValue : undefined,
			decimals: fractionDigits,
			scaleLabel: getScaleLabel(scaleSettings)
		};
	});

	$: legendEntries =
		hovering && hoverPoints.length
			? hoverPoints
			: fallbackLegend;

	$: if (editable && hovering) hovering = false;

	function formatHoverLabel(date: Date | undefined) {
		if (!date) return undefined;
		const locale = $selectedLanguage || undefined;

		const tryFormat = (options: Intl.DateTimeFormatOptions) => {
			try {
				return new Intl.DateTimeFormat(locale, options).format(date);
			} catch {
				return undefined;
			}
		};

		return (
			tryFormat({ dateStyle: 'short', timeStyle: 'short' }) ||
			tryFormat({ weekday: 'short', hour: '2-digit', minute: '2-digit' }) ||
			date.toLocaleString()
		);
	}

	$: hoverLabel = hovering ? formatHoverLabel(hoverTimestamp) : undefined;

	function fetchData() {
		if (typeof window === 'undefined') return;
		if (!resolvedEntities.length) {
			chartSeries = [];
			return;
		}

		refreshWindow();

		connection.subscribe((conn) =>
			conn
				?.sendMessagePromise<
					Record<
						string,
						{
							start: string;
							mean?: number;
							state?: number;
							unit_of_measurement?: string;
						}[]
					>
				>({
					type: 'recorder/statistics_during_period',
					start_time,
					end_time,
					statistic_ids: resolvedEntities.map((entry) => entry.entity_id),
					period: period || 'day'
				})
				.then((res) => {
					chartSeries = resolvedEntities.map((entry, index) => {
						const entityId = entry.entity_id;
						const entity = $states?.[entityId];
						const unit =
							entity?.attributes?.unit_of_measurement ||
							res?.[entityId]?.[0]?.unit_of_measurement ||
							'';
						const color = entry.color || palette[index % palette.length];

						const data = Array.isArray(res?.[entityId])
							? res[entityId]
									.map((item) => {
										const numeric = item.mean ?? item.state ?? Number.NaN;
										return {
											x: new Date(item.start),
											y: Number(numeric)
										};
									})
									.filter((point) => Number.isFinite(point.y))
							: [];

						return {
							entity_id: entityId,
							color,
							unit,
							data
						};
					});
				})
		);
	}

function handlePointerMove(event: PointerEvent | MouseEvent) {
	if (editable) return;
	if (!xScale || !chartSeries.length) return;
	hovering = true;
	const bounds = (chartElement || (event.currentTarget as HTMLElement)).getBoundingClientRect();
	const x = event.clientX - bounds.left;
		const y = event.clientY - bounds.top;
		hoverCoords = { x, y };

		const xValue = xScale.invert(x);
		hoverTimestamp = xValue;

		const bisectDate = bisector((d: ChartPoint) => d.x).left;

		hoverPoints = chartSeries.map((series) => {
			const index = bisectDate(series.data, xValue, 1);
			const previous = series.data[index - 1];
			const next = series.data[index];

			let point = previous;

			if (previous && next) {
				point =
					xValue.getTime() - previous.x.getTime() >
					next.x.getTime() - xValue.getTime()
						? next
						: previous;
			} else if (!previous) {
				point = next;
			}

			const scaleSettings = seriesScales.get(series.entity_id)?.settings;
		const fractionDigits = getFractionDigits(scaleSettings);

			return {
				entity_id: series.entity_id,
				color: series.color,
				label: getEntityLabel(series.entity_id),
				unit: series.unit,
				value: point?.y,
				decimals: fractionDigits,
				scaleLabel: getScaleLabel(scaleSettings)
			};
		});
	}

function handlePointerLeave() {
	if (editable) return;
	hovering = false;
	hoverCoords = undefined;
}

function formatValue(value: number | undefined, fractionDigits: number) {
	if (value === undefined || Number.isNaN(value)) return '—';
	return Intl.NumberFormat($selectedLanguage, {
		maximumFractionDigits: fractionDigits,
		minimumFractionDigits: fractionDigits
	}).format(value);
}

	$: if (width || height) {
		isResizing = true;
		clearTimeout(resizeTimeout);
		resizeTimeout = setTimeout(() => {
			isResizing = false;
		}, 200);
	}

	onDestroy(() => {
		if (resizeTimeout) clearTimeout(resizeTimeout);
	});
</script>

<div
	class="graph-container"
	class:sidebar={variant === 'sidebar'}
	class:card={variant === 'card'}
	class:preview={variant === 'preview'}
>
	<div class="graph-header">
		<div>
			<p class="title">{title}</p>
			{#if hoverLabel}
				<p class="subtitle">{hoverLabel}</p>
			{/if}
		</div>
	</div>

	{#if !resolvedEntities.length}
		<div class="empty-state">
			{$lang('no_entities')}
		</div>
	{:else}
		<div
			class="chart"
			class:editing={editable}
			class:loading={!chartSeries.length}
			role="presentation"
			bind:this={chartElement}
			bind:clientWidth={width}
			bind:clientHeight={height}
			on:pointermove={handlePointerMove}
			on:pointerleave={handlePointerLeave}
			on:mousemove={handlePointerMove}
			on:mouseleave={handlePointerLeave}
			style:transition={`opacity ${isResizing ? 0 : 150}ms ease`}
		>
			{#if chartSeries.length}
				<svg
					width="100%"
					height="100%"
					role="img"
					aria-label={title}
					on:pointermove={handlePointerMove}
					on:pointerleave={handlePointerLeave}
					on:mousemove={handlePointerMove}
					on:mouseleave={handlePointerLeave}
				>
					{#each seriesPaths as series (series.entity_id)}
						{#if series.path}
							<path
								d={series.path}
								class="line"
								style={`stroke: ${series.color}; stroke-width: ${stroke}`}
							/>
						{/if}
					{/each}
				</svg>
			{:else}
				<div class="loading-label">{$lang('loading')}</div>
			{/if}

			{#if hovering && hoverCoords}
				<div class="hover-line" style={`left:${hoverCoords.x}px`}></div>
			{/if}

			{#if hovering && hoverCoords && tooltipPosition && legendEntries.length}
				<div
					class="hover-tooltip"
					style={`left:${tooltipPosition.left}px; top:${tooltipPosition.top}px`}
				>
					{#if hoverLabel}
						<p class="tooltip-time">{hoverLabel}</p>
					{/if}
					{#each legendEntries as entry (entry.entity_id)}
						<div class="tooltip-row">
							<span class="dot dot-small" style={`background:${entry.color}`} />
							<span class="tooltip-label">{entry.label}</span>
							<span class="tooltip-value">
								{formatValue(entry.value, entry.decimals ?? 1)}
								{entry.unit}
							</span>
						</div>
					{/each}
				</div>
			{/if}
		</div>

		<div class="legend">
			{#if legendEntries.length}
				{#each legendEntries as entry (entry.entity_id)}
					<div class="legend-item">
						<span class="dot" style={`background:${entry.color}`} />
						<div class="legend-text">
							<span class="legend-name">{entry.label}</span>
							<span class="legend-value">
								{formatValue(entry.value, entry.decimals ?? 1)}
								{entry.unit}
							</span>
							{#if entry.scaleLabel}
								<span class="legend-scale">{entry.scaleLabel}</span>
							{/if}
						</div>
					</div>
				{/each}
			{:else}
				<div class="loading-label small">{$lang('loading')}</div>
			{/if}
		</div>
	{/if}
</div>

<style>
	.graph-container {
		display: flex;
		flex-direction: column;
		gap: 0.5rem;
		width: 100%;
	}

	.sidebar {
		padding: var(--theme-sidebar-item-padding);
	}

	.card,
	.preview {
		background: rgba(255, 255, 255, 0.08);
		border-radius: 0.65rem;
		padding: 1rem 1.1rem 1.1rem;
		height: 100%;
		box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.02);
	}

	.graph-header {
		display: flex;
		align-items: baseline;
		justify-content: space-between;
		gap: 0.75rem;
	}

	.title {
		margin: 0;
		font-weight: 600;
		font-size: 0.95rem;
	}

	.subtitle {
		margin: 0.2rem 0 0;
		font-size: 0.8rem;
		opacity: 0.7;
	}


	.empty-state {
		font-size: 0.9rem;
		opacity: 0.7;
		padding: 0.5rem 0;
	}

	.chart {
		position: relative;
		width: 100%;
		height: 5.5rem;
	}

	.card .chart,
	.preview .chart {
		height: 10rem;
	}

	.chart.editing {
		pointer-events: none;
	}

	.hover-line {
		position: absolute;
		top: 0;
		bottom: 0;
		width: 1px;
		background: rgba(255, 255, 255, 0.35);
		pointer-events: none;
	}

	.hover-tooltip {
		position: absolute;
		min-width: 9rem;
		max-width: 12rem;
		background: rgba(0, 0, 0, 0.75);
		backdrop-filter: blur(6px);
		border-radius: 0.5rem;
		padding: 0.45rem 0.55rem;
		font-size: 0.78rem;
		pointer-events: none;
		transform: translate(-50%, -100%);
		display: grid;
		gap: 0.25rem;
	}

	.tooltip-time {
		margin: 0;
		font-weight: 600;
		font-size: 0.75rem;
		opacity: 0.8;
	}

	.tooltip-row {
		display: grid;
		grid-template-columns: auto 1fr auto;
		gap: 0.35rem;
		align-items: center;
	}

	.tooltip-label {
		opacity: 0.9;
	}

	.tooltip-value {
		font-variant-numeric: tabular-nums;
		font-weight: 600;
	}

	svg {
		width: 100%;
		height: 100%;
	}

	.line {
		fill: none;
		stroke-linecap: round;
		stroke-linejoin: round;
	}

	.legend {
		display: grid;
		gap: 0.25rem;
		font-size: 0.82rem;
	}

	.legend-item {
		display: flex;
		align-items: center;
		gap: 0.45rem;
	}

	.dot {
		width: 0.65rem;
		height: 0.65rem;
		border-radius: 999px;
	}

	.dot-small {
		width: 0.45rem;
		height: 0.45rem;
	}

	.legend-text {
		display: grid;
		grid-template-columns: 1fr auto;
		width: 100%;
		gap: 0.2rem;
	}

	.legend-name {
		opacity: 0.85;
	}

	.legend-value {
		font-variant-numeric: tabular-nums;
		font-weight: 600;
	 }

	.legend-scale {
		grid-column: 2;
		font-size: 0.7rem;
		text-transform: uppercase;
		opacity: 0.6;
	}

	.loading-label {
		position: absolute;
		inset: 0;
		display: grid;
		place-items: center;
		opacity: 0.6;
		font-size: 0.85rem;
	}

	.loading-label.small {
		position: static;
	}
</style>
