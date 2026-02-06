<script>
  import { onMount } from "svelte";
  import { page } from "$app/stores";
  import { Button, Icon, Snackbar } from "m3-svelte";
  import playCircleIcon from "@iconify-icons/mdi/play-circle";
  import arrowLeftIcon from "@iconify-icons/mdi/arrow-left";
  import closeIcon from "@iconify-icons/mdi/close";
  import VideoPlayer from "$lib/components/VideoPlayer.svelte";
  import Dialog from "$lib/components/Dialog.svelte";

  let isModalOpen;
  let savedStreams = [];
  let savedStream = null;
  let isLoaded = false;

  /**
   * @type {Snackbar}
   */
  let snackbar;

  const defaultFormData = {
    streamUrl: "",
    streamType: "auto",
    cookie: "",
    referer: "",
    origin: "",
    userAgent: "",
    drmScheme: "none",
    clearKey: "",
    licenseUrl: "",
    licenseHeaders: "",
    certificateUrl: "",
    certificateHeaders: "",
    requestHeaders: "",
    shakaConfig: "",
  };

  /**
   * @type {StreamFormData}
   */
  let formData = { ...defaultFormData };

  const STORAGE_KEYS = {
    saved: "msp_saved_streams",
  };

  const rules = {
    /**
     *
     * @param value {String}
     */
    streamUrl: (value) => {
      if (!value.toString().trim()) return "Stream URL is required";

      try {
        new URL(value);
      } catch (e) {
        return "Must be a valid URL";
      }

      return null;
    },
  };

  const loadStoredList = (key) => {
    try {
      const stored = localStorage.getItem(key);
      if (!stored) {
        return [];
      }
      const parsed = JSON.parse(stored);
      return Array.isArray(parsed) ? parsed : [];
    } catch (error) {
      console.warn("Failed to parse stored list", error);
      return [];
    }
  };

  const applyStreamToForm = (stream) => {
    formData = { ...defaultFormData, ...stream };
  };

  const playStreamFromData = (stream) => {
    const urlError = rules.streamUrl(stream.streamUrl);
    if (urlError) {
      snackbar?.show({ message: urlError });
      return;
    }
    applyStreamToForm(stream);
    isModalOpen = true;
  };

  const formatTimestamp = (value) => {
    if (!value) {
      return "Just now";
    }
    const parsed = new Date(value);
    return Number.isNaN(parsed.getTime())
      ? "Just now"
      : parsed.toLocaleString();
  };

  const getValue = (value) => {
    if (value === null || value === undefined || value === "") {
      return "—";
    }
    return value;
  };

  onMount(() => {
    savedStreams = loadStoredList(STORAGE_KEYS.saved);
    const id = $page.params.id;
    savedStream = savedStreams.find((item) => item.id === id) ?? null;
    isLoaded = true;
  });

  $: isModalOpen
    ? document.body.classList.add("modal-open")
    : document.body.classList.remove("modal-open");
</script>

<div class="app-shell space-y-8">
  <header class="panel-card app-hero">
    <div class="flex flex-wrap items-start justify-between gap-6">
      <div class="space-y-3">
        <p class="text-xs uppercase tracking-[0.25em] text-on-surface">
          Saved stream info
        </p>
        <h2 class="text-3xl font-semibold text-on-body">
          {savedStream?.name ?? "Saved Stream"}
        </h2>
        <p class="text-sm text-on-surface max-w-xl">
          Review every configuration detail for this stream, including headers,
          DRM, and advanced Shaka settings.
        </p>
        <a class="nav-pill nav-pill--ghost" href="/library">
          <Icon icon={arrowLeftIcon} size={0.9} />
          <span>Back to library</span>
        </a>
      </div>
      {#if savedStream}
        <div class="flex flex-wrap gap-2">
          <Button onclick={() => playStreamFromData(savedStream.stream)}>
            <Icon icon={playCircleIcon} size={0.9} />
            <span class="ml-1">Play stream</span>
          </Button>
        </div>
      {/if}
    </div>
  </header>

  {#if !isLoaded}
    <div class="panel-card panel-card--glass">
      <p class="text-sm text-on-surface">Loading saved stream...</p>
    </div>
  {:else if !savedStream}
    <div class="panel-card panel-card--glass space-y-2">
      <p class="text-sm text-on-surface">
        We could not find that saved stream.
      </p>
      <a class="nav-pill nav-pill--ghost" href="/library">
        <Icon icon={arrowLeftIcon} size={0.9} />
        <span>Return to library</span>
      </a>
    </div>
  {:else}
    <div class="grid gap-6 xl:grid-cols-[minmax(0,1.1fr)_minmax(0,1fr)]">
      <section class="panel-card panel-card--glass space-y-4">
        <h3 class="text-base font-semibold text-on-body">Stream basics</h3>
        <div class="info-grid">
          <div class="info-item">
            <p class="info-label">Stream URL</p>
            <p class="info-value break-all">{getValue(savedStream.stream.streamUrl)}</p>
          </div>
          <div class="info-item">
            <p class="info-label">Type</p>
            <p class="info-value">{getValue(savedStream.stream.streamType)}</p>
          </div>
          <div class="info-item">
            <p class="info-label">DRM scheme</p>
            <p class="info-value">{getValue(savedStream.stream.drmScheme)}</p>
          </div>
          <div class="info-item">
            <p class="info-label">Last updated</p>
            <p class="info-value">
              {formatTimestamp(savedStream.updatedAt ?? savedStream.createdAt)}
            </p>
          </div>
        </div>
      </section>

      <section class="panel-card panel-card--glass space-y-4">
        <h3 class="text-base font-semibold text-on-body">Request headers</h3>
        <div class="info-grid">
          <div class="info-item">
            <p class="info-label">Cookie</p>
            <p class="info-value break-all">{getValue(savedStream.stream.cookie)}</p>
          </div>
          <div class="info-item">
            <p class="info-label">Origin</p>
            <p class="info-value break-all">{getValue(savedStream.stream.origin)}</p>
          </div>
          <div class="info-item">
            <p class="info-label">Referer</p>
            <p class="info-value break-all">{getValue(savedStream.stream.referer)}</p>
          </div>
          <div class="info-item">
            <p class="info-label">User-Agent</p>
            <p class="info-value break-all">{getValue(savedStream.stream.userAgent)}</p>
          </div>
        </div>
        <div class="info-block">
          <p class="info-label">Additional headers</p>
          <pre>{getValue(savedStream.stream.requestHeaders)}</pre>
        </div>
      </section>
    </div>

    <div class="grid gap-6 xl:grid-cols-[minmax(0,1.1fr)_minmax(0,1fr)]">
      <section class="panel-card panel-card--glass space-y-4">
        <h3 class="text-base font-semibold text-on-body">DRM details</h3>
        <div class="info-grid">
          <div class="info-item">
            <p class="info-label">ClearKey</p>
            <p class="info-value break-all">{getValue(savedStream.stream.clearKey)}</p>
          </div>
          <div class="info-item">
            <p class="info-label">License URL</p>
            <p class="info-value break-all">{getValue(savedStream.stream.licenseUrl)}</p>
          </div>
          <div class="info-item">
            <p class="info-label">Certificate URL</p>
            <p class="info-value break-all">{getValue(savedStream.stream.certificateUrl)}</p>
          </div>
        </div>
        <div class="info-block">
          <p class="info-label">License headers</p>
          <pre>{getValue(savedStream.stream.licenseHeaders)}</pre>
        </div>
        <div class="info-block">
          <p class="info-label">Certificate headers</p>
          <pre>{getValue(savedStream.stream.certificateHeaders)}</pre>
        </div>
      </section>

      <section class="panel-card panel-card--glass space-y-4">
        <h3 class="text-base font-semibold text-on-body">Advanced settings</h3>
        <div class="info-block">
          <p class="info-label">Shaka configuration</p>
          <pre>{getValue(savedStream.stream.shakaConfig)}</pre>
        </div>
      </section>
    </div>
  {/if}
</div>

<div class="player-modal">
  <Dialog
    headline="Player"
    bind:open={isModalOpen}
    closedby="closerequest"
    closeOnEsc={true}
    icon={false}
  >
    {#snippet children()}
      <Button
        style="position: absolute; top: 4px; right: 10px;"
        variant="outlined"
        onclick={() => (isModalOpen = false)}
        iconType="full"
      >
        <Icon icon={closeIcon} size={0.9} />
      </Button>

      {#if isModalOpen}
        <VideoPlayer stream={formData} />
      {/if}
    {/snippet}
    {#snippet buttons()}{/snippet}
  </Dialog>
</div>

<Snackbar class="shaka-snack holder" bind:this={snackbar} />
