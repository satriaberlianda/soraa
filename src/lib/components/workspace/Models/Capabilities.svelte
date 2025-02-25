<script lang="ts">
	import { getContext } from 'svelte';
	import Checkbox from '$lib/components/common/Checkbox.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import { marked } from 'marked';
	import { mobile } from '$lib/stores';

	const i18n = getContext('i18n');

	const helpText = {
		vision: $i18n.t('Model accepts image inputs from camera/screen capture'),
		documents: $i18n.t('Model accepts document uploads'),
		usage: $i18n.t(
			'Sends `stream_options: { include_usage: true }` in the request.\nSupported providers will return token usage information in the response when set.'
		),
		citations: $i18n.t('Displays citations in the response'),
		web_search: $i18n.t('Enables web search'),
		image_generation: $i18n.t('Enables image generation'),
		math: $i18n.t('Mathematical expressions and calculations')
	};

	// Custom labels mapping for display
	const capabilityLabels = {
		vision: 'Vision',
		documents: 'Documents',
		usage: 'Usage',
		citations: 'Citations',
		web_search: 'Search',
		image_generation: 'Image',
		math: 'Math'
	};

	export let capabilities: {
		vision?: boolean;
		documents?: boolean;
		usage?: boolean;
		citations?: boolean;
		web_search?: boolean;
		image_generation?: boolean;
		math?: boolean;
	} = {};
</script>

<div>
	<div class="flex w-full justify-between mb-1">
		<div class=" self-center text-sm font-semibold">{$i18n.t('Capabilities')}</div>
	</div>
	<!-- Ubah flex menjadi grid untuk mobile -->
	<div class="grid {$mobile ? 'grid-cols-2 gap-2' : 'grid-cols-4 gap-3'} sm:grid-cols-4">
		{#each Object.keys(capabilities) as capability}
			<div class=" flex items-center gap-2">
				<Checkbox
					state={capabilities[capability] ? 'checked' : 'unchecked'}
					on:change={(e) => {
						capabilities[capability] = e.detail === 'checked';
					}}
				/>

				<div class=" py-0.5 text-sm whitespace-nowrap">
					<Tooltip content={marked.parse(helpText[capability])}>
						{capabilityLabels[capability]}
					</Tooltip>
				</div>
			</div>
		{/each}
	</div>
</div>

<style>
/* Tambahkan CSS untuk memastikan grid responsif */
@media (max-width: 640px) {
    .grid {
        row-gap: 0.75rem;
    }
}
</style>
