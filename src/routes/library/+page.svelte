<script>
  import { Button, Icon, Snackbar } from "m3-svelte";
  import { onMount } from "svelte";
  import playCircleIcon from "@iconify-icons/mdi/play-circle";
  import bookmarkIcon from "@iconify-icons/mdi/bookmark";
  import historyIcon from "@iconify-icons/mdi/history";
  import trashIcon from "@iconify-icons/mdi/trash-can-outline";
  import infoIcon from "@iconify-icons/mdi/information-outline";
  import arrowLeftIcon from "@iconify-icons/mdi/arrow-left";
  import closeIcon from "@iconify-icons/mdi/close";
  import VideoPlayer from "$lib/components/VideoPlayer.svelte";
  import Dialog from "$lib/components/Dialog.svelte";

  let isModalOpen;

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

  let streamHistory = [];
  let savedStreams = [];

  const STORAGE_KEYS = {
    history: "msp_stream_history",
    saved: "msp_saved_streams",
  };
  const HISTORY_LIMIT = 12;

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

  const createId = () =>
    crypto?.randomUUID
      ? crypto.randomUUID()
      : `${Date.now()}-${Math.random().toString(16).slice(2)}`;

  const getStreamPayload = (data) => {
    const payload = {};
    Object.keys(defaultFormData).forEach((key) => {
      payload[key] = data[key] ?? defaultFormData[key];
    });
    return payload;
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

  const persistList = (key, value) => {
    localStorage.setItem(key, JSON.stringify(value));
  };

  const updateHistory = (stream) => {
    const payload = getStreamPayload(stream);
    const signature = JSON.stringify(payload);
    const newEntry = {
      id: createId(),
      lastPlayed: new Date().toISOString(),
      signature,
      stream: payload,
    };
    streamHistory = [
      newEntry,
      ...streamHistory.filter((item) => item.signature !== signature),
    ].slice(0, HISTORY_LIMIT);
    persistList(STORAGE_KEYS.history, streamHistory);
  };

  const updateSavedStreams = (streams) => {
    savedStreams = streams;
    persistList(STORAGE_KEYS.saved, savedStreams);
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
    updateHistory(stream);
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

  onMount(() => {
    streamHistory = loadStoredList(STORAGE_KEYS.history);
    savedStreams = loadStoredList(STORAGE_KEYS.saved);
  });

  const deleteSavedStream = (itemId) => {
    updateSavedStreams(savedStreams.filter((item) => item.id !== itemId));
  };

  const clearHistory = () => {
    streamHistory = [];
    persistList(STORAGE_KEYS.history, streamHistory);
  };

  $: isModalOpen
    ? document.body.classList.add("modal-open")
    : document.body.classList.remove("modal-open");
</script>

<div class="app-shell space-y-8">
  <header class="panel-card app-hero">
    <div class="flex flex-wrap items-start justify-between gap-6">
      <div class="space-y-3">
        <p class="text-xs uppercase tracking-[0.25em] text-on-surface">
          Saved library
        </p>
        <h2 class="text-3xl font-semibold text-on-body">Stream Library</h2>
        <p class="text-sm text-on-surface max-w-xl">
          Manage saved streams, review playback history, and jump straight back
          into trusted setups.
        </p>
        <a class="nav-pill nav-pill--ghost" href="/">
          <Icon icon={arrowLeftIcon} size={0.9} />
          <span>Back to workspace</span>
        </a>
      </div>
      <div class="flex flex-wrap gap-3 items-center">
        <div class="stat-chip">
          <p class="text-xs text-on-surface">Saved</p>
          <p class="text-base font-semibold text-on-body">
            {savedStreams.length}
          </p>
        </div>
        <div class="stat-chip">
          <p class="text-xs text-on-surface">History</p>
          <p class="text-base font-semibold text-on-body">
            {streamHistory.length}
          </p>
        </div>
      </div>
    </div>
  </header>

  <div class="grid gap-6 xl:grid-cols-[minmax(0,1.2fr)_minmax(0,1fr)]">
    <section class="panel-card panel-card--glass space-y-4">
      <div class="flex items-center justify-between gap-3">
        <div class="flex items-center gap-2">
          <Icon icon={bookmarkIcon} size={1} />
          <h3 class="text-base font-semibold text-on-body">Saved Streams</h3>
        </div>
        {#if savedStreams.length}
          <span class="text-xs text-on-surface"
            >{savedStreams.length} total</span
          >
        {/if}
      </div>

      {#if savedStreams.length}
        <div class="space-y-3">
          {#each savedStreams as item}
            <div class="saved-item">
              <div class="flex items-start justify-between gap-3">
                <div>
                  <p class="text-sm font-semibold text-on-body">{item.name}</p>
                  <p class="text-xs text-on-surface break-all">
                    {item.stream.streamUrl}
                  </p>
                  <p class="text-xs text-on-surface mt-1">
                    Updated {formatTimestamp(item.updatedAt ?? item.createdAt)}
                  </p>
                </div>
                <div class="flex flex-col gap-2">
                  <Button
                    iconType="full"
                    title="Play saved stream"
                    onclick={() => playStreamFromData(item.stream)}
                  >
                    <Icon icon={playCircleIcon} size={0.9} />
                  </Button>
                  <a
                    class="icon-button"
                    href={`/saved/${item.id}`}
                    title="View saved stream details"
                  >
                    <Icon icon={infoIcon} size={0.9} />
                  </a>
                </div>
              </div>
              <div class="flex flex-wrap gap-2">
                <Button
                  variant="outlined"
                  onclick={() => deleteSavedStream(item.id)}
                >
                  <Icon icon={trashIcon} size={0.8} />
                  <span class="ml-1">Delete</span>
                </Button>
              </div>
            </div>
          {/each}
        </div>
      {:else}
        <p class="text-sm text-on-surface">
          Save stream setups in the workspace to see them here.
        </p>
      {/if}
    </section>

    <section class="panel-card panel-card--glass space-y-4">
      <div class="flex items-center justify-between gap-3">
        <div class="flex items-center gap-2">
          <Icon icon={historyIcon} size={1} />
          <h3 class="text-base font-semibold text-on-body">Stream History</h3>
        </div>
        {#if streamHistory.length}
          <Button variant="outlined" onclick={clearHistory}>Clear</Button>
        {/if}
      </div>

      {#if streamHistory.length}
        <div class="space-y-3">
          {#each streamHistory as item}
            <button
              type="button"
              class="history-item"
              on:click={() => playStreamFromData(item.stream)}
            >
              <div class="flex items-start justify-between gap-3">
                <div>
                  <p class="text-sm font-semibold text-on-body">
                    {item.stream.streamUrl}
                  </p>
                  <p class="text-xs text-on-surface mt-1">
                    Played {formatTimestamp(item.lastPlayed)}
                  </p>
                </div>
                <Icon icon={playCircleIcon} size={1} />
              </div>
            </button>
          {/each}
        </div>
      {:else}
        <p class="text-sm text-on-surface">
          Your recent streams will show up here for quick replay.
        </p>
      {/if}
    </section>
  </div>
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
