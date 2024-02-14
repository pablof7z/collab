<script lang="ts">
	import { NDKRelay } from '@nostr-dev-kit/ndk';
	import { goto } from '$app/navigation';
	import { ndk } from '$lib/ndk';
	import { NDKEvent, NDKUser, type NDKUserProfile, type NostrEvent } from '@nostr-dev-kit/ndk';
	import { Avatar, Name } from '@nostr-dev-kit/ndk-svelte-components';
	import { onMount } from 'svelte';
	import { Button } from "$lib/components/ui/button";
	import { Textarea } from "$lib/components/ui/textarea";
	import { Input } from "$lib/components/ui/input";
	import * as Card from "$lib/components/ui/card";
	import Label from '@/components/ui/label/label.svelte';
	import ListDocuments from './ListDocuments.svelte';

	let userProfile: NDKUserProfile;
	let authorized: string;
	let authorizedPubkeys: string[] = [];
	let relaysModified = false;
	let relays: string = "";
	let relayUrls: string[] = [];
	let currentUser: NDKUser;

	let title: string = "Nostr collab";
	let kind: string = "31111";

	onMount(() => {
		$ndk.signer?.user().then((user) => {
			currentUser = user;
			user.fetchProfile().then((profile) => {
				if (profile) userProfile = profile;
			});
		});

		$ndk.pool.on("relay:ready", () => {
			if (relaysModified) return;
			relays = $ndk.pool.connectedRelays().map((relay: NDKRelay) => relay.url).join('\n');
		});
	})

	async function gatherUsers() {
		const pubkeys: string[] = [];

		await Promise.all(authorized.split('\n').map(async (addr) => {
			let user: NDKUser | undefined;

			try {
				if (addr.match(/@/)) {
					user = await NDKUser.fromNip05(addr);
				} else if (addr.startsWith('npub')) {
					user = $ndk.getUser({npub: addr});
				} else {
					user = $ndk.getUser({pubkey: addr});
				}
				if (user) { pubkeys.push(user.pubkey); }
			} catch {}
		}));

		authorizedPubkeys = pubkeys;
	}

	async function create() {
		const event = new NDKEvent($ndk, {
			kind: parseInt(kind),
			tags: [
				...authorizedPubkeys.map((pubkey) => (["p", pubkey])),
				["relays", ...relayUrls],
				["title", title],
			]
		} as NostrEvent);
		try {
			await event.publish();
		} catch (e: any) {
			console.error(e.relayErrors);
		}

		event.onRelays = relayUrls;

		goto(`/a/${event.encode()}`);
	}

	$: relayUrls = relays.split(/[\n| |,]+/).filter((url: string) => url && url.trim() !== '');
</script>

{#if userProfile}
	<p>Logged in {userProfile.name} <b>Nostr Address:</b> <code>{userProfile.nip05}</code></p>
{/if}

{#key authorizedPubkeys}
<Card.Root class="mb-6">
	<Card.Header>
		<Card.Title>New Document</Card.Title>
		<Card.Description>Create a new collaborative document</Card.Description>
	</Card.Header>
	<Card.Content>
		<div class="grid w-full items-center gap-4">
			<div class="flex flex-col space-y-1.5">
				<Label for="name">Title</Label>
				<Input placeholder="Name of your document" bind:value={title} />
			</div>
			{#if authorizedPubkeys.length > 0}
				<p>
					Authorized users:
				</p>

				{#each authorizedPubkeys as pubkey}
					<p>
						<Avatar ndk={$ndk} pubkey={pubkey} class="rounded-full w-8 h-8 object-cover" />
						<Name ndk={$ndk} pubkey={pubkey} />
					</p>
				{/each}
			{:else}
				<h3>Authorized users (pubkeys, npubs or nostr addresses)</h3>

				<Textarea bind:value={authorized}/>
			{/if}

			<div class="flex flex-col space-y-1.5">
				<Label for="name">Kind</Label>
				<Input placeholder="Name of your document" bind:value={kind} />
				{#if kind === "30023"}
					<div class="text-xs">
						This will show up in NIP-23 long-form readers like Highlighter, Yakihonne, Habla with your identity.
					</div>
				{:else if parseInt(kind) < 30000 || parseInt(kind) >= 40000}
					<div class="text-xs">
						Needs to be a number between 30000 and 39999
					</div>
				{/if}
			</div>
		</div>
	</Card.Content>
	<Card.Footer>
		{#if authorizedPubkeys.length > 0}
			<Button on:click={create}>
				Create new doc
			</Button>
		{:else}
			<Button on:click={gatherUsers}>
				Select authorized users
			</Button>
		{/if}
	</Card.Footer>
	</Card.Root>
{/key}

<Card.Root>
	<Card.Header>
		<Card.Title>Relays</Card.Title>
		<Card.Description>Which relays do you want to use to collaborate on this document?</Card.Description>
	</Card.Header>

	<Card.Content>
		<div class="flex flex-row gap-6">
			<Textarea bind:value={relays} on:keyup={() => relaysModified = true} class="w-1/2" />

			<ul class="whitespace-nowrap w-1/2">
				{#key relays}
					{#each relayUrls as url}
						<li>{url}</li>
					{/each}
				{/key}
			</ul>
		</div>
	</Card.Content>
</Card.Root>

{#if currentUser}
	<ListDocuments {currentUser} />
{/if}