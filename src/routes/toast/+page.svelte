<script>
    import Toast, { notice } from "$lib/elements/Toast.svelte";

    let button;
</script>

<Toast />

<button
    bind:this={button}
    on:click={() => {
        notice.setNotice("info test", { type: "info", autoHide: false });
    }}>Click</button
>

<button
    on:click={() => {
        notice.setNotice("success test", { type: "success", autoHide: false });
    }}>Click</button
>

<button
    on:click={() => {
        notice.setNotice("success test", { type: "warning", autoHide: false });
    }}>Click</button
>

<button
    on:click={() => {
        notice.setNotice("success test", { type: "error", autoHide: false });
    }}>Click</button
>
