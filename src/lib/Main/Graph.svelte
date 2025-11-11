<script lang="ts">
	import Graph from '$lib/Sidebar/Graph.svelte';
	import { editMode, ripple } from '$lib/Stores';
	import { openModal } from 'svelte-modals';
	import Ripple from 'svelte-ripple';

	export let sel: any;

	function handleClick(event: MouseEvent) {
		if (!$editMode) return;
		event.stopPropagation();
		openModal(() => import('$lib/Modal/GraphConfig.svelte'), { sel });
	}

	function handleKeydown(event: KeyboardEvent) {
		if (!$editMode) return;
		if (event.key === 'Enter' || event.key === ' ') {
			event.preventDefault();
			handleClick(event as unknown as MouseEvent);
		}
	}
</script>

<div
	role="button"
	class="graph-card"
		class:interactive={$editMode}
	tabindex={$editMode ? 0 : -1}
	on:click={handleClick}
	on:keydown={handleKeydown}
	use:Ripple={$editMode ? $ripple : undefined}
	aria-disabled={!$editMode}
	style="min-height: 100%; width: 100%;"
>
	<Graph
		entities={sel?.entities}
		entity_id={sel?.entity_id}
		name={sel?.name}
		period={sel?.period}
		stroke={sel?.stroke}
		scale_mode={sel?.scale_mode}
		scale_min={sel?.scale_min}
		scale_max={sel?.scale_max}
		scale_type={sel?.scale_type}
		decimals={sel?.decimals}
		editable={$editMode}
		variant="card"
	/>
</div>

<style>
	.graph-card {
		all: unset;
		display: block;
		height: 100%;
		width: 100%;
		border-radius: 0.65rem;
		overflow: hidden;
	}

	.interactive {
		cursor: pointer;
	}
</style>
