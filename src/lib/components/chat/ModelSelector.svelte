<script lang="ts">
	import { models, showSettings, settings, user, mobile, config } from '$lib/stores';
	import { onMount, tick, getContext } from 'svelte';
	import { toast } from 'svelte-sonner';
	import Selector from './ModelSelector/Selector.svelte';
	import Tooltip from '../common/Tooltip.svelte';

	const i18n = getContext('i18n');

	export let selectedModels = [''];
	export let disabled = false;

	$: if (selectedModels.length !== 1) {
		selectedModels = [selectedModels[0]];
	}

	// Initialize with default model from settings
	onMount(async () => {
		if ($settings?.models?.length > 0) {
			// Check if default model exists in available models
			const defaultModel = $settings.models[0];
			if ($models.some(m => m.id === defaultModel)) {
				selectedModels = [defaultModel];
			} else {
				// If default model doesn't exist, try to use first available model
				if ($models.length > 0) {
					selectedModels = [$models[0].id];
				}
			}
		} else if ($models.length > 0) {
			// No default model set, use first available model
			selectedModels = [$models[0].id];
		}
	});

	// Save model when selection changes
	$: if (selectedModels[0] && selectedModels[0] !== '') {
		saveDefaultModel(selectedModels[0]);
	}

	// Add function to save default model
	const saveDefaultModel = async (modelId: string) => {
		if (!$settings) {
			settings.set({});
		}
		
		const updatedSettings = {
			...$settings,
			models: [modelId]
		};
		
		settings.set(updatedSettings);
		localStorage.setItem('settings', JSON.stringify(updatedSettings));
	};
</script>

<style lang="postcss">
  :global(.model-item) {
    @apply flex items-center justify-between px-3 py-2 transition-all duration-200;
  }

  :global(.model-item:not(.selected):hover) {
    @apply bg-gray-100 dark:bg-gray-700 rounded-md;
  }

  :global(.model-item.selected) {
    @apply bg-blue-50 dark:bg-blue-900/30;
  }

  :global(.model-check) {
    @apply w-4 h-4 text-blue-600 dark:text-blue-400 opacity-0 transition-opacity duration-200;
  }

  :global(.model-item.selected .model-check) {
    @apply opacity-100;
  }

  :global(.model-item:not(.selected):hover .model-check) {
    @apply opacity-50;
  }
</style>

<div class="flex flex-col w-full items-start">
	{#each selectedModels as selectedModel, selectedModelIdx}
		<div class="flex w-full max-w-fit">
			<div class="overflow-hidden w-full">
				<div class="mr-1 max-w-full">
					<Selector
						id={`${selectedModelIdx}`}
						placeholder={$i18n.t('Select a model')}
						items={$models
							.filter(model => !['sora-assistant', 'gemini-2.0-flash'].includes(model.id))
							.map((model) => ({
								value: model.id,
								label: model.name,
								model: model
							}))}
						showTemporaryChatControl={$user.role === 'user'
							? ($user?.permissions?.chat?.temporary ?? true)
							: true}
						bind:value={selectedModel}
					/>
				</div>
			</div>
		</div>
	{/each}
</div>
