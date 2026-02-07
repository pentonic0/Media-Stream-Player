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

<div class="flex items-center h-[var(--navbar-height)] appbar fixed w-full top-0 start-0 z-[1000] px-4 bg-surface/40 backdrop-blur-xl border-b border-white/[0.03]">
  <div class="drag flex items-center gap-2 flex-1 h-full">
    <div class="w-6 h-6 rounded-lg bg-gradient-to-br from-primary/20 to-primary/5 flex items-center justify-center border border-primary/20 shadow-sm shadow-primary/10">
       <img src="favicon.png" alt="MS" class="h-3 w-3 opacity-90" />
    </div>
    <div class="flex items-center gap-1.5">
      <span class="text-[10px] font-black tracking-[0.2em] text-white/90 uppercase">Media Stream</span>
      <span class="text-[10px] font-medium tracking-[0.2em] text-primary uppercase">Player</span>
    </div>
  </div>

  <div class="flex justify-end window-controls no-drag h-full">
    <button
      type="button"
      class="no-drag titlebar-button inline-flex items-center justify-center hover:bg-white/[0.05] transition-all duration-200 group"
      onclick={() => appWindow.minimize()}
      title="Minimize"
    >
      <svg width="12" height="1" viewBox="0 0 12 1" fill="none" xmlns="http://www.w3.org/2000/svg">
        <rect width="12" height="1" fill="white" fill-opacity="0.4" class="group-hover:fill-opacity-100 transition-opacity"/>
      </svg>
    </button>

    <button
      type="button"
      class="no-drag titlebar-button inline-flex items-center justify-center hover:bg-white/[0.05] transition-all duration-200 group"
      onclick={() => appWindow.toggleMaximize()}
      title={isMaximized ? "Restore" : "Maximize"}
    >
      <svg width="10" height="10" viewBox="0 0 10 10" fill="none" xmlns="http://www.w3.org/2000/svg">
        <rect x="0.75" y="0.75" width="8.5" height="8.5" stroke="white" stroke-opacity="0.4" stroke-width="1.2" class="group-hover:stroke-opacity-100 transition-opacity"/>
      </svg>
    </button>

    <button
      type="button"
      class="no-drag titlebar-button inline-flex items-center justify-center hover:bg-red-500/90 transition-all duration-200 group"
      onclick={() => appWindow.close()}
      title="Close"
    >
      <svg width="10" height="10" viewBox="0 0 10 10" fill="none" xmlns="http://www.w3.org/2000/svg">
        <path d="M1.5 1.5L8.5 8.5M8.5 1.5L1.5 8.5" stroke="white" stroke-opacity="0.4" stroke-width="1.5" class="group-hover:stroke-opacity-100 transition-opacity"/>
      </svg>
    </button>
  </div>
</div>

<style>
  .drag { -webkit-app-region: drag; }
  .no-drag { -webkit-app-region: no-drag; }
  .titlebar-button {
    width: 46px;
    height: 100%;
  }
</style>
