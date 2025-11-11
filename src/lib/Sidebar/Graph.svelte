<script lang="ts">
	import { states, connection, selectedLanguage, lang } from '$lib/Stores';
	import { scaleTime, scaleLinear } from 'd3-scale';
	import { line, curveMonotoneX } from 'd3-shape';
	import { extent, bisector } from 'd3-array';
	import { getName } from '$lib/Utils';
	import { onDestroy } from 'svelte';

	type GraphEntity = {
		entity_id?: string;
		color?: string;
	};

	type ChartPoint = {
		x: Date;
		y: number;
	};

	type ChartSeries = {
		entity_id: string;
		color: string;
		label: string;
		unit: string;
		data: ChartPoint[];
	};

	type HoverPoint = {
		entity_id: string;
		color: string;
		label: string;
		unit: string;
		value: number | undefined;
	};

	export let entity_id: string | undefined;
	export let entities: GraphEntity[] | undefined;
	export let name: string | undefined = undefined;
	export let period = 'day';
	export let stroke = 2;
	export let scale_mode: 'auto' | 'zero' | 'custom' = 'auto';
	export let scale_min: number | undefined;
	export let scale_max: number | undefined;
	export let variant: 'sidebar' | 'card' | 'preview' = 'sidebar';

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

	$: resolvedEntities = (entities && entities.length ? entities : entity_id ? [{ entity_id }] : [])
		.filter((entry): entry is { entity_id: string; color?: string } => Boolean(entry?.entity_id));

	$: title =
		name ??
		(resolvedEntities.length === 1
			? getName(undefined, $states?.[resolvedEntities[0].entity_id]) ||
				resolvedEntities[0].entity_id
			: $lang('graph'));

	$: fetchKey = `${period}-${resolvedEntities.map((entry) => entry.entity_id).join(',')}`;

	$: if (fetchKey) fetchData();

	$: chartPoints = chartSeries.flatMap((series) => series.data);

	$: xDomain = chartPoints.length
		? (extent(chartPoints, (point) => point.x) as [Date, Date])
		: [new Date(Date.now() - defaultWindow), new Date()];

	$: computedYExtent =
		chartPoints.length
			? (extent(chartPoints, (point) => point.y) as [number, number])
			: [0, 1];

	function refreshWindow() {
		end_time = new Date().toISOString();
		start_time = new Date(Date.now() - defaultWindow).toISOString();
	}

	function resolveYDomain(): [number, number] {
		const fallbackMin = computedYExtent?.[0] ?? 0;
		const fallbackMax = computedYExtent?.[1] ?? (fallbackMin || 1) + 1;

		if (scale_mode === 'custom') {
			const minValue = scale_min ?? fallbackMin;
			const maxValue = scale_max ?? fallbackMax;
			const min = Number.isFinite(minValue) ? minValue : fallbackMin;
			let max = Number.isFinite(maxValue) ? maxValue : fallbackMax;

			if (max === min) {
				max = min + 1;
			}

			return [Math.min(min, max), Math.max(min, max)];
		}

		if (scale_mode === 'zero') {
			const upper = Number.isFinite(fallbackMax) ? fallbackMax : 1;
			return [0, upper === 0 ? 1 : upper];
		}

		if (computedYExtent?.[0] === computedYExtent?.[1]) {
			const min = computedYExtent?.[0] ?? 0;
			return [min, min + 1];
		}

		return [fallbackMin, fallbackMax];
	}

	$: xScale =
		width && typeof window !== 'undefined'
			? scaleTime()
					.domain(xDomain)
					.range([padding, Math.max(width - padding, padding + 1)])
			: undefined;

	$: yScale =
		height && typeof window !== 'undefined'
			? scaleLinear()
					.domain(resolveYDomain())
					.range([Math.max(height - padding, padding + 1), padding])
					.nice()
			: undefined;

	$: lineGenerator =
		xScale && yScale
			? line<ChartPoint>()
					.x((d) => xScale(d.x))
					.y((d) => yScale(d.y))
					.curve(curveMonotoneX)
			: undefined;

	$: seriesPaths =
		lineGenerator
			? chartSeries.map((series) => ({
					entity_id: series.entity_id,
					color: series.color,
					path: lineGenerator(series.data)
				}))
			: [];

	$: fallbackLegend = chartSeries.map((series) => {
		const last = series.data[series.data.length - 1];
		const fallbackValue =
			last?.y ?? Number.parseFloat(String($states?.[series.entity_id]?.state));
		return {
			entity_id: series.entity_id,
			color: series.color,
			label: series.label,
			unit: series.unit,
			value: Number.isFinite(fallbackValue) ? fallbackValue : undefined
		};
	});

	$: legendEntries =
		hovering && hoverPoints.length
			? hoverPoints
			: fallbackLegend;

	$: hoverLabel =
		hovering && hoverTimestamp
			? new Intl.DateTimeFormat($selectedLanguage, {
					weekday: 'short',
					timeStyle: 'short'
				}).format(hoverTimestamp)
			: undefined;

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
						const label = getName(undefined, entity) || entityId;
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
							label,
							unit,
							data
						};
					});
				})
		);
	}

	function handlePointerMove(event: PointerEvent) {
		if (!xScale || !chartSeries.length) return;
		hovering = true;
		const bounds = (event.currentTarget as HTMLElement).getBoundingClientRect();
		const x = event.clientX - bounds.left;
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

			return {
				entity_id: series.entity_id,
				color: series.color,
				label: series.label,
				unit: series.unit,
				value: point?.y
			};
		});
	}

	function handlePointerLeave() {
		hovering = false;
	}

	function formatValue(value: number | undefined) {
		if (value === undefined || Number.isNaN(value)) return '—';
		return Intl.NumberFormat($selectedLanguage, {
			maximumFractionDigits: 1
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

		{#if scale_mode === 'zero'}
			<span class="badge">{$lang('scale_mode_zero')}</span>
		{:else if scale_mode === 'custom'}
			<span class="badge">{$lang('scale_mode_custom')}</span>
		{/if}
	</div>

	{#if !resolvedEntities.length}
		<div class="empty-state">
			{$lang('no_entities')}
		</div>
	{:else}
		<div
			class="chart"
			class:loading={!chartSeries.length}
			bind:clientWidth={width}
			bind:clientHeight={height}
			on:pointermove={handlePointerMove}
			on:pointerleave={handlePointerLeave}
			style:transition={`opacity ${isResizing ? 0 : 150}ms ease`}
		>
			{#if chartSeries.length}
				<svg width="100%" height="100%" role="img" aria-label={title}>
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
		</div>

		<div class="legend">
			{#if legendEntries.length}
				{#each legendEntries as entry (entry.entity_id)}
					<div class="legend-item">
						<span class="dot" style={`background:${entry.color}`} />
						<div class="legend-text">
							<span class="legend-name">{entry.label}</span>
							<span class="legend-value">
								{formatValue(entry.value)}
								{entry.unit}
							</span>
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

	.badge {
		font-size: 0.7rem;
		text-transform: uppercase;
		letter-spacing: 0.05em;
		background: rgba(255, 255, 255, 0.08);
		padding: 0.15rem 0.5rem;
		border-radius: 999px;
		align-self: flex-start;
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

	.legend-text {
		display: flex;
		justify-content: space-between;
		width: 100%;
		gap: 0.4rem;
	}

	.legend-name {
		opacity: 0.85;
	}

	.legend-value {
		font-variant-numeric: tabular-nums;
		font-weight: 600;
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
