<script>
  import { onMount } from "svelte";
  import Icon from "@iconify/svelte";
  import playCircleIcon from "@iconify-icons/mdi/play-circle";
  import bookmarkIcon from "@iconify-icons/mdi/bookmark";
  import historyIcon from "@iconify-icons/mdi/history";
  import trashIcon from "@iconify-icons/mdi/trash-can-outline";
  import infoIcon from "@iconify-icons/mdi/information-outline";
  import closeIcon from "@iconify-icons/mdi/close";

  import VideoPlayer from "$lib/components/VideoPlayer.svelte";
  import Dialog from "$lib/components/Dialog.svelte";
  import { showToast } from "$lib/toast.svelte.js";

  let isModalOpen = $state(false);

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
  let formData = $state({ ...defaultFormData });

  let streamHistory = $state([]);
  let savedStreams = $state([]);

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
      showToast(urlError);
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
    showToast("Profile removed from library.");
  };

  const clearHistory = () => {
    streamHistory = [];
    persistList(STORAGE_KEYS.history, streamHistory);
    showToast("Playback history cleared.");
  };

  $effect(() => {
    if (isModalOpen) {
      document.body.classList.add("modal-open");
    } else {
      document.body.classList.remove("modal-open");
    }
  });
</script>

<div class="space-y-10 animate-fade-in">
  <header class="flex flex-col md:flex-row md:items-center justify-between gap-6">
    <div class="space-y-2">
      <div class="hero-badge bg-primary/10 text-primary border border-primary/20 px-3 py-1 rounded-full text-[10px] font-bold uppercase tracking-wider inline-block">Library & History</div>
      <h1 class="text-4xl font-black text-white tracking-tight">Your Collection</h1>
      <p class="text-on-surface-variant text-sm max-w-lg leading-relaxed">
        Access your saved stream profiles and playback history. Manage your collection and resume sessions instantly.
      </p>
    </div>
    <div class="flex items-center gap-4">
      <div class="glass-panel rounded-2xl px-5 py-3 border border-white/5 flex flex-col items-center min-w-[100px] shadow-xl">
        <span class="text-[9px] uppercase tracking-[0.2em] text-on-surface-variant font-bold mb-1">Saved</span>
        <span class="text-2xl font-black text-primary leading-none">{savedStreams.length}</span>
      </div>
      <div class="glass-panel rounded-2xl px-5 py-3 border border-white/5 flex flex-col items-center min-w-[100px] shadow-xl">
        <span class="text-[9px] uppercase tracking-[0.2em] text-on-surface-variant font-bold mb-1">History</span>
        <span class="text-2xl font-black text-secondary leading-none">{streamHistory.length}</span>
      </div>
    </div>
  </header>

  <div class="grid grid-cols-1 xl:grid-cols-2 gap-10">
    <!-- Saved Streams Section -->
    <section class="space-y-6">
      <div class="flex items-center justify-between px-2">
        <div class="flex items-center gap-3">
          <div class="w-8 h-8 rounded-lg bg-primary/10 flex items-center justify-center border border-primary/10">
            <Icon icon={bookmarkIcon} class="text-primary text-lg" />
          </div>
          <h2 class="text-xl font-bold text-white tracking-tight">Saved Profiles</h2>
        </div>
      </div>

      {#if savedStreams.length}
        <div class="grid grid-cols-1 gap-4">
          {#each savedStreams as item}
            <div class="glass-card p-6 flex flex-col md:flex-row md:items-center justify-between gap-6 group hover:border-primary/30 transition-all">
              <div class="space-y-1.5 flex-1 min-w-0">
                <div class="flex items-center gap-2">
                  <h3 class="font-bold text-white text-lg truncate">{item.name}</h3>
                  <span class="px-2 py-0.5 rounded-md bg-white/5 border border-white/5 text-on-surface-variant text-[9px] font-bold uppercase tracking-widest">
                    {item.stream.streamType === 'auto' ? 'Auto' : (item.stream.streamType.includes('dash') ? 'DASH' : 'HLS')}
                  </span>
                </div>
                <p class="text-[11px] text-on-surface-variant truncate font-mono opacity-60 leading-none">{item.stream.streamUrl}</p>
                <div class="flex items-center gap-4 pt-2">
                   <span class="text-[10px] uppercase tracking-tighter text-on-surface/40 font-bold">
                    Added {formatTimestamp(item.updatedAt ?? item.createdAt)}
                   </span>
                </div>
              </div>
              <div class="flex items-center gap-2">
                <button
                  onclick={() => playStreamFromData(item.stream)}
                  class="w-11 h-11 rounded-xl bg-primary text-white hover:scale-105 active:scale-95 transition-all shadow-lg shadow-primary/20 flex items-center justify-center"
                  title="Play Stream"
                >
                  <Icon icon={playCircleIcon} class="text-xl" />
                </button>
                <a
                  href={`/saved/${item.id}`}
                  class="w-11 h-11 rounded-xl bg-white/5 flex items-center justify-center text-on-surface-variant hover:text-white hover:bg-white/10 transition-all border border-white/10"
                  title="Details"
                >
                  <Icon icon={infoIcon} class="text-lg" />
                </a>
                <button
                  onclick={() => deleteSavedStream(item.id)}
                  class="w-11 h-11 rounded-xl bg-white/5 text-on-surface-variant hover:text-red-400 hover:bg-red-500/10 hover:border-red-500/20 transition-all border border-white/10 flex items-center justify-center"
                  title="Delete"
                >
                  <Icon icon={trashIcon} class="text-lg" />
                </button>
              </div>
            </div>
          {/each}
        </div>
      {:else}
        <div class="glass-card p-12 text-center border-dashed border-white/10 opacity-60">
          <div class="w-16 h-16 rounded-3xl bg-white/5 flex items-center justify-center mx-auto mb-4 border border-white/5">
            <Icon icon={bookmarkIcon} class="text-on-surface-variant/30 text-3xl" />
          </div>
          <p class="text-sm text-on-surface-variant font-medium">Your library is empty.</p>
          <p class="text-[11px] text-on-surface-variant/60 mt-1 uppercase tracking-widest font-bold">Save profiles in the workspace</p>
        </div>
      {/if}
    </section>

    <!-- History Section -->
    <section class="space-y-6">
      <div class="flex items-center justify-between px-2">
        <div class="flex items-center gap-3">
          <div class="w-8 h-8 rounded-lg bg-secondary/10 flex items-center justify-center border border-secondary/10">
            <Icon icon={historyIcon} class="text-secondary text-lg" />
          </div>
          <h2 class="text-xl font-bold text-white tracking-tight">Playback History</h2>
        </div>
        {#if streamHistory.length}
          <button
            onclick={clearHistory}
            class="text-[10px] font-black text-red-400 hover:text-red-300 transition-colors uppercase tracking-[0.2em] bg-red-500/10 px-3 py-1.5 rounded-lg border border-red-500/10"
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
              class="w-full text-left glass-card p-5 hover:border-secondary/30 group relative transition-all"
              onclick={() => playStreamFromData(item.stream)}
            >
              <div class="flex items-center justify-between gap-4">
                <div class="flex-1 min-w-0 space-y-1.5">
                  <p class="text-sm font-bold text-white truncate pr-4 leading-none">{item.stream.streamUrl}</p>
                  <p class="text-[10px] text-on-surface-variant font-medium uppercase tracking-tighter">Last played {formatTimestamp(item.lastPlayed)}</p>
                </div>
                <div class="w-10 h-10 rounded-xl bg-secondary/10 flex items-center justify-center text-secondary group-hover:bg-secondary group-hover:text-white transition-all shadow-lg shadow-secondary/10 border border-secondary/10 scale-90 group-hover:scale-100">
                  <Icon icon={playCircleIcon} class="text-xl" />
                </div>
              </div>
            </button>
          {/each}
        </div>
      {:else}
        <div class="glass-card p-12 text-center border-dashed border-white/10 opacity-60">
          <div class="w-16 h-16 rounded-3xl bg-white/5 flex items-center justify-center mx-auto mb-4 border border-white/5">
            <Icon icon={historyIcon} class="text-on-surface-variant/30 text-3xl" />
          </div>
          <p class="text-sm text-on-surface-variant font-medium">No recent activity.</p>
          <p class="text-[11px] text-on-surface-variant/60 mt-1 uppercase tracking-widest font-bold">Start a session to build history</p>
        </div>
      {/if}
    </section>
  </div>
</div>

<div class="player-modal">
  <Dialog
    headline="Stream Player"
    class="video-player-dialog"
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
        <Icon icon={closeIcon} />
      </button>

      {#if isModalOpen}
        <div class="video-player-frame">
          <VideoPlayer stream={formData} />
        </div>
      {/if}
    {/snippet}
    {#snippet buttons()}{/snippet}
  </Dialog>
</div>

