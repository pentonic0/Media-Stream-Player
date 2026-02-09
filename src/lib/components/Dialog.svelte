<script>
  import Icon from "@iconify/svelte";

  /** @type {{
    icon?: any,
    headline?: string,
    class?: string,
    buttons?: any,
    children: any,
    open: boolean,
    closedby?: "none" | "any" | "closerequest",
    closeOnEsc?: boolean,
    onEsc?: Function,
    closeOnClick?: boolean,
    onClick?: Function
  }} */
  let {
    icon,
    headline,
    class: className = "",
    buttons,
    children,
    open = $bindable(),
    closedby = "any",
    ...extra
  } = $props();

  let dialog = $state();
  $effect(() => {
    if (!dialog) return;
    if (open) {
      if (!dialog.open) {
        dialog.show();
      }
      document.body.classList.add("modal-open");
    } else {
      if (dialog.open) {
        dialog.close();
      }
      document.body.classList.remove("modal-open");
    }
  });
</script>

<dialog
  class={`glass-panel ${className}`}
  ontoggle={(e) => {
    open = e.newState == "open";
  }}
  oncancel={(e) => {
    if (e.target != e.currentTarget) return;
    if (extra.closeOnEsc && extra.onEsc) {
      extra.onEsc();
    }
  }}
  onclick={(e) => {
    if (e.target != e.currentTarget) return;
    if (extra.closeOnClick && extra.onClick) {
      extra.onClick();
    }
  }}
  bind:this={dialog}
  closedby={closedby}
  role="alertdialog"
  {...extra}
>
  <div class="flex flex-col h-full w-full">
    {#if icon}
      <div class="pt-6 px-6 flex justify-center">
        <Icon {icon} class="text-3xl text-primary" />
      </div>
    {/if}
    {#if headline}
      <div class="pt-6 px-8">
        <h2 class="text-xl font-black text-white tracking-tight {icon ? 'text-center' : ''}">{headline}</h2>
      </div>
    {/if}
    <div class="flex-1 overflow-auto">
      {@render children()}
    </div>
    {#if buttons}
      <form method="dialog" class="flex justify-end gap-3 p-6 border-t border-white/5">
        {@render buttons()}
      </form>
    {/if}
  </div>
</dialog>

<style>
  dialog {
    z-index: 5000;
    background: rgba(var(--m3-scheme-surface-container-high) / 0.8) !important;
    backdrop-filter: blur(40px);
    border: 1px solid rgba(255, 255, 255, 0.1) !important;
    border-radius: 2.5rem;
    min-width: 20rem;
    max-width: 95vw;
    max-height: 95vh;
    padding: 0;
    overflow: hidden;
    box-shadow: 0 50px 100px -20px rgba(0, 0, 0, 0.7);
  }

  dialog {
    position: fixed;
    top: 50%;
    left: 50%;
    margin: 0;
    opacity: 0;
    visibility: hidden;
    pointer-events: none;
    transition:
      opacity 0.4s cubic-bezier(0.4, 0, 0.2, 1),
      visibility 0.4s cubic-bezier(0.4, 0, 0.2, 1),
      transform 0.4s cubic-bezier(0.34, 1.56, 0.64, 1);
    transform: translate(-50%, -50%) scale(0.95);
  }
  dialog[open] {
    opacity: 1;
    visibility: visible;
    pointer-events: auto;
    transform: translate(-50%, -50%) scale(1);
  }

  dialog::backdrop {
    background-color: rgba(var(--m3-scheme-background) / 0.2);
    backdrop-filter: blur(10px);
    transition: opacity 0.4s ease;
  }

  dialog.video-player-dialog {
    width: min(94vw, 1420px);
    height: min(calc(88vh - var(--navbar-height)), 860px);
    max-width: 94vw;
    max-height: calc(92vh - var(--navbar-height));
    top: calc(50% + (var(--navbar-height) / 2));
    border-radius: 2rem;
  }
</style>
