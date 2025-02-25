<script>
	import { toast } from 'svelte-sonner';

	import { onMount, getContext } from 'svelte';
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';

	import { getBackendConfig } from '$lib/apis';
	import { ldapUserSignIn, getSessionUser, userSignIn, userSignUp } from '$lib/apis/auths';

	import { WEBUI_API_BASE_URL, WEBUI_BASE_URL } from '$lib/constants';
	import { WEBUI_NAME, config, user, socket } from '$lib/stores';

	import { generateInitialsImage, canvasPixelTest } from '$lib/utils';

	import Spinner from '$lib/components/common/Spinner.svelte';
	import OnBoarding from '$lib/components/OnBoarding.svelte';

	const i18n = getContext('i18n');

	let loaded = false;

	let mode = $config?.features.enable_ldap ? 'ldap' : 'signin';

	let name = '';
	let email = '';
	let password = '';
	let phone = '';

	let ldapUsername = '';

	let showPassword = false;
    
    const togglePassword = () => {
        showPassword = !showPassword;
    };

    const clearFormFields = () => {
        name = '';
        email = '';
        password = '';
        confirmPassword = '';
        ldapUsername = '';
    };

    $: if (mode) {
        clearFormFields();
    }

	const setSessionUser = async (sessionUser) => {
		if (sessionUser) {
			console.log(sessionUser);
			toast.success($i18n.t(`You're now logged in.`));
			if (sessionUser.token) {
				localStorage.token = sessionUser.token;
			}

			$socket.emit('user-join', { auth: { token: sessionUser.token } });
			await user.set(sessionUser);
			await config.set(await getBackendConfig());
			goto('/');
		}
	};

	const signInHandler = async () => {
		const sessionUser = await userSignIn(email, password).catch((error) => {
			toast.error(`${error}`);
			return null;
		});

		await setSessionUser(sessionUser);
	};

	const signUpHandler = async () => {
		const sessionUser = await userSignUp(
			name,
			email,
			password, 
			generateInitialsImage(name),
				phone,
				confirmPassword 
		).catch(
			(error) => {
				toast.error(`${error}`);
				return null;
			}
		);

		await setSessionUser(sessionUser);
	};

	const ldapSignInHandler = async () => {
		const sessionUser = await ldapUserSignIn(ldapUsername, password).catch((error) => {
			toast.error(`${error}`);
			return null;
		});
		await setSessionUser(sessionUser);
	};

	let confirmPassword = '';
    let formSubmitted = false;

    const submitHandler = async () => {
        if (mode === 'ldap') {
            await ldapSignInHandler();
        } else if (mode === 'signin') {
            await signInHandler();
        } else {
            await signUpHandler();
        }
    };

	const checkOauthCallback = async () => {
		if (!$page.url.hash) {
			return;
		}
		const hash = $page.url.hash.substring(1);
		if (!hash) {
			return;
		}
		const params = new URLSearchParams(hash);
		const token = params.get('token');
		if (!token) {
			return;
		}
		const sessionUser = await getSessionUser(token).catch((error) => {
			toast.error(`${error}`);
			return null;
		});
		if (!sessionUser) {
			return;
		}
		localStorage.token = token;
		await setSessionUser(sessionUser);
	};

	let onboarding = false;

	onMount(async () => {
		if ($user !== undefined) {
			await goto('/');
		}
		await checkOauthCallback();

		loaded = true;
		if (($config?.features.auth_trusted_header ?? false) || $config?.features.auth === false) {
			await signInHandler();
		} else {
			onboarding = $config?.onboarding ?? false;
		}
	});

    const passwordVisibility = (node) => {
        return {
            update(showPassword) {
                node.type = showPassword ? 'text' : 'password';
            }
        };
    };

</script>

<svelte:head>
    <title>Sora Intellence</title>
</svelte:head>

<OnBoarding
	bind:show={onboarding}
	getStartedHandler={() => {
		onboarding = false;
		mode = $config?.features.enable_ldap ? 'ldap' : 'signup';
	}}
/>

<style>
    /* Complete autofill fix */
    @-webkit-keyframes autofill {
        0%,100% {
            background: rgb(0 0 0);
            color: white;
        }
    }

    /* Improved autofill styles */
    input:-webkit-autofill,
    input:-webkit-autofill:hover,
    input:-webkit-autofill:focus,
    input:-webkit-autofill:active {
        -webkit-text-fill-color: white !important;
        -webkit-box-shadow: 0 0 0px 1000px rgb(0, 0, 0) inset !important;
        transition: background-color 5000s ease-in-out 0s;
        caret-color: white;
    }

    /* Fixed input styling */
    input {
        @apply w-full h-14 pl-12 pr-6
               bg-black
               border border-gray-700
               rounded-xl
               text-base text-white
               placeholder-gray-400
               shadow-inner
               focus:outline-none focus:ring-0
               focus:border-gray-600
               hover:border-gray-600;
        background-color: rgb(0 0 0) !important;
    }

    .input-container {
        @apply relative flex items-center;
    }

    .input-icon {
        @apply absolute left-4 text-gray-400;
    }

    .password-toggle {
        @apply absolute right-4 text-gray-400 cursor-pointer hover:text-gray-300;
    }

    /* Account toggle styling */
    .account-toggle {
        @apply mt-6 text-center text-sm text-gray-400;
    }

    .account-toggle button {
        @apply text-white hover:text-gray-300 underline font-medium ml-1;
    }

    /* Toggle text styling */
    .toggle-text {
        @apply text-sm text-gray-400 
               hover:text-gray-300 
               transition-colors cursor-pointer
               mt-4;
    }

    /* Form container - remove animation */
    .auth-form {
        @apply flex flex-col justify-center gap-4
               bg-gray-900 bg-opacity-30
               p-8 rounded-xl
               border border-gray-800 border-opacity-30
               shadow-xl shadow-black/20
               backdrop-filter backdrop-blur-md;
    }

    /* Remove fadeIn keyframes since we're not using it */
    /* @keyframes fadeIn {
        from {
            opacity: 0;
            transform: translateY(-20px);
        }
        to {
            opacity: 1;
            transform: translateY(0);
        }
    } */

    /* Submit button styling */
    button[type="submit"] {
        @apply w-full h-14 px-6
               bg-gray-800 hover:bg-gray-700
               text-white font-medium text-base
               rounded-xl
               border border-gray-700 border-opacity-50
               hover:border-gray-600
               transition-all duration-200
               shadow-lg shadow-black/20;
    }

    /* Add these styles */
    .text-input {
        -webkit-text-security: none !important;
        text-security: none !important;
    }

    /* Add this style for password visibility */
    input[type="password"].show-password {
        -webkit-text-security: none;
        text-security: none;
    }

</style>

<div class="w-full h-screen max-h-[100dvh] text-white relative">
	<div class="w-full h-full absolute top-0 left-0 bg-black"></div>

	<div class="w-full absolute top-0 left-0 right-0 h-8 drag-region" />

	{#if loaded}
		<div class="fixed bg-transparent min-h-screen w-full flex justify-center font-primary z-50 text-black dark:text-white">
			<div class="w-full sm:max-w-md px-10 min-h-screen flex flex-col text-center">
				{#if ($config?.features.auth_trusted_header ?? false) || $config?.features.auth === false}
					<div class=" my-auto pb-10 w-full">
						<div
							class="flex items-center justify-center gap-3 text-xl sm:text-2xl text-center font-semibold dark:text-gray-200"
						>
							<div>
								{$i18n.t('Signing in to {{WEBUI_NAME}}', { WEBUI_NAME: $WEBUI_NAME })}
							</div>

							<div>
								<Spinner />
							</div>
						</div>
					</div>
				{:else}
					<div class="  my-auto pb-10 w-full dark:text-gray-100">
							{#if mode === 'signin'}
								 <div>
									<form 
										class="auth-form" 
										on:submit|preventDefault={submitHandler}
										method="post"
										autocomplete="on"
									>
										<div class="text-2xl font-medium mb-6">
											{#if $config?.onboarding ?? false}
												{$i18n.t(`Get started with {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
											{:else if mode === 'ldap'}
												{$i18n.t(`Log in to {{WEBUI_NAME}} with LDAP`, { WEBUI_NAME: $WEBUI_NAME })}
											{:else if mode === 'signin'}
												{$i18n.t(`Log in to {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
											{:else}
												{$i18n.t(`Sign up to {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
											{/if}
										</div>

										<div class="input-container">
											<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
												<path d="M2.003 5.884L10 9.882l7.997-3.998A2 2 0 0016 4H4a2 2 0 00-1.997 1.884z" />
												<path d="M18 8.118l-8 4-8-4V14a2 2 0 002 2h12a2 2 0 002-2V8.118z" />
											</svg>
											<input
												bind:value={email}
												type="email"
												autocomplete="email"
												name="email"
												id="email"
												placeholder={$i18n.t('Enter Your Email')}
												required
											/>
										</div>

										<div class="input-container">
											<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
												<path fill-rule="evenodd" d="M5 9V7a5 5 0 0110 0v2a2 2 0 012 2v5a2 2 0 01-2 2H5a2 2 0 01-2-2v-5a2 2 0 012-2zm8-2v2H7V7a3 3 0 016 0z" clip-rule="evenodd" />
											</svg>
											<input
												bind:value={password}
												type="password"
												use:passwordVisibility={showPassword}
												autocomplete="current-password"
												name="current-password"
												id="current-password"
												placeholder={$i18n.t('Enter Your Password')}
												required
											/>
											<div class="password-toggle" on:click={togglePassword}>
												{#if showPassword}
													<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
														<path fill-rule="evenodd" d="M3.707 2.293a1 1 0 00-1.414 1.414l14 14a1 1 0 001.414-1.414l-1.473-1.473A10.014 10.014 0 0019.542 10C18.268 5.943 14.478 3 10 3a9.958 9.958 0 00-4.512 1.074l-1.78-1.781zm4.261 4.26l1.514 1.515a2.003 2.003 0 012.45 2.45l1.514 1.514a4 4 0 00-5.478-5.478z" clip-rule="evenodd" />
														<path d="M12.454 16.697L9.75 13.992a4 4 0 01-3.742-3.741L2.335 6.578A9.98 9.98 0 00.458 10c1.274 4.057 5.065 7 9.542 7 .847 0 1.669-.105 2.454-.303z" />
													</svg>
												{:else}
													<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
														<path d="M10 12a2 2 0 100-4 2 2 0 000 4z" />
														<path fill-rule="evenodd" d="M.458 10C1.732 5.943 5.522 3 10 3s8.268 2.943 9.542 7c-1.274 4.057-5.064 7-9.542 7S1.732 14.057.458 10zM14 10a4 4 0 11-8 0 4 4 0 018 0z" clip-rule="evenodd" />
													</svg>
												{/if}
											</div>
										</div>

										<div class="mt-2">
											<button type="submit">
												{$i18n.t('Log in')}
											</button>

											{#if mode !== 'ldap' && $config?.features.enable_signup && !($config?.onboarding ?? false)}
												<div class="account-toggle">
													{#if mode === 'signin'}
														<span>{$i18n.t("Don't have an account?")}</span>
														<button type="button" on:click={() => mode = 'signup'}>
															{$i18n.t('Sign up')}
														</button>
													{:else}
														<span>{$i18n.t('Already have an account?')}</span>
														<button type="button" on:click={() => mode = 'signin'}>
															{$i18n.t('Log in')}
														</button>
													{/if}
												</div>
											{/if}
										</div>
									</form>
								</div>
							{:else}
								<div>
									<form class="auth-form" on:submit|preventDefault={submitHandler} autocomplete="off">
										<div class="text-2xl font-medium mb-6">
											{#if $config?.onboarding ?? false}
												{$i18n.t(`Get started with {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
											{:else if mode === 'ldap'}
												{$i18n.t(`Log in to {{WEBUI_NAME}} with LDAP`, { WEBUI_NAME: $WEBUI_NAME })}
											{:else if mode === 'signin'}
												{$i18n.t(`Log in to {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
											{:else}
												{$i18n.t(`Sign up to {{WEBUI_NAME}}`, { WEBUI_NAME: $WEBUI_NAME })}
											{/if}
										</div>

									<div class="flex flex-col gap-4">
										{#if mode === 'signup'}
											<div class="input-container">
												<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
													<path fill-rule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clip-rule="evenodd" />
												</svg>
												<input
													bind:value={name}
													type="text"
													autocomplete="off"
													name="signup-name"
													id="signup-name"
													placeholder={$i18n.t('Enter Your Name')}
													required
												/>
											</div>

											<div class="input-container">
												<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
													<path d="M2.003 5.884L10 9.882l7.997-3.998A2 2 0 0016 4H4a2 2 0 00-1.997 1.884z" />
													<path d="M18 8.118l-8 4-8-4V14a2 2 0 002 2h12a2 2 0 002-2V8.118z" />
												</svg>
												<input
													bind:value={email}
													type="email"
													autocomplete="off"
													name="signup-email"
													id="signup-email" 
													placeholder={$i18n.t('Enter Your Email')}
													required
												/>
											</div>

											<div class="input-container">
												<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
													<path d="M2 3a1 1 0 011-1h2.153a1 1 0 01.986.836l.74 4.435a1 1 0 01-.54 1.06l-1.548.773a11.037 11.037 0 006.105 6.105l.774-1.548a1 1 0 011.059-.54l4.435.74a1 1 0 01.836.986V17a1 1 0 01-1 1h-2C7.82 18 2 12.18 2 5V3z" />
												</svg>
												<input
													bind:value={phone}
													type="tel"
													autocomplete="off"
													name="signup-phone"
													id="signup-phone"
													placeholder={$i18n.t('Enter Your Phone Number')}
													required
												/>
											</div>
										{/if}

										{#if mode === 'ldap'}
											<div class="input-container">
												<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
													<path fill-rule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clip-rule="evenodd" />
												</svg>
												<input
													bind:value={ldapUsername}
													type="text"
													autocomplete="username"
													name="username"
													placeholder={$i18n.t('Enter Your Username')}
													required
												/>
											</div>
										{:else if mode === 'signin'}
											<div class="input-container">
												<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
													<path d="M2.003 5.884L10 9.882l7.997-3.998A2 2 0 0016 4H4a2 2 0 00-1.997 1.884z" />
													<path d="M18 8.118l-8 4-8-4V14a2 2 0 002 2h12a2 2 0 002-2V8.118z" />
												</svg>
												<input
													bind:value={email}
													type="email"
													autocomplete="username"
													name="email"
													id="email"
													placeholder={$i18n.t('Enter Your Email')}
													required
												/>
											</div>
										{/if}

										<div class="input-container">
											<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
												<path fill-rule="evenodd" d="M5 9V7a5 5 0 0110 0v2a2 2 0 012 2v5a2 2 0 01-2 2H5a2 2 0 01-2-2v-5a2 2 0 012-2zm8-2v2H7V7a3 3 0 016 0z" clip-rule="evenodd" />
											</svg>
											<input
												bind:value={password}
												type="password"
												use:passwordVisibility={showPassword}
												autocomplete={mode === 'signup' ? 'off' : 'current-password'}
												name={mode === 'signup' ? 'new-password' : 'password'}
												id={mode === 'signup' ? 'signup-password' : 'password'}
												placeholder={$i18n.t('Enter Your Password')}
												required
											/>
											<div class="password-toggle" on:click={togglePassword}>
												{#if showPassword}
													<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
														<path fill-rule="evenodd" d="M3.707 2.293a1 1 0 00-1.414 1.414l14 14a1 1 0 001.414-1.414l-1.473-1.473A10.014 10.014 0 0019.542 10C18.268 5.943 14.478 3 10 3a9.958 9.958 0 00-4.512 1.074l-1.78-1.781zm4.261 4.26l1.514 1.515a2.003 2.003 0 012.45 2.45l1.514 1.514a4 4 0 00-5.478-5.478z" clip-rule="evenodd" />
														<path d="M12.454 16.697L9.75 13.992a4 4 0 01-3.742-3.741L2.335 6.578A9.98 9.98 0 00.458 10c1.274 4.057 5.065 7 9.542 7 .847 0 1.669-.105 2.454-.303z" />
													</svg>
												{:else}
													<svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
														<path d="M10 12a2 2 0 100-4 2 2 0 000 4z" />
														<path fill-rule="evenodd" d="M.458 10C1.732 5.943 5.522 3 10 3s8.268 2.943 9.542 7c-1.274 4.057-5.064 7-9.542 7S1.732 14.057.458 10zM14 10a4 4 0 11-8 0 4 4 0 018 0z" clip-rule="evenodd" />
													</svg>
												{/if}
											</div>
										</div>

										{#if mode === 'signup'}
											<div class="input-container">
												<svg xmlns="http://www.w3.org/2000/svg" class="input-icon h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
													<path fill-rule="evenodd" d="M5 9V7a5 5 0 0110 0v2a2 2 0 012 2v5a2 2 0 01-2 2H5a2 2 0 01-2-2v-5a2 2 0 012-2zm8-2v2H7V7a3 3 0 016 0z" clip-rule="evenodd" />
												</svg>
												<input
													bind:value={confirmPassword}
													type="password"
													autocomplete="off"
													name="signup-confirm-password"
													id="signup-confirm-password"
													placeholder={$i18n.t('Confirm Your Password')}
													required
												/>
											</div>
										{/if}
									</div>

									<div class="mt-2">
										<button type="submit">
												{#if mode === 'ldap'}
													{$i18n.t('Authenticate')}
												{:else if mode === 'signin'}
													{$i18n.t('Log in')}
												{:else}
													{$i18n.t('Create Account')}
												{/if}
										</button>

										{#if mode !== 'ldap' && $config?.features.enable_signup && !($config?.onboarding ?? false)}
											<div class="account-toggle">
												{#if mode === 'signin'}
													<span>{$i18n.t("Don't have an account?")}</span>
													<button type="button" on:click={() => mode = 'signup'}>
														{$i18n.t('Sign up')}
													</button>
												{:else}
													<span>{$i18n.t('Already have an account?')}</span>
													<button type="button" on:click={() => mode = 'signin'}>
														{$i18n.t('Log in')}
													</button>
												{/if}
											</div>
										{/if}
									</div>
								</form>
							</div>
							{/if}

						{#if Object.keys($config?.oauth?.providers ?? {}).length > 0}
							<div class="inline-flex items-center justify-center w-full">
								<hr class="w-32 h-px my-4 border-0 dark:bg-gray-100/10 bg-gray-700/10" />
								{#if $config?.features.enable_login_form || $config?.features.enable_ldap}
									<span
										class="px-3 text-sm font-medium text-gray-900 dark:text-white bg-transparent"
										>{$i18n.t('or')}</span
									>
								{/if}

								<hr class="w-32 h-px my-4 border-0 dark:bg-gray-100/10 bg-gray-700/10" />
							</div>
							<div class="flex flex-col space-y-2">
								{#if $config?.oauth?.providers?.google}
									<button
										class="flex justify-center items-center bg-gray-700/5 hover:bg-gray-700/10 dark:bg-gray-100/5 dark:hover:bg-gray-100/10 dark:text-gray-300 dark:hover:text-white transition w-full rounded-full font-medium text-sm py-2.5"
										on:click={() => {
											window.location.href = `${WEBUI_BASE_URL}/oauth/google/login`;
										}}
									>
										<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 48 48" class="size-6 mr-3">
											<path
												fill="#EA4335"
												d="M24 9.5c3.54 0 6.71 1.22 9.21 3.6l6.85-6.85C35.9 2.38 30.47 0 24 0 14.62 0 6.51 5.38 2.56 13.22l7.98 6.19C12.43 13.72 17.74 9.5 24 9.5z"
											/><path
												fill="#4285F4"
												d="M46.98 24.55c0-1.57-.15-3.09-.38-4.55H24v9.02h12.94c-.58 2.96-2.26 5.48-4.78 7.18l7.73 6c4.51-4.18 7.09-10.36 7.09-17.65z"
											/><path
												fill="#FBBC05"
												d="M10.53 28.59c-.48-1.45-.76-2.99-.76-4.59s.27-3.14.76-4.59l-7.98-6.19C.92 16.46 0 20.12 0 24c0 3.88.92 7.54 2.56 10.78l7.97-6.19z"
											/><path
												fill="#34A853"
												d="M24 48c6.48 0 11.93-2.13 15.89-5.81l-7.73-6c-2.15 1.45-4.92 2.3-8.16 2.3-6.26 0-11.57-4.22-13.47-9.91l-7.98 6.19C6.51 42.62 14.62 48 24 48z"
											/><path fill="none" d="M0 0h48v48H0z" />
										</svg>
										<span>{$i18n.t('Continue with {{provider}}', { provider: 'Google' })}</span>
									</button>
								{/if}
								{#if $config?.oauth?.providers?.microsoft}
									<button
										class="flex justify-center items-center bg-gray-700/5 hover:bg-gray-700/10 dark:bg-gray-100/5 dark:hover:bg-gray-100/10 dark:text-gray-300 dark:hover:text-white transition w-full rounded-full font-medium text-sm py-2.5"
										on:click={() => {
											window.location.href = `${WEBUI_BASE_URL}/oauth/microsoft/login`;
										}}
									>
										<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 21 21" class="size-6 mr-3">
											<rect x="1" y="1" width="9" height="9" fill="#f25022" /><rect
												x="1"
												y="11"
												width="9"
												height="9"
												fill="#00a4ef"
											/><rect x="11" y="1" width="9" height="9" fill="#7fba00" /><rect
												x="11"
												y="11"
												width="9"
												height="9"
												fill="#ffb900"
											/>
										</svg>
										<span>{$i18n.t('Continue with {{provider}}', { provider: 'Microsoft' })}</span>
									</button>
								{/if}
								{#if $config?.oauth?.providers?.github}
									<button
										class="flex justify-center items-center bg-gray-700/5 hover:bg-gray-700/10 dark:bg-gray-100/5 dark:hover:bg-gray-100/10 dark:text-gray-300 dark:hover:text-white transition w-full rounded-full font-medium text-sm py-2.5"
										on:click={() => {
											window.location.href = `${WEBUI_BASE_URL}/oauth/github/login`;
										}}
									>
										<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" class="size-6 mr-3">
											<path
												fill="currentColor"
												d="M12 0C5.37 0 0 5.37 0 12c0 5.31 3.435 9.795 8.205 11.385.6.105.825-.255.825-.57 0-.285-.015-1.23-.015-2.235-3.015.555-3.795-.735-4.035-1.41-.135-.345-.72-1.41-1.23-1.695-.42-.225-1.02-.78-.015-.795.945-.015 1.62.87 1.845 1.23 1.08 1.815 2.805 1.305 3.495.99.105-.78.42-1.305.765-1.605-2.67-.3-5.46-1.335-5.46-5.925 0-1.305.465-2.385 1.23-3.225-.12-.3-.54-1.53.12-3.18 0 0 1.005-.315 3.3 1.23.96-.27 1.98-.405 3-.405s2.04.135 3 .405c2.295-1.56 3.3-1.23 3.3-1.23.66 1.65.24 2.88.12 3.18.765.84 1.23 1.92 1.23 3.225 0 4.605-2.805 5.625-5.475 5.925.435.375.81 1.095.81 2.22 0 1.605-.015 2.895-.015 3.3 0 .315.225.69.825.57C20.565 21.795 24 17.31 24 12c0-6.63-5.37-12-12-12z"
											/>
										</svg>
										<span>{$i18n.t('Continue with {{provider}}', { provider: 'GitHub' })}</span>
									</button>
								{/if}
								{#if $config?.oauth?.providers?.oidc}
									<button
										class="flex justify-center items-center bg-gray-700/5 hover:bg-gray-700/10 dark:bg-gray-100/5 dark:hover:bg-gray-100/10 dark:text-gray-300 dark:hover:text-white transition w-full rounded-full font-medium text-sm py-2.5"
										on:click={() => {
											window.location.href = `${WEBUI_BASE_URL}/oauth/oidc/login`;
										}}
									>
										<svg
											xmlns="http://www.w3.org/2000/svg"
											fill="none"
											viewBox="0 0 24 24"
											stroke-width="1.5"
											stroke="currentColor"
											class="size-6 mr-3"
										>
											<path
												stroke-linecap="round"
												stroke-linejoin="round"
												d="M15.75 5.25a3 3 0 0 1 3 3m3 0a6 6 0 0 1-7.029 5.912c-.563-.097-1.159.026-1.563.43L10.5 17.25H8.25v2.25H6v2.25H2.25v-2.818c0-.597.237-1.17.659-1.591l6.499-6.499c.404-.404.527-1 .43-1.563A6 6 0 1 1 21.75 8.25Z"
											/>
										</svg>

										<span
											>{$i18n.t('Continue with {{provider}}', {
												provider: $config?.oauth?.providers?.oidc ?? 'SSO'
											})}</span
										>
									</button>
								{/if}
							</div>
						{/if}

						{#if $config?.features.enable_ldap && $config?.features.enable_login_form}
							<div class="mt-2">
								<button
									class="flex justify-center items-center text-xs w-full text-center underline"
									type="button"
									on:click={() => {
										if (mode === 'ldap')
											mode = ($config?.onboarding ?? false) ? 'signup' : 'signin';
										else mode = 'ldap';
									}}
								>
									<span
										>{mode === 'ldap'
											? $i18n.t('Continue with Email')
											: $i18n.t('Continue with LDAP')}</span
									>
								</button>
							</div>
						{/if}
					</div>
				{/if}
			</div>
		</div>
	{/if}
</div>
