<script>
  import "../app.css";
  import AppBar from "$lib/components/AppBar.svelte";
  import { onMount, onDestroy } from "svelte";
  import { invoke } from "@tauri-apps/api/core";
  import { getCurrentWindow } from "@tauri-apps/api/window";

  import { Menu, PredefinedMenuItem } from "@tauri-apps/api/menu";

  const fullscreenchanged = async () => {
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
      // Prevent F5 or Ctrl+R (Windows/Linux) and Command+R (Mac) from refreshing the page
      if (
        event.key === "F5" ||
        (event.ctrlKey && event.key === "r") ||
        (event.metaKey && event.key === "r")
      ) {
        event.preventDefault();
      }
    });
  };

  // Disable right click, refresh etc on production
  if (!import.meta.env.DEV) {
    disableUserInteraction();
  }

  onMount(async () => {
    document.addEventListener("fullscreenchange", fullscreenchanged);

    const copy = await PredefinedMenuItem.new({
      text: "Copy",
      item: "Copy",
    });

    const cut = await PredefinedMenuItem.new({
      text: "Cut",
      item: "Cut",
    });

    const paste = await PredefinedMenuItem.new({
      text: "Paste",
      item: "Paste",
    });

    const select_all = await PredefinedMenuItem.new({
      text: "Select All",
      item: "SelectAll",
    });

    const menu = await Menu.new({
      items: [copy, cut, paste, select_all],
    });

    document.addEventListener("contextmenu", async (event) => {
      if (import.meta.env.DEV) {
        return;
      }
      event.preventDefault();
      const target = /** @type {HTMLElement} */ (event.target);

      if (["TEXTAREA", "INPUT"].includes(target.tagName)) {
        await menu.popup();
      }
    });

    requestIdleCallback(() => {
      // Finally show the window
      invoke("show_main_window");
    });
  });

  onDestroy(async () => {
    // Don't really need to, but we are good citizen
    document.removeEventListener("fullscreenchange", fullscreenchanged);
  });
</script>

<AppBar />

<div
  class="px-5 pt-6 flex flex-col h-full relative"
  id="content"
  style="padding-bottom:8rem"
>
  <slot />
</div>
<div class="modal-backdrop"></div>
