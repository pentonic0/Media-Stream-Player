<script>
  import "../app.css";
  import AppBar from "$lib/components/AppBar.svelte";
  import Sidebar from "$lib/components/Sidebar.svelte";
  import ToastContainer from "$lib/components/ToastContainer.svelte";
  import { onMount, onDestroy } from "svelte";
  import { invoke } from "@tauri-apps/api/core";
  import { getCurrentWindow } from "@tauri-apps/api/window";
  import { Menu, PredefinedMenuItem } from "@tauri-apps/api/menu";

  const isTauri = typeof window !== 'undefined' && window.__TAURI_INTERNALS__;

  const fullscreenchanged = async () => {
    if (!isTauri) return;
    const appWindow = getCurrentWindow();
    if (document.fullscreenElement) {
      document.body.classList.add("is-fullscreen");
      await appWindow.setFullscreen(true);
      await appWindow.setShadow(false);
    } else {
      document.body.classList.remove("is-fullscreen");
      await appWindow.setFullscreen(false);
      await appWindow.setShadow(true);
    }
  };

  const disableUserInteraction = () => {
    document.addEventListener("keydown", function (event) {
      if (
        event.key === "F5" ||
        (event.ctrlKey && event.key === "r") ||
        (event.metaKey && event.key === "r")
      ) {
        event.preventDefault();
      }
    });
  };

  if (!import.meta.env.DEV) {
    disableUserInteraction();
  }

  onMount(async () => {
    document.addEventListener("fullscreenchange", fullscreenchanged);

    if (isTauri) {
      const menuItems = await Promise.all([
        PredefinedMenuItem.new({ text: "Copy", item: "Copy" }),
        PredefinedMenuItem.new({ text: "Cut", item: "Cut" }),
        PredefinedMenuItem.new({ text: "Paste", item: "Paste" }),
        PredefinedMenuItem.new({ text: "Select All", item: "SelectAll" }),
      ]);

      const menu = await Menu.new({ items: menuItems });

      document.addEventListener("contextmenu", async (event) => {
        if (import.meta.env.DEV) return;
        event.preventDefault();
        const target = /** @type {HTMLElement} */ (event.target);
        if (["TEXTAREA", "INPUT"].includes(target.tagName)) {
          await menu.popup();
        }
      });

      requestIdleCallback(() => {
        invoke("show_main_window");
      });
    }
  });

  onDestroy(async () => {
    document.removeEventListener("fullscreenchange", fullscreenchanged);
  });
</script>

<AppBar />

<div id="app-container" class="pt-[var(--navbar-height)]">
  <Sidebar />
  <main id="main-content">
    <div id="scroll-area">
      <slot />
    </div>
  </main>
  <ToastContainer />
</div>

<div class="modal-backdrop"></div>

<style>
  :global(.is-fullscreen) #app-container {
    padding-top: 0;
  }
  :global(.is-fullscreen) :global(.sidebar),
  :global(.is-fullscreen) :global(.appbar) {
    display: none;
  }
</style>
