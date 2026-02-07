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

  <div class="p-4 mt-auto">
    <div class="relative overflow-hidden p-4 rounded-2xl bg-gradient-to-br from-white/[0.05] to-transparent border border-white/10 group">
      <div class="absolute -right-4 -top-4 w-16 h-16 bg-primary/10 rounded-full blur-2xl group-hover:bg-primary/20 transition-all duration-500"></div>

      <div class="flex items-center gap-2 mb-2 relative z-10">
        <div class="w-5 h-5 rounded-md bg-primary/20 flex items-center justify-center">
          <Icon icon={infoIcon} class="text-primary text-[10px]" />
        </div>
        <span class="text-[11px] font-bold text-white tracking-wide">Pro Version</span>
      </div>
      <p class="text-[10px] text-on-surface-variant leading-relaxed relative z-10">
        Unlock advanced <span class="text-primary/80">DRM support</span> and cloud sync features.
      </p>

      <button class="mt-3 w-full py-1.5 rounded-lg bg-primary/10 hover:bg-primary/20 text-[10px] font-bold text-primary border border-primary/20 transition-all relative z-10">
        Upgrade Now
      </button>
    </div>
  </div>
</aside>

<style>
  .sidebar {
    width: var(--sidebar-width);
    transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  }
</style>
