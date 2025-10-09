<script>
  import "../app.css";
  import AppBar from "$lib/components/AppBar.svelte";
  import { onMount, onDestroy } from "svelte";
  import { invoke } from "@tauri-apps/api/core";
  import { getCurrentWindow } from "@tauri-apps/api/window";

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
    document.addEventListener("contextmenu", (event) => event.preventDefault());

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
