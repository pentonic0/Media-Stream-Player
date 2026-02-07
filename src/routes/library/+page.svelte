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
    streamUrl: (value) => {
      if (!value.toString().trim()) return "Stream URL is required";
      try { new URL(value); } catch (e) { return "Must be a valid URL"; }
      return null;
    },
  };

  const createId = () => crypto?.randomUUID ? crypto.randomUUID() : `${Date.now()}-${Math.random().toString(16).slice(2)}`;

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
      if (!stored) return [];
      const parsed = JSON.parse(stored);
      return Array.isArray(parsed) ? parsed : [];
    } catch (error) {
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
    if (!value) return "Just now";
    const parsed = new Date(value);
    return Number.isNaN(parsed.getTime()) ? "Just now" : parsed.toLocaleString();
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

  $: isModalOpen ? document.body.classList.add("modal-open") : document.body.classList.remove("modal-open");
</script>

<div class="space-y-10 animate-fade-in">
  <header class="flex flex-col md:flex-row md:items-center justify-between gap-6">
    <div class="space-y-2">
      <div class="hero-badge">Library & History</div>
      <h1 class="hero-title">Your Collection</h1>
      <p class="text-on-surface-variant text-sm max-w-lg leading-relaxed">
        Access your saved stream profiles and playback history. Manage your collection and resume sessions instantly.
      </p>
    </div>
    <div class="flex items-center gap-4">
      <div class="stat-chip flex flex-col items-center min-w-[80px]">
        <span class="text-[10px] uppercase tracking-widest text-on-surface/40">Saved</span>
        <span class="text-xl font-bold text-primary">{savedStreams.length}</span>
      </div>
      <div class="stat-chip flex flex-col items-center min-w-[80px]">
        <span class="text-[10px] uppercase tracking-widest text-on-surface/40">History</span>
        <span class="text-xl font-bold text-secondary">{streamHistory.length}</span>
      </div>
    </div>
  </header>

  <div class="grid grid-cols-1 xl:grid-cols-2 gap-10">
    <!-- Saved Streams Section -->
    <section class="space-y-6">
      <div class="flex items-center justify-between px-2">
        <div class="flex items-center gap-3">
          <Icon icon={bookmarkIcon} class="text-primary text-xl" />
          <h2 class="text-xl font-bold text-on-surface">Saved Profiles</h2>
        </div>
      </div>

      {#if savedStreams.length}
        <div class="grid grid-cols-1 gap-4">
          {#each savedStreams as item}
            <div class="glass-card p-6 flex flex-col md:flex-row md:items-center justify-between gap-6 group">
              <div class="space-y-1 flex-1 min-w-0">
                <h3 class="font-bold text-on-surface text-lg truncate">{item.name}</h3>
                <p class="text-xs text-on-surface-variant truncate font-mono opacity-60">{item.stream.streamUrl}</p>
                <div class="flex items-center gap-4 pt-2">
                   <span class="text-[10px] uppercase tracking-tighter text-on-surface/40">
                    Added {formatTimestamp(item.updatedAt ?? item.createdAt)}
                   </span>
                   <span class="px-2 py-0.5 rounded bg-primary/10 text-primary text-[9px] font-bold uppercase">
                     {item.stream.streamType === 'auto' ? 'Auto' : (item.stream.streamType.includes('dash') ? 'DASH' : 'HLS')}
                   </span>
                </div>
              </div>
              <div class="flex items-center gap-2">
                <Button
                  iconType="full"
                  onclick={() => playStreamFromData(item.stream)}
                  class="!rounded-xl bg-primary/20 text-primary hover:bg-primary hover:text-white transition-all shadow-lg shadow-primary/10"
                >
                  <Icon icon={playCircleIcon} size={1} />
                </Button>
                <a
                  href={`/saved/${item.id}`}
                  class="w-10 h-10 rounded-xl bg-white/5 flex items-center justify-center text-on-surface-variant hover:text-on-surface hover:bg-white/10 transition-all border border-white/5"
                >
                  <Icon icon={infoIcon} size={0.9} />
                </a>
                <button
                  onclick={() => deleteSavedStream(item.id)}
                  class="w-10 h-10 rounded-xl bg-red-500/10 text-red-400 flex items-center justify-center hover:bg-red-500 hover:text-white transition-all border border-red-500/10"
                >
                  <Icon icon={trashIcon} size={0.9} />
                </button>
              </div>
            </div>
          {/each}
        </div>
      {:else}
        <div class="glass-card p-12 text-center border-dashed border-white/10">
          <div class="w-16 h-16 rounded-full bg-white/5 flex items-center justify-center mx-auto mb-4">
            <Icon icon={bookmarkIcon} class="text-on-surface/20 text-3xl" />
          </div>
          <p class="text-on-surface-variant italic">Your library is empty. Save profiles in the workspace.</p>
        </div>
      {/if}
    </section>

    <!-- History Section -->
    <section class="space-y-6">
      <div class="flex items-center justify-between px-2">
        <div class="flex items-center gap-3">
          <Icon icon={historyIcon} class="text-secondary text-xl" />
          <h2 class="text-xl font-bold text-on-surface">Playback History</h2>
        </div>
        {#if streamHistory.length}
          <button
            onclick={clearHistory}
            class="text-xs font-bold text-red-400 hover:text-red-300 transition-colors uppercase tracking-widest"
          >
            Clear All
          </button>
        {/if}
      </div>

      {#if streamHistory.length}
        <div class="space-y-3">
          {#each streamHistory as item}
            <button
              type="button"
              class="w-full text-left glass-card p-4 hover:border-secondary/30 group relative transition-all"
              onclick={() => playStreamFromData(item.stream)}
            >
              <div class="flex items-center justify-between gap-4">
                <div class="flex-1 min-w-0 space-y-1">
                  <p class="text-sm font-semibold text-on-surface truncate pr-4">{item.stream.streamUrl}</p>
                  <p class="text-[10px] text-on-surface-variant">Last played {formatTimestamp(item.lastPlayed)}</p>
                </div>
                <div class="w-8 h-8 rounded-lg bg-secondary/10 flex items-center justify-center text-secondary group-hover:bg-secondary group-hover:text-white transition-all shadow-lg shadow-secondary/10">
                  <Icon icon={playCircleIcon} size={0.8} />
                </div>
              </div>
            </button>
          {/each}
        </div>
      {:else}
        <div class="glass-card p-12 text-center border-dashed border-white/10">
          <div class="w-16 h-16 rounded-full bg-white/5 flex items-center justify-center mx-auto mb-4">
            <Icon icon={historyIcon} class="text-on-surface/20 text-3xl" />
          </div>
          <p class="text-on-surface-variant italic">No recent streams found.</p>
        </div>
      {/if}
    </section>
  </div>
</div>

<div class="player-modal">
  <Dialog
    headline="Stream Player"
    bind:open={isModalOpen}
    closedby="closerequest"
    closeOnEsc={true}
    icon={false}
  >
    {#snippet children()}
      <button
        class="absolute top-4 right-4 z-[1001] w-10 h-10 rounded-full bg-black/50 text-white flex items-center justify-center hover:bg-black/70 transition-all border border-white/10"
        onclick={() => (isModalOpen = false)}
      >
        <Icon icon={closeIcon} size={1} />
      </button>

      {#if isModalOpen}
        <div class="bg-black w-full h-full flex items-center justify-center">
          <VideoPlayer stream={formData} />
        </div>
      {/if}
    {/snippet}
    {#snippet buttons()}{/snippet}
  </Dialog>
</div>

<Snackbar class="shaka-snack holder" bind:this={snackbar} />
