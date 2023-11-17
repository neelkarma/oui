<script>
  import { onMount } from "svelte";

  let dots = "";

  onMount(() => {
    const interval = setInterval(() => {
      dots = dots.length < 3 ? dots + "." : "";
    }, 500);
    return () => clearInterval(interval);
  });
</script>

{dots}
