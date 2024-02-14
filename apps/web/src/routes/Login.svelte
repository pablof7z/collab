<script lang="ts">
	import { ndk } from "$lib/ndk";
	import { NDKNip07Signer } from "@nostr-dev-kit/ndk";
	import { onMount } from "svelte";

    onMount(() => {
        nip07();
    })

    async function nip07() {
        const signer = new NDKNip07Signer();
        const user = await signer.blockUntilReady();

        if (user) {
            $ndk.signer = signer;
            $ndk = $ndk
        }
    }
</script>

<button class="" on:click={nip07}>
    Login with NIP-07
</button>