<script lang="ts">
	import { page } from '$app/stores';

	import Chat from '$lib/components/chat/Chat.svelte';
	import Help from '$lib/components/layout/Help.svelte';
</script>

<svelte:head>
    <title>Sora Intellence</title>
</svelte:head>

<Help />
<Chat chatIdProp={$page.params.id} />
