<script lang="ts">
	import { getContext } from 'svelte';
	const i18n = getContext('i18n');
	import Tooltip from '../common/Tooltip.svelte';
</script>

<!--
<div class="hidden lg:flex fixed bottom-0 right-0 px-2 py-2 z-20">
	<Tooltip content={$i18n.t('Help')} placement="left">
		<a
			href="mailto:help@intellence.cloud"
			class="text-gray-600 dark:text-gray-300 bg-gray-300/20 size-5 flex items-center justify-center text-[0.7rem] rounded-full"
		>
			?
		</a>
	</Tooltip>
</div>
-->
