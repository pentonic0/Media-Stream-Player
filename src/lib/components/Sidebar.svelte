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

<aside class="sidebar h-full flex flex-col glass-panel border-r border-outline-variant">
  <div class="p-6 mb-4">
    <div class="flex items-center gap-3">
      <div class="w-10 h-10 rounded-xl bg-primary flex items-center justify-center shadow-lg shadow-primary/20">
        <Icon icon="mdi:play-circle" class="text-white text-2xl" />
      </div>
      <div class="flex flex-col">
        <span class="text-sm font-bold tracking-tight text-on-surface">Stream Player</span>
        <span class="text-[10px] uppercase tracking-widest text-primary font-semibold">Premium UI</span>
      </div>
    </div>
  </div>

  <nav class="flex-1 px-4 space-y-2">
    {#each navItems as item}
      {@const isActive = activePath === item.href || (item.href !== '/' && activePath.startsWith(item.href))}
      <a
        href={item.href}
        class="flex items-center gap-3 px-4 py-3 rounded-xl transition-all duration-300 group
          {isActive
            ? 'bg-primary/10 text-primary border border-primary/20'
            : 'text-on-surface-variant hover:bg-white/5 hover:text-on-surface'}"
      >
        <div class="relative">
          <Icon icon={item.icon} class="text-xl {isActive ? 'text-primary' : 'group-hover:scale-110 transition-transform'}" />
          {#if isActive}
            <div class="absolute -left-5 top-1/2 -translate-y-1/2 w-1.5 h-6 bg-primary rounded-r-full shadow-[4px_0_12px_rgba(var(--m3-scheme-primary),0.5)]"></div>
          {/if}
        </div>
        <span class="font-medium">{item.name}</span>
      </a>
    {/each}
  </nav>

  <div class="p-4 mt-auto">
    <div class="p-4 rounded-2xl bg-gradient-to-br from-primary/10 to-secondary/5 border border-white/5">
      <div class="flex items-center gap-2 mb-2">
        <Icon icon={infoIcon} class="text-primary text-sm" />
        <span class="text-xs font-semibold text-on-surface">Pro Version</span>
      </div>
      <p class="text-[11px] text-on-surface-variant leading-relaxed">
        Unlock advanced DRM support and cloud sync features.
      </p>
    </div>
  </div>
</aside>

<style>
  .sidebar {
    width: var(--sidebar-width);
    transition: all 0.3s ease;
  }
</style>
