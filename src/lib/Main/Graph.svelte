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
	class="graph-card"
	class:interactive={$editMode}
	role={$editMode ? 'button' : undefined}
	tabindex={$editMode ? 0 : -1}
	on:click={handleClick}
	on:keydown={handleKeydown}
	use:Ripple={$editMode ? $ripple : undefined}
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
		variant="card"
	/>
</div>

<style>
	.graph-card {
		height: 100%;
	}

	.interactive {
		cursor: pointer;
	}
</style>
