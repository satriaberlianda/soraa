<script lang="ts">
    import { fade } from 'svelte/transition';
    import { user } from '$lib/stores';
    import { getContext } from 'svelte';

    const i18n = getContext('i18n');

    function getGreeting() {
        const options = { timeZone: 'Asia/Jakarta', hour: 'numeric', hour12: false };
        const hour = parseInt(new Date().toLocaleString('en-US', options));
        
        if (hour >= 5 && hour < 12) {
            return $i18n.t('Hello, Good Morning');
        } else if (hour >= 12 && hour < 15) {
            return $i18n.t('Hello, Good Afternoon'); 
        } else if (hour >= 15 && hour < 19) {
            return $i18n.t('Hello, Good Evening');
        } else {
            return $i18n.t('Hello, Good Night');
        }
    }
</script>

<style>
    .greeting-text {
        /* Base styles */
        @apply text-2xl font-semibold leading-9;
        
        /* Responsive text sizing */
        @media (max-width: 640px) {
            width: calc(100% - 3rem); /* Memberikan margin 1.5rem di kanan dan kiri */
            margin: 0 auto; /* Center the text */
            font-size: 1.5rem; /* Adjusted to match reference */
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
            padding: 0 1rem; /* Tambahan padding dalam */
        }
    }

    .greeting-container {
        @apply text-center w-full mb-4;
        padding: 0 1.5rem; /* Padding konsisten untuk container */
    }
</style>

<div class="greeting-container">
    <h1 class="greeting-text">
        {getGreeting()}
    </h1>
</div>
