<script>
  import { page } from "$app/stores";
  import Icon from "@iconify/svelte";
  import libraryIcon from "@iconify-icons/mdi/library-shelves";
  import workspaceIcon from "@iconify-icons/mdi/view-dashboard-outline";
  import infoIcon from "@iconify-icons/mdi/information-outline";

  const navItems = [
    { name: "Workspace", href: "/", icon: workspaceIcon },
    { name: "Library", href: "/library", icon: libraryIcon },
  ];

  $: activePath = $page.url.pathname;
</script>

<aside class="sidebar h-full flex flex-col glass-panel border-r border-white/5">
  <div class="p-5 mb-2">
    <div class="flex items-center gap-3">
      <div class="w-9 h-9 rounded-xl bg-gradient-to-br from-primary to-primary-container flex items-center justify-center shadow-lg shadow-primary/20">
        <Icon icon="mdi:play-circle" class="text-white text-xl" />
      </div>
      <div class="flex flex-col">
        <span class="text-sm font-black tracking-tight text-white leading-none">Stream Player</span>
        <span class="text-[9px] uppercase tracking-[0.15em] text-primary/80 font-bold mt-1">Premium UI</span>
      </div>
    </div>
  </div>

  <nav class="flex-1 px-3 space-y-1">
    {#each navItems as item}
      {@const isActive = activePath === item.href || (item.href !== '/' && activePath.startsWith(item.href))}
      <a
        href={item.href}
        class="flex items-center gap-3 px-3 py-2.5 rounded-xl transition-all duration-300 group relative
          {isActive
            ? 'bg-primary/10 text-primary'
            : 'text-on-surface-variant hover:bg-white/[0.03] hover:text-on-surface'}"
      >
        <div class="relative z-10 flex items-center gap-3">
          <Icon icon={item.icon} class="text-lg {isActive ? 'text-primary' : 'group-hover:scale-110 group-hover:text-white transition-all'}" />
          <span class="font-semibold text-sm">{item.name}</span>
        </div>

        {#if isActive}
          <div class="absolute inset-0 bg-primary/5 rounded-xl border border-primary/20"></div>
          <div class="absolute left-0 top-1/2 -translate-y-1/2 w-1 h-5 bg-primary rounded-r-full shadow-[2px_0_8px_rgba(var(--m3-scheme-primary),0.6)]"></div>
        {/if}
      </a>
    {/each}
  </nav>

</aside>

<style>
  .sidebar {
    width: var(--sidebar-width);
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }
</style>
