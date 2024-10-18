<script lang="ts">
	import { browser } from '$app/environment';
	export async function load({ }) {
		if (browser) {
			if(location.protocol === 'http:') {
				location.protocol = 'https:';
			}
		}
	}


	import {decrypt, getConversationKey} from 'nostr-tools/nip44'
	import {NPool, NRelay1, NSecSigner} from "@jsr/nostrify__nostrify";
	import type {
		NostrEvent
	} from "@jsr/nostrify__nostrify";
	import {hexToBytes} from "@noble/hashes/utils";
	import QRCode from "qrcode/lib/browser";
	import {type Writable, writable} from "svelte/store";

	let user: string | undefined
	let errorMessage: string | undefined;
	let successMessage: string | undefined;
	let secretKeyHex: string;
	let output: string = ""


	let chestNuts: Writable<Map<string, object>> = writable(new Map<string, object>());

	const relays = [
		'wss://nostrue.com',
		'wss://nostr.cercatrova.me',
		'wss://relay.damus.io'
	];
	const pool = new NPool({
		open(url) {
			return new NRelay1(relays[0]);
		},
		reqRouter: async (filters) => {
			return new Map(relays.map((relay) => {
				return [relay, filters];
			}));
		},
		eventRouter: async event => {
			return relays;
		},
	});

	function addOutput(s: string) {
		output += s + "\n"
	}

	async function getInboxRelays(pubkeyHex: any): Promise<string[]> {
		return ['wss://nostrue.com']  // Fake it till you make it!
	}

	async function getOutboxRelays(pubkeyHex: any): Promise<string[]> {
		return ['wss://nostrue.com'] // Fake it till you make it!
	}

	async function publishEvent(event: NostrEvent){
		let relays: string[] = []

		const pTag: string[] | undefined = getTag(event, "p")

		if(pTag?.[2]){
			relays.push(pTag[2]);
		}

		if(event.kind !== 2747 && event.kind !== 20747){
			// We are the exit node
			relays = await getOutboxRelays(event.pubkey);
		} else {
			const nextRecipient = pTag?.[1];
			if(!nextRecipient){
				throw new Error("p tag is required to forward onion.")
			}

			(await getInboxRelays(nextRecipient)).forEach(inboxRelay => {
				relays.push(inboxRelay);
			})
		}

		relays.forEach((relay) => {
			new NRelay1(relay).event(event);
		})
	}

	async function handle(event: NostrEvent) {
		addOutput(`Received event ${event.id} at ${Date.now()}`)
		console.log(event);

		const conversationKey = getConversationKey(hexToBytes(secretKeyHex), event.pubkey)
		const decryptedContent = decrypt(event.content, conversationKey)

		console.log(decryptedContent)

		const encryptedCashu = getTag(event, 'cashu')?.[1];

		if(encryptedCashu === undefined){
			throw new Error("Onion is rotten because there are no nuts.")
		}
		const cashuToken = decrypt(encryptedCashu, conversationKey)

		const qrUrl = await toQrUrl(cashuToken)
		chestNuts.update(cn => cn.set(cashuToken, {event: event, qrUrl: qrUrl, token: cashuToken}))

		// await delay(Math.random() * 1500)
		await publishEvent(JSON.parse(decryptedContent))
	}

	function delay(ms: number) {
		return new Promise( resolve => setTimeout(resolve, ms) );
	}

	async function submitBob() {
		user = "Bob"
		await submit("a6b665c0cfe6d10f48500f95b81646b211dffcdc7eaaefa50faedcb93721d3f7")
	}
	async function submitAlice() {
		user = "Alice"
		await submit("5ed4916b3f39303159f4ac92bc229850edca0af50ca1e24ab8c93fb577e54060")
	}
	async function submitJoe() {
		user = "Joe"
		await submit("e4360db79a076da6b7c0ab3203eb64b3eea765274f09a21a1dbeb088f2005bea")
	}
	async function submitFrank() {
		user = "Frank"
		await submit("efa7a3d189605b8468464e3d56202dd64885684aec7eb5a6f377bc119b98ed16")
	}
	async function submitCustom() {
		user = "Custom User"
		await submit(secretKeyHex)
	}

	async function submit(presetHex: string) {
		if(presetHex){
			console.log(`using preset hex: ${presetHex}`);
			secretKeyHex = presetHex;
		}

		if (isNullOrEmpty(secretKeyHex)) {
			errorMessage = "Please provide a secret key and repo naddr to start watching a repository."
			return;
		}
		errorMessage = ""

		const signer = new NSecSigner(secretKeyHex);
		const pubkey = await signer.getPublicKey();

		try {
			var filters = [
				{
					kinds: [2747],
					"#p": [pubkey],
					limit: 10,
					since: nostrNow() - 60
				}
			]

			for await (const msg of pool.req(filters, {})) {
				if (msg[0] === 'EVENT') {
					handle(msg[2])
					continue;
				}
				if (msg[0] !== 'EOSE') {
					console.log(`No more historic messages`)
				}
			}
			return;
		} catch (ex) {
			console.log(`Error sending event: ${ex}`)
		}

		errorMessage = "Something went wrong sending the event, please try again later."
	}

	const nostrNow = (): number => Math.floor(Date.now() / 1000);

	export function isNullOrEmpty(input: string | undefined): boolean {
		return !(input && input.trim());
	}

	export function getTag(event: NostrEvent, tagName: string) : string[] | undefined {
		const tagArray =  event.tags.filter(item => item[0] === tagName)

		if(!tagArray || tagArray.length === 0) return undefined;

		return tagArray.reduce(item => item[0]);
	}

	async function toQrUrl(cashu: string): Promise<string> {
		return await QRCode.toDataURL(cashu);
	}
</script>

<svelte:head>
	<title>StensDev</title>
	<meta name="description" content="StensDev Site" />
</svelte:head>

<section>

	<div hidden="{isNullOrEmpty(errorMessage)}" class="alert alert-danger mb-2" role="alert" style="color: red">
		<strong>(!)</strong> {errorMessage}
	</div>
	<div hidden="{isNullOrEmpty(successMessage)}" class="alert alert-danger mb-2" role="alert" style="color: green">
		<strong>(!)</strong> {successMessage}
	</div>
	<input type="text" bind:value={secretKeyHex} placeholder="SecretKey (HEX!)">

	<button type="button" on:click={submitCustom} class="btn btn-blue">Start peeling</button>
	<button type="button" on:click={submitAlice} class="btn btn-blue">I'm Alice</button>
	<button type="button" on:click={submitBob} class="btn btn-blue">I'm Bob</button>
	<button type="button" on:click={submitJoe} class="btn btn-blue">I'm Joe</button>
	<button type="button" on:click={submitFrank} class="btn btn-blue">I'm Frank</button>

	<h1>{user}'s Peels:</h1>
	<table class="table table-xs table-column">
		<tr>
			<td><strong>ChestNut</strong></td>
			<td><strong>Event</strong></td>
		</tr>
		{#each [...$chestNuts] as [cashu, bla]}
			<tr>
				<td><a href="cashu://{bla.token}"><img src="{bla.qrUrl}" alt="Nut"></a> <br><textarea>{cashu}</textarea></td>
				<td><textarea>{JSON.stringify(bla.event, null, 2)}</textarea></td>
			</tr>
		{/each}
	</table>


	<p>{@html output.replace(/\n/g, '<br>')}</p>
</section>

<style>
	section {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		flex: 0.6;
	}

	h1 {
		width: 100%;
	}

	table {
		max-width: 100vw;
		height: 100%;
	}

	textarea {
		box-sizing: border-box;
		width: 100%;
		height: 100%;
	}
	table, td, th {
		border: 1px solid black;
		border-collapse: collapse;
	}
</style>
