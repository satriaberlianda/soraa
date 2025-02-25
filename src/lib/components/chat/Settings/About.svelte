<script lang="ts">
	import { getOllamaVersion } from '$lib/apis/ollama';
	import { WEBUI_NAME } from '$lib/stores';
	import { onMount, getContext } from 'svelte';

	const i18n = getContext('i18n');
	let ollamaVersion = '';

	onMount(async () => {
		ollamaVersion = await getOllamaVersion(localStorage.token).catch((error) => {
			return '';
		});
	});
</script>

<div class="flex flex-col h-full justify-between space-y-3 text-sm mb-6">
	<div class="space-y-3 overflow-y-scroll max-h-[28rem] lg:max-h-full">
		<div>
			<div class="mb-2.5 text-sm font-medium flex space-x-2 items-center">
				<div>
					About
				</div>
			</div>
		</div>

		<div class="prose dark:prose-invert max-w-none text-sm">
			<p>
				Intellence is a research team focused on developing advanced AI models that aim to shape the
				future of technology. Our mission is to create intelligent, safe, and responsible AI, making it
				accessible for a wide range of users. We offer a variety of models that serve different
				purposes, such as:
			</p>

			<ul class="list-disc pl-6 mt-2">
				<li>
					<strong>Sora 2.1Turbo:</strong> Built for fast, accurate language tasks across diverse domains,
					it handles everything from simple queries to complex responses with ease.
				</li>
				<li>
					<strong>Sora 3.4Compute:</strong> Specializes in heavy computational work, making it perfect for
					data analysis, complex calculations, and in-depth information processing.
				</li>
				<li>
					<strong>Sora Reasoner:</strong> Equipped with advanced reasoning capabilities, this model is ideal
					for tasks that require logical thinking, decision-making, and deep problem solving.
				</li>
				<li>
						<strong>Xsr Image:</strong> A powerful tool for generating images, 
						which empowers creative projects with AI-generated visuals.
				</li>
			</ul>

			<p>
				Sora Intellence is the platform that ties everything together, offering an easy-to-use
				interface for interacting with our models. Whether you're looking to integrate these
				capabilities into your own applications or explore them personally, Sora Intellence is
				available for both personal and commercial use via API. From content generation to image
				processing, our models cover a wide array of functions to enhance productivity and creativity.
			</p>

			<p class="mt-4">We invite you to try out Sora Intellence and share your thoughts with us!</p>

			<p class="mt-4">
				Feedback Email: <a href="mailto:feedback@intellence.com">feedback@intellence.com</a>
			</p>
		</div>

		{#if ollamaVersion}
			<hr class="dark:border-gray-850" />
			<div>
				<div class="mb-2.5 text-sm font-medium">{$i18n.t('Ollama Version')}</div>
				<div class="flex w-full">
					<div class="flex-1 text-xs text-gray-700 dark:text-gray-200">
						{ollamaVersion ?? 'N/A'}
					</div>
				</div>
			</div>
		{/if}
	</div>
</div>
