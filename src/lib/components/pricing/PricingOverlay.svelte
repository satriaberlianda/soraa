<script lang="ts">
    import { fade } from 'svelte/transition';
    import { createEventDispatcher } from 'svelte';
    
    const dispatch = createEventDispatcher();
    export let show = false;
    
    let isMonthly = true;

    $: plans = [
        {
            name: 'Basic',
            price: isMonthly ? '$10' : '$99',
            period: isMonthly ? '/month' : '/year',
            features: [
                '10 AI Conversations/day',
                'Basic Chat Features',
                'Email Support'
            ]
        },
        {
            name: 'Pro',
            price: isMonthly ? '$25' : '$249',
            period: isMonthly ? '/month' : '/year',
            features: [
                'Unlimited AI Conversations',
                'Advanced Chat Features', 
                'Priority Support',
                'Custom AI Models'
            ],
            popular: true
        },
        {
            name: 'Enterprise',
            price: isMonthly ? '$50' : '$499', 
            period: isMonthly ? '/month' : '/year',
            features: [
                'All Pro Features',
                'Custom Integration',
                'Dedicated Account Manager',
                'SLA Guarantee'
            ]
        }
    ];

    function handleUpgrade(plan) {
        // TODO: Implement upgrade logic
        console.log(`Upgrading to ${plan.name} plan`);
    }
</script>

{#if show}
    <div class="fixed inset-0 z-50 flex items-center justify-center p-2 sm:p-4" 
         transition:fade={{duration: 200}}>
        <!-- Backdrop with blur -->
        <div class="absolute inset-0 backdrop-blur-sm bg-black/30"
             on:click={() => dispatch('close')}></div>
        
        <!-- Content -->
        <div class="relative w-full bg-gray-900 rounded-xl p-4 sm:p-6 max-w-4xl shadow-2xl border border-gray-800 max-h-[90vh] overflow-y-auto scrollbar-none"
             transition:fade={{duration: 150}}>
            
            <!-- Close Button -->
            <button class="absolute right-3 top-3 sm:right-4 sm:top-4 text-gray-400 hover:text-gray-200"
                    on:click={() => dispatch('close')}>
                <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5 sm:h-6 sm:w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
                </svg>
            </button>

            <!-- Header -->
            <div class="text-center mb-6 sm:mb-8">
                <h2 class="text-xl sm:text-2xl font-bold text-white mb-2">Upgrade Your Experience</h2>
                <p class="text-sm sm:text-base text-gray-400">Choose the perfect plan for your needs</p>
                
                <!-- Billing Toggle -->
                <div class="mt-4 flex items-center justify-center gap-3">
                    <span class="text-xs sm:text-sm {!isMonthly ? 'text-white font-medium' : 'text-gray-400'}">Annual</span>
                    <button class="relative inline-flex h-5 sm:h-6 w-10 sm:w-11 flex-shrink-0 rounded-full border-2 border-transparent transition-colors duration-200 ease-in-out focus:outline-none 
                                 {isMonthly ? 'bg-gray-700' : 'bg-gray-800'}"
                            on:click={() => isMonthly = !isMonthly}>
                        <span class="translate-x-0 pointer-events-none inline-block h-4 sm:h-5 w-4 sm:w-5 transform rounded-full bg-gray-200 shadow ring-0 transition duration-200 ease-in-out
                                   {isMonthly ? 'translate-x-5' : 'translate-x-0'}"></span>
                    </button>
                    <span class="text-xs sm:text-sm {isMonthly ? 'text-white font-medium' : 'text-gray-400'}">Monthly</span>
                </div>
            </div>

            <!-- Pricing Grid -->
            <div class="grid grid-cols-1 md:grid-cols-3 gap-4 sm:gap-6">
                {#each plans as plan}
                    <div class="relative flex flex-col border border-gray-700 rounded-xl p-4 sm:p-6 transition-transform hover:scale-[1.02] sm:hover:scale-105 
                               {plan.popular ? 'bg-gray-800 ring-2 ring-gray-600' : 'bg-gray-900'}">
                        {#if plan.popular}
                            <div class="absolute -top-3 left-1/2 transform -translate-x-1/2">
                                <span class="bg-gray-700 text-white text-xs font-semibold px-3 py-1 rounded-full">
                                    Most Popular
                                </span>
                            </div>
                        {/if}
                        
                        <h3 class="text-base sm:text-lg font-semibold text-white">{plan.name}</h3>
                        <div class="mt-3 sm:mt-4 flex items-baseline">
                            <span class="text-2xl sm:text-3xl font-bold text-white">{plan.price}</span>
                            <span class="text-sm sm:text-base text-gray-400 ml-1">{plan.period}</span>
                        </div>
                        
                        <ul class="mt-4 sm:mt-6 space-y-2 sm:space-y-3 flex-grow">
                            {#each plan.features as feature}
                                <li class="flex items-center text-sm sm:text-base text-gray-300">
                                    <svg class="h-4 w-4 sm:h-5 sm:w-5 text-gray-500 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7"/>
                                    </svg>
                                    {feature}
                                </li>
                            {/each}
                        </ul>
                        
                        <button class="mt-6 sm:mt-8 w-full py-2 px-4 rounded-lg transition duration-200 text-sm sm:text-base font-medium
                                     {plan.popular ? 
                                     'bg-gray-700 hover:bg-gray-600 text-white' : 
                                     'bg-gray-800 hover:bg-gray-700 text-gray-100'}"
                                on:click={() => handleUpgrade(plan)}>
                            Get Started
                        </button>
                    </div>
                {/each}
            </div>
        </div>
    </div>
{/if}
