<script>
  import { onDestroy } from "svelte";
  import { getCurrentWindow, LogicalSize } from "@tauri-apps/api/window";

  const isTauri = typeof window !== 'undefined' && window.__TAURI_INTERNALS__;
  const appWindow = isTauri ? getCurrentWindow() : null;
  let isMaximized = false;
  let windowResizeUnListen;

  async function tauriResizeEvent() {
    if (!isTauri || !appWindow) return;
    await appWindow.setMinSize(new LogicalSize(900, 640));
    // @ts-ignore
    windowResizeUnListen = await appWindow.onResized(async () => {
      isMaximized = await appWindow.isMaximized();
    });
  }

  if (isTauri) {
    tauriResizeEvent();
  }

  onDestroy(async () => {
    await windowResizeUnListen?.();
  });
</script>

<div class="flex items-center h-[var(--navbar-height)] appbar fixed w-full top-0 start-0 z-[1000] px-4">
  <div class="drag flex items-center gap-2 flex-1 h-full">
    <div class="w-6 h-6 rounded-lg bg-primary/20 flex items-center justify-center border border-primary/20">
       <img src="favicon.png" alt="MS" class="h-3.5 w-3.5 opacity-80" />
    </div>
    <span class="text-[11px] font-bold tracking-widest text-on-surface/60 uppercase">Media Stream Player</span>
  </div>

  <div class="flex justify-end window-controls no-drag h-full">
    <button
      type="button"
      class="no-drag titlebar-button inline-flex items-center justify-center hover:bg-white/5 transition-colors"
      onclick={() => appWindow.minimize()}
      title="Minimize"
    >
      <svg width="10" height="1" viewBox="0 0 10 1" fill="none" xmlns="http://www.w3.org/2000/svg">
        <rect width="10" height="1" fill="currentColor" fill-opacity="0.6"/>
      </svg>
    </button>

    <button
      type="button"
      class="no-drag titlebar-button inline-flex items-center justify-center hover:bg-white/5 transition-colors"
      onclick={() => appWindow.toggleMaximize()}
      title={isMaximized ? "Restore" : "Maximize"}
    >
      <svg width="10" height="10" viewBox="0 0 10 10" fill="none" xmlns="http://www.w3.org/2000/svg">
        <rect x="0.5" y="0.5" width="9" height="9" stroke="currentColor" stroke-opacity="0.6"/>
      </svg>
    </button>

    <button
      type="button"
      class="no-drag titlebar-button inline-flex items-center justify-center hover:bg-red-500/80 hover:text-white transition-all group"
      onclick={() => appWindow.close()}
      title="Close"
    >
      <svg width="10" height="10" viewBox="0 0 10 10" fill="none" xmlns="http://www.w3.org/2000/svg">
        <path d="M1 1L9 9M9 1L1 9" stroke="currentColor" stroke-opacity="0.6" stroke-width="1.2" class="group-hover:stroke-opacity-100"/>
      </svg>
    </button>
  </div>
</div>

<style>
  .drag { -webkit-app-region: drag; }
  .no-drag { -webkit-app-region: no-drag; }
  .titlebar-button {
    width: 44px;
    height: 100%;
  }
</style>
