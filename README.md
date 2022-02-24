# sveltekit-hcaptcha
hCaptcha component for Svelte

Inspired by https://github.com/rodneylab/sveltekit-hcaptcha-form - Thank you, it really helped a lot!

## About
A simple component to use hCaptcha in your Svelte powered app.
Tested with SvelteKit but I see no reason why it shouldn't work with plain Svelte with some minor modifications mainly around the use of `$app/env`.
>If you were able to get this working with plain Svelte, please let me know here!


## Usage

### Client Side
>Hcaptcha.svelte

```
<script>
	import { onMount, onDestroy } from 'svelte'
	import { browser } from '$app/env'

	let hcaptcha
	let hcaptchaWidgetID


	export let token = null
	// forces 1-way binding
	let captchaToken
	$: token = captchaToken

	export let isValid = false
	// forces 1-way binding
	let captchaValid
	$: isValid = captchaValid

	onMount(() => {
		setTimeout(function () {
			if (browser) {
				hcaptcha = window.hcaptcha
				if (hcaptcha.render) {
					hcaptchaWidgetID = hcaptcha.render('hcaptcha', {
						sitekey: import.meta.env.VITE_CAPTCHA_KEY,
						size: 'normal',
						callback: onValidCaptcha,
						'error-callback': onErrorCaptcha,
						theme: 'dark'
					})
				}
			}
		}, 500)
	})
	onDestroy(() => {
		if (browser) {
			hcaptcha = null
		}
	})


	function onValidCaptcha(e) {
		//console.log('verified event', e)
		captchaToken = e
		captchaValid = true
	}

	function onErrorCaptcha(e) {
		//console.log('error event', {error: e.error})
		captchaToken = null
		captchaValid = false
		hcaptcha.reset(hcaptchaWidgetID)
	}
</script>


<svelte:head>
	<script src='https://js.hcaptcha.com/1/api.js?render=explicit' async defer></script>
</svelte:head>

<div id='hcaptcha' class='h-captcha border-0'/>

```


Import the component into your page:
```
import Hcaptcha from './Hcaptcha.svelte'
```

Set local variables that will be bound to the component:
```
let captchaValid = false // use this to check frontend validity
let captchaToken  // generated token will be stored here - send this to hCaptcha's validator via POST in your onSubmit method for your form
```

Add the component to your markup:
```
<Hcaptcha bind:token={captchaToken} bind:isValid={captchaValid} />
```

### Backend example
```
export async function post({request}) {
	const data = await request.json()
	const postData = data.formData


	try {
		const captchaToken = postData.captchaToken

		// validate captcha

		const captchaData = {'secret': process.env.CAPTCHA_SECRET, 'response': captchaToken}
		const searchParams = new URLSearchParams(captchaData)
		const submit = await fetch(process.env.CAPTCHA_VERIFY_URL, {
			method: 'POST',
			body: searchParams
		})
		const captchaResults = await submit.json()
		if (!captchaResults.success) {
			throw new Error("Robot check failed. Please try again.")
		}
		
		// do stuff with submitted form data
		
		return {
			body: {status: "success"}
		}
	} catch (e) {
		console.error("error: ", e)
		console.error(e.message)
		if (e.response) {
			console.error(e.response.body)
		}
		return {
			status: 400,
			body: {status: "error", msg: e.message}
		}
	}

}
```
