<script lang="ts">
	import { ndk } from "$lib/ndk";
	import type { NDKUserProfile } from "@nostr-dev-kit/ndk";
	import { onMount } from "svelte";

    let userProfile: NDKUserProfile;

    onMount(() => {
		$ndk.signer?.user().then((user) => {
			user.fetchProfile().then((profile) => {
				if (profile) userProfile = profile;
			});
		});
	})
</script>

{#if userProfile}
    <p class="my-6">Logged in {userProfile.name}</p>
{/if}

<slot />