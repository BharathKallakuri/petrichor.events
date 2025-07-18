<script lang="ts">
	import '../app.css'
	import Loading from '$lib/components/Loading.svelte';
	import PopUpBox from '$lib/components/PopUpBox.svelte';
	import { PopUp } from '$lib/PopUp';
	import { onMount, setContext } from 'svelte';
	import Background from '$lib/components/Background.svelte';
	import { page } from '$app/stores';
	import { POST, API, defaultUser } from '$lib/index.js';
	import { access_token, invalidate, loggedIn, user } from '$lib/stores.js';
	import { beforeNavigate, goto } from '$app/navigation';

	import { Header, Footer, BtpBtn } from '$lib/components/ui';
	// Dummy data in this helper file
	import { footerLinks, headerLinks } from '$lib/helper';
	import { fade, fly } from 'svelte/transition';
	import { events_data } from '$lib/new_data.js';

	let path: string;
	export let data;
	let windowX: number;
	let loading = false;
	let PopUpObj = new PopUp('', '', false, null);

	beforeNavigate(async () => {
		let access_token = $access_token;
		if (access_token == null || access_token == "") {
			access_token = data.access_token
		}
		if (access_token == null || !$loggedIn) {
			invalidate.set(true);
			sessionStorage.setItem('loggedIn', 'false');
			sessionStorage.setItem('user', JSON.stringify(defaultUser));
		}
		if (($invalidate || !$loggedIn) && (access_token != null && access_token != "")) {
			await whoami(access_token ?? 'a');
		}
	});

	async function getData() {
		const session_data = sessionStorage.getItem('user') ?? JSON.stringify(defaultUser);
		user.set(JSON.parse(session_data));
		loggedIn.set(Boolean(sessionStorage.getItem('loggedIn')) ?? false);
		
		let access_token_saved = sessionStorage.getItem('access');
		if (access_token_saved == "" || access_token_saved == null) {
			access_token_saved = data.access_token
		}
		access_token.set(access_token_saved);
		if (($invalidate || !$loggedIn) && (access_token_saved != null && access_token_saved != "")) {
			console.log("Drop")
			await whoami(access_token_saved);	
		}
		if (session_data == JSON.stringify(defaultUser) || session_data == '{}') {
			if ($access_token == null || $access_token == undefined || !$loggedIn) {
				// user have not logged In no need for refreshing the data through whoami

				loggedIn.set(false);
				invalidate.set(true);
			} else {
				// user may have logged in
				// console.log("HH")
				await whoami(access_token_saved);
			}
		} else { 
			loggedIn.set(Boolean(sessionStorage.getItem('loggedIn')) ?? false);
			invalidate.set(false);
		}
	}

	async function whoami(accessToken: string) {
		// console.log("Reloading" + accessToken)
		return await POST(
			API.whoami,
			{
				getUser: true,
				getEvents: true
			},
			accessToken
		)
			.then((res) => res.json())
			.then((res) => {
				// console.log(res.user_data)
				if (res.status == 200) {
					// invalidate
					invalidate.set(false);
					loggedIn.set(true);
					access_token.set(accessToken);
					const events = res.user_events.map((ele: string) => {
						// @ts-ignore
						// console.log(ele)
						const event_type = ele.eventId.at(0);
						let eventData = events_data[event_type].events[ele.eventId];
						if (eventData == null) {
							eventData = {
								name: 'Some Event',
								image: 'https://picsum.photos/200/300',
								id: 'T'
							};
						}
						// @ts-ignore
						eventData['verified'] = ele.verified;
						return eventData;
					});
					const user_content = {
						user_data: res.user_data,
						user_events: events
					};
					user.set(user_content);
					// console.log($user)
					sessionStorage.setItem('user', JSON.stringify($user));
					sessionStorage.setItem('loggedIn', 'true');
					sessionStorage.setItem('access', accessToken);
					return true;
				} else {
					invalidate.set(true);
					sessionStorage.setItem('loggedIn', 'false');
					sessionStorage.setItem('user', JSON.stringify(defaultUser));
					return false;
				}
			})
			.catch((err) => {
				// console.log(err.toString());
				invalidate.set(true);
				sessionStorage.setItem('loggedIn', 'false');
				sessionStorage.setItem('user', JSON.stringify(defaultUser));
				return false;
			});
	}

	const title: string = 'Petrichor 25';

	let winsize = 3000;
	onMount(async () => {
		/// disable reload on older browsers
		document.addEventListener('touchstart', () => {}, { passive: false });
		document.addEventListener('touchmove', () => {}, { passive: false });
		/// disable navigation in older browsers
		if (window.safari) {
			history.pushState(null, null, location.href);
			window.onpopstate = function (event) {
				history.go(1);
			};
		}
		///
		await getData();
		windowX = window.innerWidth;
		window.onresize = () => {
			windowX = window.innerWidth;
		};
		preventReload()
	});

	function preventReload() {
		if (path.includes('/payment/register')) {
			if (window != undefined) {
				window.onbeforeunload = (e) => {
					e.preventDefault();
					if (
						confirm('You may lose the participants added to the event. Do you want to continue? ')
					) {
						e.returnValue = true;
					}
					e.returnValue = false;
				};
			}
		} else {
			if (window != undefined) {
				window.onbeforeunload = () => {}
			}
		}
	}

	function updateLoading(val: boolean) {
		loading = val;
	}

	/**
	 * display a popUp Box
	 * @param title
	 * @param message
	 * @param timeOutTime
	 */
	const displayPopUp = (
		title: string,
		message: string,
		timeOutTime: number,
		callback: CallableFunction
	) => {
		// console.log("cal" + callback)
		// temporary condition until other modes are made
		PopUpObj.title = title;
		PopUpObj.message = message;
		PopUpObj.afterEnd = callback;
		PopUpObj.totalTime = timeOutTime;
		PopUpObj.isOn = true;
		return;
	};
	setContext('displayPopUp', displayPopUp);
	setContext('loading', updateLoading);
	setContext('getData', getData);
	setContext('whoami', whoami);
	$: {
		path = $page.url.pathname;
	}
</script>

<Loading spinning={loading} />

{#if PopUpObj.isOn}
	<PopUpBox bind:PopUpObj />
{/if}

{#if path != '/'}
	<Header {title} links={headerLinks} />
	{/if}

<Background {path} />
<!-- svelte-ignore a11y-click-events-have-key-events -->
<!-- svelte-ignore a11y-no-static-element-interactions -->
{#key data.url}
	<div class="main" in:fade={{ duration: 400, delay: 400 }} out:fly={{ x: windowX, duration: 400 }}>
		<slot />
	</div>
{/key}

{#if path != '/' && path != '/home' && path.startsWith('events')}
	<Footer {title} links={footerLinks} />
	<BtpBtn />
{/if}

<style>
</style>
