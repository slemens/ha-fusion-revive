<script lang="ts">
	export let variant: 'sidebar' | 'card' = 'sidebar';

	const accentA = '#7dd3fc';
	const accentB = '#f472b6';
</script>

<div class="thumbnail" class:card={variant !== 'sidebar'}>
	<div class="header">
		<div class="title"></div>
		<div class="meta"></div>
	</div>

	<svg viewBox="0 0 160 70" preserveAspectRatio="none">
		<defs>
			<linearGradient id="graph-thumb-fill" gradientTransform="rotate(90)">
				<stop offset="0%" stop-color="rgba(255,255,255,0.35)" />
				<stop offset="100%" stop-color="rgba(255,255,255,0.05)" />
			</linearGradient>
		</defs>

		<polyline
			points="0,55 20,45 35,50 55,30 70,35 90,15 110,25 130,20 150,10"
			fill="none"
			stroke={accentA}
			stroke-width="3"
			stroke-linecap="round"
			stroke-linejoin="round"
		/>

		<polyline
			points="0,35 18,40 40,20 65,25 82,32 100,18 120,22 138,12 160,18"
			fill="none"
			stroke={accentB}
			stroke-width="2"
			stroke-linecap="round"
			stroke-linejoin="round"
			opacity="0.85"
		/>
	</svg>
</div>

<style>
	.thumbnail {
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		background: rgba(255, 255, 255, 0.05);
		border-radius: 0.6rem;
		height: 100%;
		padding: 1rem 1.2rem;
		gap: 0.5rem;
	}

	.card {
		background: rgba(255, 255, 255, 0.08);
	}

	.header {
		display: flex;
		flex-direction: column;
		gap: 0.6rem;
	}

	.title {
		width: 60%;
		height: 0.55rem;
		border-radius: 999px;
		background: rgba(255, 255, 255, 0.4);
	}

	.meta {
		width: 35%;
		height: 0.35rem;
		border-radius: 999px;
		background: rgba(255, 255, 255, 0.25);
	}

	svg {
		width: 100%;
		height: 3.5rem;
	}
</style>
