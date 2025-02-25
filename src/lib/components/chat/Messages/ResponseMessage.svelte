<script lang="ts">
	import { toast } from 'svelte-sonner';
	import dayjs from 'dayjs';

	import { createEventDispatcher } from 'svelte';
	import { onMount, tick, getContext } from 'svelte';

	const i18n = getContext<Writable<i18nType>>('i18n');

	const dispatch = createEventDispatcher();

	import { config, models, settings, user } from '$lib/stores';
	import { synthesizeOpenAISpeech } from '$lib/apis/audio';
	import { imageGenerations } from '$lib/apis/images';
	import {
		copyToClipboard as _copyToClipboard,
		approximateToHumanReadable,
		getMessageContentParts,
		sanitizeResponseContent,
		createMessagesList,
		formatDate
	} from '$lib/utils';
	import { WEBUI_BASE_URL } from '$lib/constants';

	import Name from './Name.svelte';
	import ProfileImage from './ProfileImage.svelte';
	import Skeleton from './Skeleton.svelte';
	import Image from '$lib/components/common/Image.svelte';
	import Tooltip from '$lib/components/common/Tooltip.svelte';
	import RateComment from './RateComment.svelte';
	import WebSearchResults from './ResponseMessage/WebSearchResults.svelte';
	import Sparkles from '$lib/components/icons/Sparkles.svelte';
	import Error from './Error.svelte';
	import Citations from './Citations.svelte';
	import CodeExecutions from './CodeExecutions.svelte';

	import type { Writable } from 'svelte/store';
	import type { i18n as i18nType } from 'i18next';
	import ContentRenderer from './ContentRenderer.svelte';
	import { createNewFeedback, getFeedbackById, updateFeedbackById } from '$lib/apis/evaluations';
	import { getChatById } from '$lib/apis/chats';
	import { generateTags } from '$lib/apis';

	interface MessageType {
		id: string;
		model: string;
		content: string;
		files?: { type: string; url: string }[];
		timestamp: number;
		role: string;
		statusHistory?: {
			done: boolean;
			action: string;
			description: string;
			urls?: string[];
			query?: string;
		}[];
		status?: {
			done: boolean;
			action: string;
			description: string;
			urls?: string[];
			query?: string;
		};
		done: boolean;
		error?: boolean | { content: string };
		sources?: string[];
		code_executions?: {
			uuid: string;
			name: string;
			code: string;
			language?: string;
			result?: {
				error?: string;
				output?: string;
				files?: { name: string; url: string }[];
			};
		}[];
		info?: {
			openai?: boolean;
			prompt_tokens?: number;
			completion_tokens?: number;
			total_tokens?: number;
			eval_count?: number;
			eval_duration?: number;
			prompt_eval_count?: number;
			prompt_eval_duration?: number;
			total_duration?: number;
			load_duration?: number;
			usage?: unknown;
		};
		annotation?: { type: string; rating: number };
	}

	export let chatId = '';
	export let history;
	export let messageId;

	let message: MessageType = JSON.parse(JSON.stringify(history.messages[messageId]));
	$: if (history.messages) {
		if (JSON.stringify(message) !== JSON.stringify(history.messages[messageId])) {
			message = JSON.parse(JSON.stringify(history.messages[messageId]));
		}
	}

	export let siblings;

	export let showPreviousMessage: Function;
	export let showNextMessage: Function;

	export let updateChat: Function;
	export let editMessage: Function;
	export let saveMessage: Function;
	export let rateMessage: Function;
	export let actionMessage: Function;
	export let deleteMessage: Function;

	export let submitMessage: Function;
	export let continueResponse: Function;
	export let regenerateResponse: Function;

	export let addMessages: Function;

	export let isLastMessage = true;
	export let readOnly = false;

	let model = null;
	$: model = $models.find((m) => m.id === message.model);

	let edit = false;
	let editedContent = '';
	let editTextAreaElement: HTMLTextAreaElement;

	let audioParts: Record<number, HTMLAudioElement | null> = {};
	let speaking = false;
	let speakingIdx: number | undefined;

	let loadingSpeech = false;
	let generatingImage = false;

	let showRateComment = false;

	const copyToClipboard = async (text) => {
		const res = await _copyToClipboard(text);
		if (res) {
			toast.success($i18n.t('Copying to clipboard was successful!'));
		}
	};

	const playAudio = (idx: number) => {
		return new Promise<void>((res) => {
			speakingIdx = idx;
			const audio = audioParts[idx];

			if (!audio) {
				return res();
			}

			audio.play();
			audio.onended = async () => {
				await new Promise((r) => setTimeout(r, 300));

				if (Object.keys(audioParts).length - 1 === idx) {
					speaking = false;
				}

				res();
			};
		});
	};

	const toggleSpeakMessage = async () => {
		if (speaking) {
			try {
				speechSynthesis.cancel();

				if (speakingIdx !== undefined && audioParts[speakingIdx]) {
					audioParts[speakingIdx]!.pause();
					audioParts[speakingIdx]!.currentTime = 0;
				}
			} catch {}

			speaking = false;
			speakingIdx = undefined;
			return;
		}

		if (!(message?.content ?? '').trim().length) {
			toast.info($i18n.t('No content to speak'));
			return;
		}

		speaking = true;

		if ($config.audio.tts.engine !== '') {
			loadingSpeech = true;

			const messageContentParts: string[] = getMessageContentParts(
				message.content,
				$config?.audio?.tts?.split_on ?? 'punctuation'
			);

			if (!messageContentParts.length) {
				console.log('No content to speak');
				toast.info($i18n.t('No content to speak'));

				speaking = false;
				loadingSpeech = false;
				return;
			}

			console.debug('Prepared message content for TTS', messageContentParts);

			audioParts = messageContentParts.reduce(
				(acc, _sentence, idx) => {
					acc[idx] = null;
					return acc;
				},
				{} as typeof audioParts
			);

			let lastPlayedAudioPromise = Promise.resolve(); // Initialize a promise that resolves immediately

			for (const [idx, sentence] of messageContentParts.entries()) {
				const res = await synthesizeOpenAISpeech(
					localStorage.token,
					$settings?.audio?.tts?.defaultVoice === $config.audio.tts.voice
						? ($settings?.audio?.tts?.voice ?? $config?.audio?.tts?.voice)
						: $config?.audio?.tts?.voice,
					sentence
				).catch((error) => {
					console.error(error);
					toast.error(`${error}`);

					speaking = false;
					loadingSpeech = false;
				});

				if (res) {
					const blob = await res.blob();
					const blobUrl = URL.createObjectURL(blob);
					const audio = new Audio(blobUrl);
					audio.playbackRate = $settings.audio?.tts?.playbackRate ?? 1;

					audioParts[idx] = audio;
					loadingSpeech = false;
					lastPlayedAudioPromise = lastPlayedAudioPromise.then(() => playAudio(idx));
				}
			}
		} else {
			let voices = [];
			const getVoicesLoop = setInterval(() => {
				voices = speechSynthesis.getVoices();
				if (voices.length > 0) {
					clearInterval(getVoicesLoop);

					const voice =
						voices
							?.filter(
								(v) => v.voiceURI === ($settings?.audio?.tts?.voice ?? $config?.audio?.tts?.voice)
							)
							?.at(0) ?? undefined;

					console.log(voice);

					const speak = new SpeechSynthesisUtterance(message.content);
					speak.rate = $settings.audio?.tts?.playbackRate ?? 1;

					console.log(speak);

					speak.onend = () => {
						speaking = false;
						if ($settings.conversationMode) {
							document.getElementById('voice-input-button')?.click();
						}
					};

					if (voice) {
						speak.voice = voice;
					}

					speechSynthesis.speak(speak);
				}
			}, 100);
		}
	};

	const editMessageHandler = async () => {
		edit = true;
		editedContent = message.content;

		await tick();

		editTextAreaElement.style.height = '';
		editTextAreaElement.style.height = `${editTextAreaElement.scrollHeight}px`;
	};

	const editMessageConfirmHandler = async () => {
		editMessage(message.id, editedContent ? editedContent : '', false);

		edit = false;
		editedContent = '';

		await tick();
	};

	const saveAsCopyHandler = async () => {
		editMessage(message.id, editedContent ? editedContent : '');

		edit = false;
		editedContent = '';

		await tick();
	};

	const cancelEditMessage = async () => {
		edit = false;
		editedContent = '';
		await tick();
	};

	const generateImage = async (message: MessageType) => {
		generatingImage = true;
		const res = await imageGenerations(localStorage.token, message.content).catch((error) => {
			toast.error(`${error}`);
		});
		console.log(res);

		if (res) {
			const files = res.map((image) => ({
				type: 'image',
				url: `${image.url}`
			}));

			saveMessage(message.id, {
				...message,
				files: files
			});
		}

		generatingImage = false;
	};

	let feedbackLoading = false;

	const feedbackHandler = async (rating: number | null = null, details: object | null = null) => {
		feedbackLoading = true;
		console.log('Feedback', rating, details);

		const updatedMessage = {
			...message,
			annotation: {
				...(message?.annotation ?? {}),
				...(rating !== null ? { rating: rating } : {}),
				...(details ? details : {})
			}
		};

		const chat = await getChatById(localStorage.token, chatId).catch((error) => {
			toast.error(`${error}`);
		});
		if (!chat) {
			return;
		}

		const messages = createMessagesList(history, message.id);

		let feedbackItem = {
			type: 'rating',
			data: {
				...(updatedMessage?.annotation ? updatedMessage.annotation : {}),
				model_id: message?.selectedModelId ?? message.model,
				...(history.messages[message.parentId].childrenIds.length > 1
					? {
							sibling_model_ids: history.messages[message.parentId].childrenIds
								.filter((id) => id !== message.id)
								.map((id) => history.messages[id]?.selectedModelId ?? history.messages[id].model)
						}
					: {})
			},
			meta: {
				arena: message ? message.arena : false,
				model_id: message.model,
				message_id: message.id,
				message_index: messages.length,
				chat_id: chatId
			},
			snapshot: {
				chat: chat
			}
		};

		const baseModels = [
			feedbackItem.data.model_id,
			...(feedbackItem.data.sibling_model_ids ?? [])
		].reduce((acc, modelId) => {
			const model = $models.find((m) => m.id === modelId);
			if (model) {
				acc[model.id] = model?.info?.base_model_id ?? null;
			} else {
				// Log or handle cases where corresponding model is not found
				console.warn(`Model with ID ${modelId} not found`);
			}
			return acc;
		}, {});
		feedbackItem.meta.base_models = baseModels;

		let feedback = null;
		if (message?.feedbackId) {
			feedback = await updateFeedbackById(
				localStorage.token,
				message.feedbackId,
				feedbackItem
			).catch((error) => {
				toast.error(`${error}`);
			});
		} else {
			feedback = await createNewFeedback(localStorage.token, feedbackItem).catch((error) => {
				toast.error(`${error}`);
			});

			if (feedback) {
				updatedMessage.feedbackId = feedback.id;
			}
		}

		console.log(updatedMessage);
		saveMessage(message.id, updatedMessage);

		await tick();

		if (!details) {
			showRateComment = true;

			if (!updatedMessage.annotation?.tags) {
				// attempt to generate tags
				const tags = await generateTags(localStorage.token, message.model, messages, chatId).catch(
					(error) => {
						console.error(error);
						return [];
					}
				);
				console.log(tags);

				if (tags) {
					updatedMessage.annotation.tags = tags;
					feedbackItem.data.tags = tags;

					saveMessage(message.id, updatedMessage);
					await updateFeedbackById(
						localStorage.token,
						updatedMessage.feedbackId,
						feedbackItem
					).catch((error) => {
						toast.error(`${error}`);
					});
				}
			}
		}

		feedbackLoading = false;
	};

	const deleteMessageHandler = async () => {
		deleteMessage(message.id);
	};

	$: if (!edit) {
		(async () => {
			await tick();
		})();
	}

	onMount(async () => {
		// console.log('ResponseMessage mounted');

		await tick();
	});
</script>

{#key message.id}
	<div
		class="flex w-full message-{message.id}"
		id="message-{message.id}"
		dir={$settings.chatDirection}
	>
		<div class="{`flex-shrink-0 ${($settings?.chatDirection ?? 'LTR') === 'LTR' ? 'mr-3' : 'ml-3'}`}" style="display: none;">
			<!-- Hide the profile image with display:none -->
			<ProfileImage
				src={model?.info?.meta?.profile_image_url ??
					($i18n.language === 'dg-DG' ? `/doge.png` : `${WEBUI_BASE_URL}/static/favicon.png`)}
				className={'size-8'}
			/>
		</div>

		<div class="flex-auto w-full pl-1">
			<!-- Hide the Name component with display:none -->
			<div style="display: none;">
				<Name>
					<Tooltip content={model?.name ?? message.model} placement="top-start">
						<span class="line-clamp-1">
							{model?.name ?? message.model}
						</span>
					</Tooltip>

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

			<div>
				{#if message?.files && message.files?.filter((f) => f.type === 'image').length > 0}
					<div class="my-2.5 w-full flex overflow-x-auto gap-2 flex-wrap justify-center">
						{#each message.files as file}
							<div class="max-w-[320px] max-h-[320px]"> <!-- Changed from 512px to 320px -->
								{#if file.type === 'image'}
									<Image src={file.url} alt={message.content} />
								{/if}
							</div>
						{/each}
					</div>
				{/if}

				<div class="chat-{message.role} w-full min-w-full markdown-prose">
					<div>
						{#if (message?.statusHistory ?? [...(message?.status ? [message?.status] : [])]).length > 0}
							{@const status = (
								message?.statusHistory ?? [...(message?.status ? [message?.status] : [])]
							).at(-1)}
							{#if !status?.hidden}
								<div class="status-description flex items-center gap-2 py-0.5">
									{#if status?.action === 'web_search' && status?.urls}
										<WebSearchResults {status}>
											<div class="flex flex-col justify-center -space-y-0.5">
												<!-- Comment out spinner in web search status
												<div>
													<Spinner className="size-4" />
												</div> 
												-->
												{#if status?.description.includes('{{count}}')}
													{$i18n.t(status?.description, {
														count: status?.urls.length
													})}
												{:else if status?.description === 'No search query generated'}
													{$i18n.t('No search query generated')}
												{:else if status?.description === 'Generating search query'}
													{$i18n.t('Generating search query')}
												{:else}
													{status?.description}
												{/if}
											</div>
											</WebSearchResults>
									{:else if status?.action === 'knowledge_search'}
										<div class="flex flex-col justify-center -space-y-0.5">
											<div
												class="{status?.done === false
													? 'shimmer'
													: ''} text-gray-500 dark:text-gray-500 text-base line-clamp-1 text-wrap"
											>
												{$i18n.t(`Searching Knowledge for "{{searchQuery}}"`, {
													searchQuery: status.query
												})}
											</div>
										</div>
									{:else}
										<div class="flex flex-col justify-center -space-y-0.5">
											<div
												class="{status?.done === false
													? 'shimmer'
													: ''} text-gray-500 dark:text-gray-500 text-base line-clamp-1 text-wrap"
											>
												<!-- $i18n.t(`Searching "{{searchQuery}}"`) -->
												{#if status?.description.includes('{{searchQuery}}')}
													{$i18n.t(status?.description, {
														searchQuery: status?.query
													})}
												{:else if status?.description === 'No search query generated'}
													{$i18n.t('No search query generated')}
												{:else if status?.description === 'Generating search query'}
													{$i18n.t('Generating search query')}
												{:else}
													{status?.description}
												{/if}
											</div>
										</div>
									{/if}
								</div>
							{/if}
						{/if}

						{#if edit === true}
							<div class="w-full bg-gray-50 dark:bg-gray-800 rounded-3xl px-5 py-3 my-2">
								<textarea
									id="message-edit-{message.id}"
									bind:this={editTextAreaElement}
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

								<div class=" mt-2 mb-1 flex justify-between text-sm font-medium">
									<div>
										<button
											id="save-new-message-button"
											class=" px-4 py-2 bg-gray-50 hover:bg-gray-100 dark:bg-gray-800 dark:hover:bg-gray-700 border dark:border-gray-700 text-gray-700 dark:text-gray-200 transition rounded-3xl"
											on:click={() => {
												saveAsCopyHandler();
											}}
										>
											{$i18n.t('Save As Copy')}
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
											{$i18n.t('Save')}
										</button>
									</div>
								</div>
							</div>
						{:else}
							<div class="w-full flex flex-col relative" id="response-content-container">
								{#if message.content === '' && !message.error}
									<!-- Temporarily hide skeleton shimmer
									<Skeleton />
									-->
								{:else if message.content && message.error !== true}
									<ContentRenderer
										id={message.id}
										{history}
										content={message.content}
										sources={message.sources}
										floatingButtons={message?.done}
										save={!readOnly}
										{model}
										onSourceClick={(e) => {
											console.log(e);
											const sourceButton = document.getElementById(`source-${e}`);

											if (sourceButton) {
												sourceButton.click();
											}
										}}
										onAddMessages={({ modelId, parentId, messages }) => {
											addMessages({ modelId, parentId, messages });
										}}
										on:update={(e) => {
											const { raw, oldContent, newContent } = e.detail;

											history.messages[message.id].content = history.messages[
												message.id
											].content.replace(raw, raw.replace(oldContent, newContent));

											updateChat();
										}}
										on:select={(e) => {
											const { type, content } = e.detail;

											if (type === 'explain') {
												submitMessage(
													message.id,
													`Jelaskan bagian ini dengan detail\n\n"${content}"`
												);
											} else if (type === 'ask') {
												const input = e.detail?.input ?? '';
												submitMessage(message.id, `"${content}"\n"${input}"`);

											}
										}}
									/>
								{/if}

								{#if message?.error}
									<Error content={message?.error?.content ?? message.content} />
								{/if}

								{#if (message?.sources || message?.citations) && (model?.info?.meta?.capabilities?.citations ?? true)}
									<Citations sources={message?.sources ?? message?.citations} />
								{/if}

								{#if message.code_executions}
									<CodeExecutions codeExecutions={message.code_executions} />
								{/if}
							</div>
						{/if}
					</div>
				</div>

				{#if !edit}
					{#if message.done || siblings.length > 1}
						<div
							class=" flex justify-start overflow-x-auto buttons text-gray-600 dark:text-gray-500 mt-2"
						>
							{#if siblings.length > 1}
								<div class="flex self-center min-w-fit" dir="ltr">
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

									<div
										class="text-sm tracking-widest font-semibold self-center dark:text-gray-100 min-w-fit"
									>
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

							{#if message.done}
								{#if !readOnly}
									<Tooltip content={$i18n.t('Copy')} placement="bottom">
										<button
											class="{isLastMessage
												? 'visible'
												: 'invisible group-hover:visible'}  p-1.5 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg dark:hover:text-white hover:text-black transition copy-response-button"
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

									{#if message.usage && model?.info?.meta?.capabilities?.usage}
										<Tooltip
											content={message.usage
												? `<pre>${sanitizeResponseContent(
														JSON.stringify(message.usage, null, 2)
															.replace(/"([^(")"]+)":/g, '$1:')
															.slice(1, -1)
															.split('\n')
															.map((line) => line.slice(2))
															.map((line) => (line.endsWith(',') ? line.slice(0, -1) : line))
															.join('\n')
													)}</pre>`
												: ''}
											placement="bottom"
										>
											<button
												class=" {isLastMessage
													? 'visible'
													: 'invisible group-hover:visible'} p-1.5 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg dark:hover:text-white hover:text-black transition whitespace-pre-wrap"
												on:click={() => {
													console.log(message);
												}}
												id="info-{message.id}"
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
														d="M11.25 11.25l.041-.02a.75.75 0 011.063.852l-.708 2.836a.75.75 0 001.063.853l.041-.021M21 12a9 9 0 11-18 0 9 9 0 0118 0zm-9-3.75h.008v.008H12V8.25z"
													/>
												</svg>
											</button>
										</Tooltip>
									{/if}

									{#if !readOnly}
										{#if $config?.features.enable_message_rating ?? true}
											<Tooltip content={$i18n.t('Good Response')} placement="bottom">
												<button
													class="{isLastMessage
														? 'visible'
														: 'invisible group-hover:visible'} p-1.5 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg {(
														message?.annotation?.rating ?? ''
													).toString() === '1'
														? 'bg-gray-100 dark:bg-gray-800'
														: ''} dark:hover:text-white hover:text-black transition disabled:cursor-progress disabled:hover:bg-transparent"
													disabled={feedbackLoading}
													on:click={async () => {
														await feedbackHandler(1);
														window.setTimeout(() => {
															document
																.getElementById(`message-feedback-${message.id}`)
																?.scrollIntoView();
														}, 0);
													}}
												>
													<svg
														stroke="currentColor"
														fill="none"
														stroke-width="2.3"
														viewBox="0 0 24 24"
														stroke-linecap="round"
														stroke-linejoin="round"
														class="w-4 h-4"
														xmlns="http://www.w3.org/2000/svg"
													>
														<path
															d="M14 9V5a3 3 0 0 0-3-3l-4 9v11h11.28a2 2 0 0 0 2-1.7l1.38-9a2 2 0 0 0-2-2.3zM7 22H4a2 2 0 0 1-2-2v-7a2 2 0 0 1 2-2h3"
														/>
													</svg>
												</button>
											</Tooltip>

											<Tooltip content={$i18n.t('Bad Response')} placement="bottom">
												<button
													class="{isLastMessage
														? 'visible'
														: 'invisible group-hover:visible'} p-1.5 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg {(
														message?.annotation?.rating ?? ''
													).toString() === '-1'
														? 'bg-gray-100 dark:bg-gray-800'
														: ''} dark:hover:text-white hover:text-black transition disabled:cursor-progress disabled:hover:bg-transparent"
													disabled={feedbackLoading}
													on:click={async () => {
														await feedbackHandler(-1);
														window.setTimeout(() => {
															document
																.getElementById(`message-feedback-${message.id}`)
																?.scrollIntoView();
														}, 0);
													}}
												>
													<svg
														stroke="currentColor"
														fill="none"
														stroke-width="2.3"
														viewBox="0 0 24 24"
														stroke-linecap="round"
														stroke-linejoin="round"
														class="w-4 h-4"
														xmlns="http://www.w3.org/2000/svg"
													>
														<path
															d="M10 15v4a3 3 0 0 0 3 3l4-9V2H5.72a2 2 0 0 0-2 1.7l-1.38 9a2 2 0 0 0 2 2.3zm7-13h2.67A2.31 2.31 0 0 1 22 4v7a2.31 2.31 0 0 1-2.33 2H17"
														/>
													</svg>
												</button>
											</Tooltip>
										{/if}

										<Tooltip content={$i18n.t('Regenerate')} placement="bottom">
											<button
												type="button"
												class="{isLastMessage
													? 'visible'
													: 'invisible group-hover:visible'} p-1.5 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg dark:hover:text-white hover:text-black transition regenerate-response-button"
												on:click={() => {
													showRateComment = false;
													regenerateResponse(message);

													(model?.actions ?? []).forEach((action) => {
														dispatch('action', {
															id: action.id,
															event: {
																id: 'regenerate-response',
																data: {
																	messageId: message.id
																}
															}
														});
													});
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
														d="M16.023 9.348h4.992v-.001M2.985 19.644v-4.992m0 0h4.992m-4.993 0l3.181 3.183a8.25 8.25 0 0013.803-3.7M4.031 9.865a8.25 8.25 0 0113.803-3.7l3.181 3.182m0-4.991v4.99"
													/>
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
									{/if}
								{/if}
							{/if}
						</div>
					{/if}

					{#if message.done && showRateComment}
						<RateComment
							bind:message
							bind:show={showRateComment}
							on:save={async (e) => {
								await feedbackHandler(null, {
									...e.detail
								});
							}}
						/>
					{/if}
				{/if}
			</div>
		</div>
	</div>
{/key}

<style>
	.buttons::-webkit-scrollbar {
		display: none; /* for Chrome, Safari and Opera */
	}

	.buttons {
		-ms-overflow-style: none; /* IE and Edge */
		scrollbar-width: none; /* Firefox */
	}
</style>
