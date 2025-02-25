<script lang="ts">
	import { getAdminDetails } from '$lib/apis/auths';
	import { onMount, tick, getContext } from 'svelte';

	const i18n = getContext('i18n');

	let adminDetails = null;

	onMount(async () => {
		adminDetails = await getAdminDetails(localStorage.token).catch((err) => {
			console.error(err);
			return null;
		});
	});
</script>

<div class="fixed inset-0 w-full h-full bg-black/80 backdrop-blur-md z-[9999]">
    <div class="fixed inset-0 w-full h-full flex items-center justify-center">
        <div class="w-full sm:max-w-md px-10 flex flex-col">
            <div class="auth-form text-center">
                <!-- Lock icon -->
                <div class="flex justify-center mb-4">
                    <div class="p-4 bg-gray-800 rounded-full border border-gray-700">
                        <svg class="w-12 h-12 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 15v2m-6 4h12a2 2 0 002-2v-6a2 2 0 00-2-2H6a2 2 0 00-2 2v6a2 2 0 002 2zm10-10V7a4 4 0 00-8 0v4h8V7m-6 4h4" />
                        </svg>
                    </div>
                </div>

                <div class="text-2xl font-medium mb-6 text-center">
                    {$i18n.t('Account Verification')}
                </div>

                <div class="mt-4 text-center">
                    <p class="text-base text-gray-400">
                        {$i18n.t('Please complete verification process to activate your account.')}
                    </p>
                </div>

                {#if adminDetails}
                    <div class="mt-6 p-4 bg-white/5 rounded-xl border border-white/10">
                        <div class="text-center">
                            <span class="text-gray-400">{$i18n.t('Contact')}:</span>
                            <div class="font-medium text-gray-300 mt-1">
                                {adminDetails.name} 
                                <span class="text-gray-400">({adminDetails.email})</span>
                            </div>
                        </div>
                    </div>
                {/if}

                <div class="mt-6 space-y-3">
                    <button
                        type="submit"
                        class="w-full h-12 px-6 bg-gray-800 hover:bg-gray-700 text-white font-medium text-base rounded-xl border border-gray-700 border-opacity-50 hover:border-gray-600 transition-all duration-200 shadow-lg shadow-black/20"
                        on:click={async () => {
                            window.location.href = 'https://wa.me/message/TLJFISZTVVTVE1';
                        }}
                    >
                        <span>{$i18n.t('Verify Account')}</span>
                    </button>

                    <button
                        class="w-full text-sm text-gray-500 dark:text-gray-400 hover:text-gray-700 dark:hover:text-gray-200 transition-colors"
                        on:click={async () => {
                            localStorage.removeItem('token');
                            location.href = '/auth';
                        }}
                    >
                        {$i18n.t('Sign Out')}
                    </button>
                </div>
            </div>
        </div>
    </div>
</div>

<style>
    .auth-form {
        @apply flex flex-col justify-center gap-4
               bg-gray-900 bg-opacity-30 /* Changed to match +page.svelte */
               py-6 px-8 rounded-xl
               border border-gray-800 border-opacity-30
               shadow-xl shadow-black/20;
        backdrop-filter: blur(16px);
        -webkit-backdrop-filter: blur(16px);
    }

    :global(body) {
        overflow: hidden;
    }
</style>
