<script>
  import { toastStack } from "$lib/toast.svelte.js";
  import { flip } from "svelte/animate";
  import { fade, fly } from "svelte/transition";
</script>

<div class="fixed bottom-10 left-1/2 -translate-x-1/2 z-[3000] flex flex-col items-center gap-3 pointer-events-none">
  {#each toastStack as toast (toast.id)}
    <div
      in:fly={{ y: 20, duration: 300 }}
      out:fade={{ duration: 200 }}
      class="px-6 py-3 rounded-2xl bg-surface-container/80 border border-white/10 shadow-2xl backdrop-blur-xl flex items-center gap-3 pointer-events-auto"
    >
      <div class="w-2 h-2 rounded-full bg-primary animate-pulse"></div>
      <span class="text-xs font-bold text-white whitespace-nowrap">{toast.message}</span>
    </div>
  {/each}
</div>
