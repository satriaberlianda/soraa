<script lang="ts">
	import { DropdownMenu } from 'bits-ui';
	import { flyAndScale } from '$lib/utils/transitions';
	import { getContext, onMount, tick } from 'svelte';

	import { config, user, tools as _tools, mobile } from '$lib/stores';
	import { createPicker } from '$lib/utils/google-drive-picker';
	import { getTools } from '$lib/apis/tools';

	import Dropdown from '$lib/components/common/Dropdown.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import DocumentArrowUpSolid from '$lib/components/icons/DocumentArrowUpSolid.svelte';
	import Switch from '$lib/components/common/Switch.svelte';
	import WrenchSolid from '$lib/components/icons/WrenchSolid.svelte';
	import CameraSolid from '$lib/components/icons/CameraSolid.svelte';

	const i18n = getContext('i18n');

	export let screenCaptureHandler: Function;
	export let uploadFilesHandler: Function;
	export let inputFilesHandler: Function;

	export let uploadGoogleDriveHandler: Function;

	export let selectedToolIds: string[] = [];

	export let onClose: Function;

	export let modelVisionCapable: boolean = false;
	export let modelDocumentCapable: boolean = false;
	export let uploadEnabled: boolean = modelVisionCapable || modelDocumentCapable;

	let tools = {};
	let show = false;

	$: if (show) {
		init();
	}

	let fileUploadEnabled = true;
	$: fileUploadEnabled = $user.role === 'admin' || $user?.permissions?.chat?.file_upload;

	const init = async () => {
		if ($_tools === null) {
			await _tools.set(await getTools(localStorage.token));
		}

		tools = $_tools.reduce((a, tool, i, arr) => {
			if (tool.id !== 'xsrmath') { 
				a[tool.id] = {
					name: tool.name,
					description: tool.meta.description,
					enabled: selectedToolIds.includes(tool.id)
				};
			}
			return a;
		}, {});
	};

	$: hasTools = Object.keys(tools).length > 0;

	const detectMobile = () => {
		const userAgent = navigator.userAgent || navigator.vendor || window.opera;
		return /android|iphone|ipad|ipod|windows phone/i.test(userAgent);
	};

	function handleFileChange(event) {
		const inputFiles = Array.from(event.target?.files);
		if (inputFiles && inputFiles.length > 0) {
			console.log(inputFiles);
			inputFilesHandler(inputFiles);
		}
	}

	const getUploadTooltip = () => {
		if (!uploadEnabled) return $i18n.t('This model does not support file uploads');
		if (modelVisionCapable && modelDocumentCapable) return $i18n.t('Upload Files or Images');
		if (modelVisionCapable) return $i18n.t('Upload Images');
		if (modelDocumentCapable) return $i18n.t('Upload Documents');
		return '';
	};

	const handleUploadClick = () => {
		if (!uploadEnabled) {
			toast.error($i18n.t('Selected model does not support file uploads'));
			return;
		}
		uploadFilesHandler();
	};
</script>

<!-- Hidden file input used to open the camera on mobile -->
<input
	id="camera-input"
	type="file"
	accept="image/*"
	capture="environment"
	on:change={handleFileChange}
	style="display: none;"
/>

<!-- Only show dropdown if there are tools available -->
{#if hasTools}
	<Dropdown
		bind:show
		on:change={(e) => {
			if (e.detail === false) {
				onClose();
			}
		}}
	>
		<Tooltip content={$i18n.t('More')}>
			<slot />
		</Tooltip>

		<div slot="content">
			<DropdownMenu.Content
				class="w-full max-w-[220px] rounded-xl px-1 py-1  border-gray-300/30 dark:border-gray-700/50 z-50 bg-white dark:bg-gray-850 dark:text-white shadow"
				sideOffset={15}
				alignOffset={-8}
				side="top"
				align="start"
				transition={flyAndScale}
			>
				<div class="  max-h-28 overflow-y-auto scrollbar-hidden">
					{#each Object.keys(tools) as toolId}
						<button
							class="flex w-full justify-between gap-2 items-center px-3 py-2 text-sm font-medium cursor-pointer rounded-xl"
							on:click={() => {
								tools[toolId].enabled = !tools[toolId].enabled;
							}}
						>
							<div class="flex-1 truncate">
								<Tooltip
									content={tools[toolId]?.description ?? ''}
									placement="top-start"
									className="flex flex-1 gap-2 items-center"
								>
									<div class="flex-shrink-0">
										<WrenchSolid />
									</div>

									<div class=" truncate">{tools[toolId].name}</div>
								</Tooltip>
							</div>

							<div class=" flex-shrink-0">
								<Switch
									state={tools[toolId].enabled}
									on:change={async (e) => {
										const state = e.detail;
										await tick();
										if (state) {
											selectedToolIds = [...selectedToolIds, toolId];
										} else {
											selectedToolIds = selectedToolIds.filter((id) => id !== toolId);
										}
									}}
								/>
							</div>
						</button>
					{/each}
				</div>

				<hr class="border-black/5 dark:border-white/5 my-1" />
			</DropdownMenu.Content>
		</div>
	</Dropdown>
{/if}
