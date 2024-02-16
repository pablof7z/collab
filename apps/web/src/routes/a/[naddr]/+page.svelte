<script lang="ts">
	import { page } from "$app/stores";
	import { ndk } from "$lib/ndk";
	import Button from "@/components/ui/button/button.svelte";
	import Input from "@/components/ui/input/input.svelte";
	import Textarea from "@/components/ui/textarea/textarea.svelte";
	import { NDKEvent, NDKRelay, type NDKFilter, type NostrEvent, NDKSubscription, NDKRelaySet, NDKUser } from "@nostr-dev-kit/ndk";
	import { Avatar, Name } from "@nostr-dev-kit/ndk-svelte-components";
	import { onMount } from "svelte";
	import { derived, type Readable } from "svelte/store";
	import ShowUnauth from "./ShowUnauth.svelte";

    export let naddr: string;

    let start: NDKEvent;

    let events: NDKSubscription | undefined;
    let unauthEvents: NDKSubscription | undefined;
    let latest: NDKEvent;
    let latestAuth: NDKEvent;
    let latestUnauth: NDKEvent;
    let currentUserIsOwner: boolean | undefined;
    let title: string;
    let showUnauthEdits = false;
    let relaySet: NDKRelaySet | undefined;

    let currentUser: NDKUser;
    $ndk.signer?.user().then((user) => {
        currentUser = user;
    });

    $: if (start && currentUser && currentUserIsOwner === undefined) {
        currentUserIsOwner = start.pubkey === currentUser?.pubkey;
    }

    const relaysSet = undefined;

    let mounted = false;
    onMount(() => mounted = true)

    $: if ($page.params.naddr !== naddr && mounted) {
        naddr = $page.params.naddr;

        $ndk.fetchEvent(naddr, { subId: 'initial'}).then((e) => {
            if (!e) return;

            start = e;

            // get relays
            const relayUrls = start.getMatchingTags("relays")?.[0].slice(1);
            console.log({relayUrls});
            if (relayUrls.length > 0) {
                relaySet = NDKRelaySet.fromRelayUrls(relayUrls, $ndk);
            }

            const filter: NDKFilter = {
                kinds: [start.kind!, 24135],
                "#d": [start.tagValue("d")!],
                limit: 1,
                "authors": [
                    start.pubkey,
                    ...start.getMatchingTags("p").map((t) => t[1])
                ]
            };

            let eosed = false;
            events = $ndk.subscribe(filter, { subId: 'updates' }, relaySet);
            events.on("eose", () => eosed = true);
            const unauthFilter = { ...filter };
            delete unauthFilter.authors;
            unauthEvents = $ndk.subscribe(unauthFilter, { subId: 'unauth-updates' }, relaySet);

            events.on("event", (e: NDKEvent) => {
                latest = e;
                latestAuth = e;
                title = e.tagValue("title") ?? "";
            });

            unauthEvents.on("event", (e: NDKEvent) => {
                latestUnauth = e;

                if (showUnauthEdits) {
                    latest = e;
                    title = e.tagValue("title") ?? "";
                }
            });
        });
    }

    let content: string;
    let contentModifiedAt: number = 0;

    $: if (latest && latest.created_at! > contentModifiedAt) {
        if (showUnauthEdits) {
            content = latestUnauth.content;
        } else {
            content = latest.content;
        }
        content ??= "";
    }

    function keydown(e: KeyboardEvent) {
        contentModifiedAt = Math.floor(Date.now() / 1000);
    }

    async function keyup(e: KeyboardEvent) {
        // if it's a letter or number, save the event
        if (e.key.match(/[a-zA-Z0-9]/)) {
            save(24135);
        }
    }

    async function save(kind?: number) {
        const v = new NDKEvent($ndk, {
            kind: kind ?? start.kind!,
            tags: [
                ...start.tags,
                ["ms", Date.now().toString()]
            ],
            content: content
        } as NostrEvent);

        // remove summary
        v.removeTag("summary");
        v.tags.push(["summary", content.slice(0, 100)]);

        // remove title
        v.removeTag("title");
        v.tags.push(["title", title]);

        await v.publish(relaysSet);
    }
</script>

{#if latest}
    <div class="grid w-full items-center gap-4">
        <Input bind:value={title} class="text-3xl font-bold tracking-tight h-auto" />
        <Textarea bind:value={content} on:keydown={keydown} on:keyup={keyup} rows=20 />

        {#key currentUser}
            <Button on:click={() => save()}>
                {#if currentUserIsOwner && latest.pubkey !== currentUser.pubkey}
                    Accept changes from
                    <Name ndk={$ndk} pubkey={latest.pubkey} />
                {:else}
                    Save
                {/if}
            </Button>
        {/key}

        <p>
            relays = {latest.onRelays.map((r) => r.url).join(", ")}
        </p>

        <ShowUnauth bind:value={showUnauthEdits} on:changed={() => {
            if (showUnauthEdits) {
                latest = latestUnauth;
            } else {
                latest = latestAuth;
            }
        }} />
    </div>
{/if}

{#if latestUnauth?.created_at > latest?.created_at}
    <p>
        Unauth change available from
        <Name ndk={$ndk} pubkey={latestUnauth.pubkey} />
    </p>
{/if}

{#if start}
    <h3 class="mt-6">Owner</h3>
    <Avatar ndk={$ndk} pubkey={start.pubkey} class="rounded-full w-8 h-8 object-cover" />

    <h3 class="mt-6">Invited Editors</h3>
    <div class="flex -space-x-4  place-self-end">
        {#each start.getMatchingTags("p") as pubkey}
            <Avatar ndk={$ndk} pubkey={pubkey[1]} class="rounded-full w-8 h-8 object-cover" />
        {/each}
    </div>
{/if}