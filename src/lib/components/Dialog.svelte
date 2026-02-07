<script>
  import { Icon } from "m3-svelte";

  /** @type {{
    icon?: any,
    headline?: string,
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
      dialog.show();
      document.body.classList.add("modal-open");
    } else {
      dialog.close();
      document.body.classList.remove("modal-open");
    }
  });
</script>

<dialog
  class="m3-container glass-panel"
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
  {#if icon}
    <Icon {icon} size={24} />
  {/if}
  {#if headline}
    <p class="headline m3-font-headline-small" class:center={icon}>{headline}</p>
  {/if}
  <div class="content m3-font-body-medium">
    {@render children()}
  </div>
  {#if buttons}
    <form method="dialog" class="buttons">
      {@render buttons()}
    </form>
  {/if}
</dialog>

<style>
  :root {
    --m3-dialog-shape: 2rem;
  }
  dialog {
    display: flex;
    flex-direction: column;
    background: rgba(var(--m3-scheme-surface-container-high) / 0.8) !important;
    backdrop-filter: blur(24px);
    border: 1px solid rgba(255, 255, 255, 0.1) !important;
    border-radius: var(--m3-dialog-shape);
    min-width: 17.5rem;
    max-width: 90vw;
    padding: 0;
    overflow: hidden;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.5);
    > :global(svg) {
      color: rgb(var(--m3-scheme-secondary));
      flex-shrink: 0;
      align-self: center;
      margin: 1.5rem 1.5rem 0;
    }
  }
  .headline {
    color: rgb(var(--m3-scheme-on-surface));
    margin: 1.5rem 1.5rem 1rem;
    font-weight: 800;
    letter-spacing: -0.025em;
  }
  .headline.center {
    text-align: center;
  }
  .content {
    color: rgb(var(--m3-scheme-on-surface-variant));
    flex: 1;
  }
  .buttons {
    display: flex;
    justify-content: flex-end;
    gap: 0.5rem;
    padding: 1rem 1.5rem 1.5rem;
  }

  dialog {
    position: fixed;
    inset: 0;
    opacity: 0;
    visibility: hidden;
    pointer-events: none;
    transition:
      opacity 0.3s cubic-bezier(0.4, 0, 0.2, 1),
      visibility 0.3s cubic-bezier(0.4, 0, 0.2, 1),
      transform 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
    transform: scale(0.9) translateY(20px);
  }
  dialog[open] {
    opacity: 1;
    visibility: visible;
    pointer-events: auto;
    transform: scale(1) translateY(0);
  }

  dialog::backdrop {
    background-color: rgba(0, 0, 0, 0.6);
    backdrop-filter: blur(4px);
    transition: opacity 0.3s ease;
  }

  @media print, (forced-colors: active) {
    dialog {
      outline: solid 0.125rem canvastext;
    }
  }
</style>
