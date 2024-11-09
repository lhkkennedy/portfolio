<script>
    import { page } from '$app/stores';
    import { onMount } from 'svelte';
    let currentPath = $page.url.pathname
    onMount(() => {
        console.log(currentPath)
    });
</script>

<slot />