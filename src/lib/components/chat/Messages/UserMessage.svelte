<script lang="ts">
	import dayjs from 'dayjs';
	import { toast } from 'svelte-sonner';
	import { tick, getContext, onMount } from 'svelte';

	import { models, settings } from '$lib/stores';
	import { user as _user } from '$lib/stores';
	import { copyToClipboard as _copyToClipboard, formatDate } from '$lib/utils';

	import Name from './Name.svelte';
	import ProfileImage from './ProfileImage.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import FileItem from '$lib/components/common/FileItem.svelte';
	import Markdown from './Markdown.svelte';
	import Image from '$lib/components/common/Image.svelte';
	import localizedFormat from 'dayjs/plugin/localizedFormat';

	const i18n = getContext('i18n');
	dayjs.extend(localizedFormat);

	export let user;

	export let history;
	export let messageId;

	export let siblings;

	export let showPreviousMessage: Function;
	export let showNextMessage: Function;

	export let editMessage: Function;
	export let deleteMessage: Function;

	export let isFirstMessage: boolean;
	export let readOnly: boolean;

	let edit = false;
	let editedContent = '';
	let messageEditTextAreaElement: HTMLTextAreaElement;

	let message = JSON.parse(JSON.stringify(history.messages[messageId]));
	$: if (history.messages) {
		if (JSON.stringify(message) !== JSON.stringify(history.messages[messageId])) {
			message = JSON.parse(JSON.stringify(history.messages[messageId]));
		}
	}

	const copyToClipboard = async (text) => {
		const res = await _copyToClipboard(text);
		if (res) {
			toast.success($i18n.t('Copying to clipboard was successful!'));
		}
	};

	const editMessageHandler = async () => {
		edit = true;
		editedContent = message.content;

		await tick();

		messageEditTextAreaElement.style.height = '';
		messageEditTextAreaElement.style.height = `${messageEditTextAreaElement.scrollHeight}px`;

		messageEditTextAreaElement?.focus();
	};

	const editMessageConfirmHandler = async (submit = true) => {
		editMessage(message.id, editedContent, submit);

		edit = false;
		editedContent = '';
	};

	const cancelEditMessage = () => {
		edit = false;
		editedContent = '';
	};

	const deleteMessageHandler = async () => {
		deleteMessage(message.id);
	};

	onMount(() => {
		// console.log('UserMessage mounted');
	});
</script>

<div class="flex w-full user-message" dir={$settings.chatDirection} id="message-{message.id}">
	{#if !($settings?.chatBubble ?? true)}
		<div class="flex-shrink-0 {($settings?.chatDirection ?? 'LTR') === 'LTR' ? 'mr-3' : 'ml-3'}">
			<ProfileImage
				src={message.user
					? ($models.find((m) => m.id === message.user)?.info?.meta?.profile_image_url ??
						'/user.png')
					: (user?.profile_image_url ?? '/user.png')}
				className={'size-8'}
			/>
		</div>
	{/if}
	<div class="flex-auto w-full max-w-full pl-1">
		{#if !($settings?.chatBubble ?? true)}
			<div>
				<Name>
					{#if message.user}
						{$i18n.t('You')}
						<span class=" text-gray-500 text-sm font-medium">{message?.user ?? ''}</span>
					{:else if $settings.showUsername || $_user.name !== user.name}
						{user.name}
					{:else}
						{$i18n.t('You')}
					{/if}

					{#if message.timestamp}
						<div
							class=" self-center text-xs invisible group-hover:visible text-gray-400 font-medium first-letter:capitalize ml-0.5 translate-y-[1px]"
						>
							<Tooltip content={dayjs(message.timestamp * 1000).format('LLLL')}>
								<span class="line-clamp-1">{formatDate(message.timestamp * 1000)}</span>
							</Tooltip>
						</div>
					{/if}
				</Name>
			</div>
		{/if}

		<div class="chat-{message.role} w-full min-w-full markdown-prose">
			{#if message.files}
				<div class="mt-2.5 mb-1 w-full flex flex-col justify-end overflow-x-auto gap-1 flex-wrap">
					{#each message.files as file}
						<div class={($settings?.chatBubble ?? true) ? 'self-end' : ''}>
							{#if file.type === 'image'}
								<Image src={file.url} imageClassName=" max-h-96 rounded-lg" />
							{:else}
								<FileItem
									item={file}
									url={file.url}
									name={file.name}
									type={file.type}
									size={file?.size}
									colorClassName="bg-white dark:bg-gray-850 "
								/>
							{/if}
						</div>
					{/each}
				</div>
			{/if}

			{#if edit === true}
				<div class=" w-full bg-gray-50 dark:bg-gray-800 rounded-3xl px-5 py-3 mb-2">
					<div class="max-h-96 overflow-auto">
						<textarea
							id="message-edit-{message.id}"
							bind:this={messageEditTextAreaElement}
							class=" bg-transparent outline-none w-full resize-none"
							bind:value={editedContent}
							on:input={(e) => {
								e.target.style.height = '';
								e.target.style.height = `${e.target.scrollHeight}px`;
							}}
							on:keydown={(e) => {
								if (e.key === 'Escape') {
									document.getElementById('close-edit-message-button')?.click();
								}

								const isCmdOrCtrlPressed = e.metaKey || e.ctrlKey;
								const isEnterPressed = e.key === 'Enter';

								if (isCmdOrCtrlPressed && isEnterPressed) {
									document.getElementById('confirm-edit-message-button')?.click();
								}
							}}
						/>
					</div>

					<div class=" mt-2 mb-1 flex justify-between text-sm font-medium">
						<div>
							<button
								id="save-edit-message-button"
								class=" px-4 py-2 bg-gray-50 hover:bg-gray-100 dark:bg-gray-800 dark:hover:bg-gray-700 border dark:border-gray-700 text-gray-700 dark:text-gray-200 transition rounded-3xl"
								on:click={() => {
									editMessageConfirmHandler(false);
								}}
							>
								{$i18n.t('Save')}
							</button>
						</div>

						<div class="flex space-x-1.5">
							<button
								id="close-edit-message-button"
								class="px-4 py-2 bg-white dark:bg-gray-900 hover:bg-gray-100 text-gray-800 dark:text-gray-100 transition rounded-3xl"
								on:click={() => {
									cancelEditMessage();
								}}
							>
								{$i18n.t('Cancel')}
							</button>

							<button
								id="confirm-edit-message-button"
								class=" px-4 py-2 bg-gray-900 dark:bg-white hover:bg-gray-850 text-gray-100 dark:text-gray-800 transition rounded-3xl"
								on:click={() => {
									editMessageConfirmHandler();
								}}
							>
								{$i18n.t('Send')}
							</button>
						</div>
					</div>
				</div>
			{:else}
				<div class="w-full">
					<div class="flex {($settings?.chatBubble ?? true) ? 'justify-end pb-1' : 'w-full'}">
						<div
							class="rounded-3xl {($settings?.chatBubble ?? true)
								? `max-w-full px-5 py-2 bg-gray-50 dark:bg-gray-850 ${
										message.files ? 'rounded-tr-lg' : ''
									}`
								: 'w-full'}"
						>
							{#if message.content}
								<Markdown id={message.id} content={message.content} />
							{/if}
						</div>
					</div>

					<div
						class="flex {($settings?.chatBubble ?? true)
							? 'justify-end'
							: ''} text-gray-600 dark:text-gray-500"
					>
						{#if !($settings?.chatBubble ?? true)}
							{#if siblings.length > 1}
								<div class="flex self-center" dir="ltr">
									<button
										class="self-center p-1 hover:bg-black/5 dark:hover:bg-white/5 dark:hover:text-white hover:text-black rounded-md transition"
										on:click={() => {
											showPreviousMessage(message);
										}}
									>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											fill="none"
											viewBox="0 0 24 24"
											stroke="currentColor"
											stroke-width="2.5"
											class="size-3.5"
										>
											<path
												stroke-linecap="round"
												stroke-linejoin="round"
												d="M15.75 19.5 8.25 12l7.5-7.5"
											/>
										</svg>
									</button>

									<div class="text-sm tracking-widest font-semibold self-center dark:text-gray-100">
										{siblings.indexOf(message.id) + 1}/{siblings.length}
									</div>

									<button
										class="self-center p-1 hover:bg-black/5 dark:hover:bg-white/5 dark:hover:text-white hover:text-black rounded-md transition"
										on:click={() => {
											showNextMessage(message);
										}}
									>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											fill="none"
											viewBox="0 0 24 24"
											stroke="currentColor"
											stroke-width="2.5"
											class="size-3.5"
										>
											<path
												stroke-linecap="round"
												stroke-linejoin="round"
												d="m8.25 4.5 7.5 7.5-7.5 7.5"
											/>
										</svg>
									</button>
								</div>
							{/if}
						{/if}
						{#if !readOnly}
							<Tooltip content={$i18n.t('Edit')} placement="bottom">
								<button
									class="invisible group-hover:visible p-1.5 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg dark:hover:text-white hover:text-black transition edit-user-message-button"
									on:click={() => {
										editMessageHandler();
									}}
								>
									<svg
										xmlns="http://www.w3.org/2000/svg"
										fill="none"
										viewBox="0 0 24 24"
										stroke-width="2.3"
										stroke="currentColor"
										class="w-4 h-4"
									>
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											d="M16.862 4.487l1.687-1.688a1.875 1.875 0 112.652 2.652L6.832 19.82a4.5 4.5 0 01-1.897 1.13l-2.685.8.8-2.685a4.5 4.5 0 011.13-1.897L16.863 4.487zm0 0L19.5 7.125"
										/>
									</svg>
								</button>
							</Tooltip>
						{/if}

						<Tooltip content={$i18n.t('Copy')} placement="bottom">
							<button
								class="invisible group-hover:visible p-1.5 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg dark:hover:text-white hover:text-black transition"
								on:click={() => {
									copyToClipboard(message.content);
								}}
							>
								<svg 
									viewBox="0 0 20 20" 
									fill="none" 
									xmlns="http://www.w3.org/2000/svg" 
									class="w-4 h-4"
									stroke-width="0"
								>
									<g>
										<path d="M5.03 14.64C4.77 14.64 4.5 14.62 4.24 14.56C3.98 14.51 3.73 14.43 3.49 14.33C3.24 14.23 3.01 14.1 2.79 13.96C2.57 13.81 2.37 13.64 2.18 13.45C1.99 13.26 1.82 13.05 1.68 12.83C1.53 12.61 1.4 12.37 1.3 12.13C1.2 11.88 1.13 11.63 1.07 11.36C1.02 11.1 1 10.84 1 10.57L1 5.07C1 4.8 1.02 4.54 1.07 4.27C1.13 4.01 1.2 3.76 1.3 3.51C1.4 3.26 1.53 3.03 1.68 2.81C1.82 2.58 1.99 2.38 2.18 2.19C2.37 2 2.57 1.83 2.79 1.68C3.01 1.53 3.24 1.41 3.49 1.31C3.73 1.2 3.98 1.13 4.24 1.07C4.5 1.02 4.77 1 5.03 1L10.49 1C10.75 1 11.01 1.02 11.27 1.07C11.53 1.13 11.78 1.2 12.03 1.31C12.27 1.41 12.51 1.53 12.73 1.68C12.95 1.83 13.15 2 13.34 2.19C13.53 2.38 13.69 2.58 13.84 2.81C13.99 3.03 14.11 3.26 14.21 3.51C14.31 3.76 14.39 4.01 14.44 4.27C14.5 4.54 14.52 4.8 14.52 5.07L12.94 5.07C12.94 4.91 12.92 4.75 12.89 4.58C12.86 4.43 12.81 4.27 12.75 4.12C12.69 3.97 12.61 3.83 12.52 3.69C12.43 3.56 12.33 3.43 12.22 3.32C12.1 3.2 11.98 3.1 11.85 3.01C11.71 2.92 11.57 2.84 11.42 2.78C11.27 2.72 11.12 2.67 10.96 2.64C10.81 2.61 10.65 2.59 10.49 2.59L5.03 2.59C4.87 2.59 4.71 2.61 4.55 2.64C4.4 2.67 4.24 2.72 4.09 2.78C3.95 2.84 3.8 2.92 3.67 3.01C3.54 3.1 3.41 3.2 3.3 3.32C3.18 3.43 3.08 3.56 2.99 3.69C2.9 3.83 2.83 3.97 2.77 4.12C2.71 4.27 2.66 4.43 2.63 4.58C2.6 4.75 2.58 4.91 2.58 5.07L2.58 10.57C2.58 10.73 2.6 10.89 2.63 11.05C2.66 11.21 2.71 11.37 2.77 11.52C2.83 11.67 2.9 11.81 2.99 11.94C3.08 12.08 3.18 12.2 3.3 12.32C3.41 12.43 3.54 12.54 3.67 12.63C3.8 12.72 3.95 12.79 4.09 12.86C4.24 12.92 4.4 12.96 4.55 13C4.71 13.03 4.87 13.04 5.03 13.04L5.03 14.64Z" fill="currentColor"></path>
										<path d="M14.75 18.91L9.3 18.91C9.03 18.91 8.77 18.88 8.51 18.83C8.25 18.78 8 18.7 7.75 18.6C7.51 18.49 7.27 18.37 7.05 18.22C6.83 18.07 6.63 17.9 6.44 17.71C6.25 17.52 6.09 17.32 5.94 17.1C5.79 16.87 5.67 16.64 5.57 16.39C5.47 16.14 5.39 15.89 5.34 15.63C5.28 15.37 5.26 15.1 5.26 14.83L5.26 9.33C5.26 9.06 5.28 8.8 5.34 8.54C5.39 8.28 5.47 8.02 5.57 7.77C5.67 7.53 5.79 7.29 5.94 7.07C6.09 6.85 6.25 6.64 6.44 6.45C6.63 6.26 6.83 6.09 7.05 5.95C7.27 5.8 7.51 5.67 7.75 5.57C8 5.47 8.25 5.39 8.51 5.34C8.77 5.29 9.03 5.26 9.3 5.26L14.75 5.26C15.01 5.26 15.28 5.29 15.54 5.34C15.8 5.39 16.05 5.47 16.29 5.57C16.54 5.67 16.77 5.8 16.99 5.95C17.21 6.09 17.41 6.26 17.6 6.45C17.79 6.64 17.96 6.85 18.1 7.07C18.25 7.29 18.37 7.53 18.48 7.77C18.58 8.02 18.65 8.28 18.71 8.54C18.76 8.8 18.78 9.06 18.78 9.33L18.78 14.83C18.78 15.1 18.76 15.37 18.71 15.63C18.65 15.89 18.58 16.14 18.48 16.39C18.37 16.64 18.25 16.87 18.1 17.1C17.96 17.32 17.79 17.52 17.6 17.71C17.41 17.9 17.21 18.07 16.99 18.22C16.77 18.37 16.54 18.49 16.29 18.6C16.05 18.7 15.8 18.78 15.54 18.83C15.28 18.88 15.01 18.91 14.75 18.91ZM9.3 6.86C9.13 6.86 8.97 6.87 8.82 6.91C8.66 6.94 8.51 6.98 8.36 7.05C8.21 7.11 8.07 7.18 7.93 7.28C7.8 7.37 7.68 7.47 7.56 7.58C7.45 7.7 7.35 7.82 7.26 7.96C7.17 8.09 7.09 8.24 7.03 8.38C6.97 8.54 6.92 8.69 6.89 8.85C6.86 9.01 6.84 9.17 6.84 9.33L6.84 14.83C6.84 15 6.86 15.16 6.89 15.32C6.92 15.48 6.97 15.63 7.03 15.78C7.09 15.93 7.17 16.07 7.26 16.21C7.35 16.34 7.45 16.47 7.56 16.58C7.68 16.7 7.8 16.8 7.93 16.89C8.07 16.98 8.21 17.06 8.36 17.12C8.51 17.18 8.66 17.23 8.82 17.26C8.97 17.29 9.13 17.31 9.3 17.31L14.75 17.31C14.91 17.31 15.07 17.29 15.23 17.26C15.38 17.23 15.54 17.18 15.69 17.12C15.83 17.06 15.98 16.98 16.11 16.89C16.24 16.8 16.37 16.7 16.48 16.58C16.59 16.47 16.7 16.34 16.79 16.21C16.87 16.07 16.95 15.93 17.01 15.78C17.07 15.63 17.12 15.48 17.15 15.32C17.18 15.16 17.2 15 17.2 14.83L17.2 9.33C17.2 9.17 17.18 9.01 17.15 8.85C17.12 8.69 17.07 8.54 17.01 8.38C16.95 8.24 16.87 8.09 16.79 7.96C16.7 7.82 16.59 7.7 16.48 7.58C16.37 7.47 16.24 7.37 16.11 7.28C15.98 7.19 15.83 7.11 15.69 7.05C15.54 6.98 15.38 6.94 15.23 6.91C15.07 6.87 14.91 6.86 14.75 6.86L9.3 6.86Z" fill="currentColor"></path>
									</g>
								</svg>
							</button>
						</Tooltip>

						<!-- Commented out delete message button
						{#if !isFirstMessage && !readOnly}
							<Tooltip content={$i18n.t('Delete')} placement="bottom">
								<button
									class="invisible group-hover:visible p-1 rounded dark:hover:text-white hover:text-black transition"
									on:click={() => {
										deleteMessageHandler();
									}}
								>
									<svg
										xmlns="http://www.w3.org/2000/svg"
										fill="none"
										viewBox="0 0 24 24"
										stroke-width="2"
										stroke="currentColor"
										class="w-4 h-4"
									>
										<path
											stroke-linecap="round"
											stroke-linejoin="round"
											d="m14.74 9-.346 9m-4.788 0L9.26 9m9.968-3.21c.342.052.682.107 1.022.166m-1.022-.165L18.16 19.673a2.25 2.25 0 0 1-2.244 2.077H8.084a2.25 2.25 0 0 1-2.244-2.077L4.772 5.79m14.456 0a48.108 48.108 0 0 0-3.478-.397m-12 .562c.34-.059.68-.114 1.022-.165m0 0a48.11 48.11 0 0 1 3.478-.397m7.5 0v-.916c0-1.18-.91-2.164-2.09-2.201a51.964 51.964 0 0 0-3.32 0c-1.18.037-2.09 1.022-2.09 2.201v.916m7.5 0a48.667 48.667 0 0 0-7.5 0"
										/>
									</svg>
								</button>
							</Tooltip>
						{/if}
						-->
						{#if $settings?.chatBubble ?? true}
							{#if siblings.length > 1}
								<div class="flex self-center" dir="ltr">
									<button
										class="self-center p-1 hover:bg-black/5 dark:hover:bg-white/5 dark:hover:text-white hover:text-black rounded-md transition"
										on:click={() => {
											showPreviousMessage(message);
										}}
									>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											fill="none"
											viewBox="0 0 24 24"
											stroke="currentColor"
											stroke-width="2.5"
											class="size-3.5"
										>
											<path
												stroke-linecap="round"
												stroke-linejoin="round"
												d="M15.75 19.5 8.25 12l7.5-7.5"
											/>
										</svg>
									</button>

									<div class="text-sm tracking-widest font-semibold self-center dark:text-gray-100">
										{siblings.indexOf(message.id) + 1}/{siblings.length}
									</div>

									<button
										class="self-center p-1 hover:bg-black/5 dark:hover:bg-white/5 dark:hover:text-white hover:text-black rounded-md transition"
										on:click={() => {
											showNextMessage(message);
										}}
									>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											fill="none"
											viewBox="0 0 24 24"
											stroke="currentColor"
											stroke-width="2.5"
											class="size-3.5"
										>
											<path
												stroke-linecap="round"
												stroke-linejoin="round"
												d="m8.25 4.5 7.5 7.5-7.5 7.5"
											/>
										</svg>
									</button>
								</div>
							{/if}
						{/if}
					</div>
				</div>
			{/if}
		</div>
	</div>
</div>
