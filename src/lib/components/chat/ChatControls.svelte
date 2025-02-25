<script lang="ts">
	import { SvelteFlowProvider } from '@xyflow/svelte';
	import { Pane, PaneResizer } from 'paneforge';
	import { onDestroy, onMount, tick } from 'svelte';
	import { mobile, showControls } from '$lib/stores';
	import EllipsisVertical from '../icons/EllipsisVertical.svelte';

	export let history;
	export let models = [];
	export let chatId = null;
	export let chatFiles = [];
	export let params = {};
	export let files;
	export let modelId;
	export let pane;

	let mediaQuery;
	let largeScreen = false;
	let dragged = false;
	let minSize = 0;

	export const openPane = () => {
		if (parseInt(localStorage?.chatControlsSize)) {
			pane.resize(parseInt(localStorage?.chatControlsSize));
		} else {
			pane.resize(minSize);
		}
	};

	const handleMediaQuery = async (e) => {
		if (e.matches) {
			largeScreen = true;
		} else {
			largeScreen = false;
			pane = null;
		}
	};

	const onMouseDown = (event) => {
		dragged = true;
	};

	const onMouseUp = (event) => {
		dragged = false;
	};

	onMount(() => {
		// listen to resize 1024px
		mediaQuery = window.matchMedia('(min-width: 1024px)');

		mediaQuery.addEventListener('change', handleMediaQuery);
		handleMediaQuery(mediaQuery);

		// Select the container element you want to observe
		const container = document.getElementById('chat-container');

		// initialize the minSize based on the container width
		minSize = Math.floor((350 / container.clientWidth) * 100);

		// Create a new ResizeObserver instance
		const resizeObserver = new ResizeObserver((entries) => {
			for (let entry of entries) {
				const width = entry.contentRect.width;
				// calculate the percentage of 200px
				const percentage = (350 / width) * 100;
				// set the minSize to the percentage, must be an integer
				minSize = Math.floor(percentage);

				if ($showControls) {
					if (pane && pane.isExpanded() && pane.getSize() < minSize) {
						pane.resize(minSize);
					}
				}
			}
		});

		// Start observing the container's size changes
		resizeObserver.observe(container);

		document.addEventListener('mousedown', onMouseDown);
		document.addEventListener('mouseup', onMouseUp);
	});

	onDestroy(() => {
		showControls.set(false);

		mediaQuery.removeEventListener('change', handleMediaQuery);
		document.removeEventListener('mousedown', onMouseDown);
		document.removeEventListener('mouseup', onMouseUp);
	});

	const closeHandler = () => {
		showControls.set(false);
	};

	$: if (!chatId) {
		closeHandler();
	}
</script>

<SvelteFlowProvider>
	{#if !largeScreen}
		{#if $showControls}
			<!-- Empty drawer since we removed all controls -->
		{/if}
	{:else}
		{#if $showControls}
			<PaneResizer class="relative flex w-2 items-center justify-center bg-background group">
				<div class="z-10 flex h-7 w-5 items-center justify-center rounded-sm">
					<EllipsisVertical className="size-4 invisible group-hover:visible" />
				</div>
			</PaneResizer>
		{/if}

		<Pane
			bind:pane
			defaultSize={0}
			onResize={(size) => {
				if ($showControls && pane.isExpanded()) {
					if (size < minSize) {
						pane.resize(minSize);
					}
					localStorage.chatControlsSize = size < minSize ? 0 : size;
				}
			}}
			onCollapse={() => {
				showControls.set(false);
			}}
			collapsible={true}
			class="pt-8"
		>
			{#if $showControls}
				<!-- Empty pane since we removed all controls -->
			{/if}
		</Pane>
	{/if}
</SvelteFlowProvider>
