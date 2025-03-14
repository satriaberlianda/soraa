<script lang="ts">
	import { WEBUI_BASE_URL } from '$lib/constants';
	import { marked } from 'marked';

	import { config, user, models as _models, temporaryChatEnabled } from '$lib/stores';
	import { onMount, getContext } from 'svelte';

	import { blur, fade } from 'svelte/transition';

	import Suggestions from './Suggestions.svelte';
	import { sanitizeResponseContent } from '$lib/utils';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import EyeSlash from '$lib/components/icons/EyeSlash.svelte';

	const i18n = getContext('i18n');

	export let modelIds = [];
	export let models = [];

	export let submitPrompt;

	let mounted = false;
	let selectedModelIdx = 0;

	$: if (modelIds.length > 0) {
		selectedModelIdx = models.length - 1;
	}

	$: models = modelIds.map((id) => $_models.find((m) => m.id === id));

	onMount(() => {
		mounted = true;
	});

	function getGreeting() {
        // Get Jakarta time by adding UTC+7 offset
        const jakartaTime = new Date(new Date().getTime() + (7 * 60 * 60 * 1000));
        const hour = jakartaTime.getHours();
        
        if (hour >= 5 && hour < 12) {
            return $i18n.t('Good Morning');
        } else if (hour >= 12 && hour < 15) {
            return $i18n.t('Good Afternoon');
        } else if (hour >= 15 && hour < 19) {
            return $i18n.t('Good Evening');
        } else {
            return $i18n.t('Good Night');
        }
    }
</script>

{#key mounted}
	<div class="m-auto w-full max-w-[800px] px-8 lg:px-5">
		<div class="flex justify-start">
			<div class="flex -space-x-4 mb-0.5" in:fade={{ duration: 200 }}>
				{#each models as model, modelIdx}
					<button
						on:click={() => {
							selectedModelIdx = modelIdx;
						}}
					>
						<Tooltip
							content={marked.parse(
								sanitizeResponseContent(models[selectedModelIdx]?.info?.meta?.description ?? '')
							)}
							placement="right"
						>
							<img
								crossorigin="anonymous"
								src={model?.info?.meta?.profile_image_url ??
									($i18n.language === 'dg-DG'
										? `/doge.png`
										: `${WEBUI_BASE_URL}/static/favicon.png`)}
								class=" size-[2.7rem] rounded-full border-[1px] border-gray-200 dark:border-none"
								alt="logo"
								draggable="false"
							/>
						</Tooltip>
					</button>
				{/each}
			</div>
		</div>

		{#if $temporaryChatEnabled}
			<Tooltip
				content="This chat won't appear in history and your messages will not be saved."
				className="w-fit"
				placement="top-start"
			>
				<div class="flex items-center gap-2 text-gray-500 font-medium text-lg my-2 w-fit">
					<EyeSlash strokeWidth="2.5" className="size-5" /> Temporary Chat
				</div>
			</Tooltip>
		{/if}

		<div
			class="mt-2 mb-4 text-3xl text-gray-800 dark:text-gray-100 font-medium text-left flex items-center gap-4 font-primary"
		>
			<div>
				<div class=" capitalize line-clamp-1" in:fade={{ duration: 200 }}>
					 {getGreeting()}, {$user.name}
				</div>

				<div in:fade={{ duration: 200, delay: 200 }}>
					<div class=" font-medium text-gray-400 dark:text-gray-500 line-clamp-1 font-p">
						{$i18n.t('How can I help you today?')}
					</div>
				</div>

				{#if models[selectedModelIdx]?.info?.meta?.user}
					<div class="mt-2 text-sm font-normal text-gray-400 dark:text-gray-500">
						{#if models[selectedModelIdx]?.info?.meta?.description}
							<div class="markdown">
								{@html marked.parse(sanitizeResponseContent(models[selectedModelIdx]?.info?.meta?.description))}
							</div>
						{/if}
						By
						{#if models[selectedModelIdx]?.info?.meta?.user.community}
							<a href="https://openwebui.com/m/{models[selectedModelIdx]?.info?.meta?.user.username}">
								{models[selectedModelIdx]?.info?.meta?.user.name || `@${models[selectedModelIdx]?.info?.meta?.user.username}`}
							</a>
						{:else}
							{models[selectedModelIdx]?.info?.meta?.user.name}
						{/if}
					</div>
				{/if}
			</div>
		</div>

		<div class="w-full font-primary" in:fade={{ duration: 200, delay: 300 }}>
			<Suggestions
				className="grid grid-cols-2"
				suggestionPrompts={models[selectedModelIdx]?.info?.meta?.suggestion_prompts ??
					$config?.default_prompt_suggestions ??
					[]}
				on:select={(e) => {
					submitPrompt(e.detail);
				}}
			/>
		</div>
	</div>
{/key}
