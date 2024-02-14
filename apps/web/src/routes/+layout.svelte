<script lang="ts">
    import { ndk } from "$lib/ndk";
	import Login from "./Login.svelte";
    import "../app.css";

    $ndk.connect(2000);
</script>

{#if $ndk.signer}
    <slot />
{:else}
    <Login />
{/if}