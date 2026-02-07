<script>
  import { onMount } from "svelte";
  import { page } from "$app/stores";
  import { Button, Icon, Snackbar } from "m3-svelte";
  import playCircleIcon from "@iconify-icons/mdi/play-circle";
  import arrowLeftIcon from "@iconify-icons/mdi/arrow-left";
  import closeIcon from "@iconify-icons/mdi/close";
  import tuneIcon from "@iconify-icons/mdi/tune-variant";
  import shieldIcon from "@iconify-icons/mdi/shield-key-outline";
  import webIcon from "@iconify-icons/mdi/web";
  import codeIcon from "@iconify-icons/mdi/xml";
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
    streamUrl: (value) => {
      if (!value.toString().trim()) return "Stream URL is required";
      try { new URL(value); } catch (e) { return "Must be a valid URL"; }
      return null;
    },
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
    if (!value) return "Just now";
    const parsed = new Date(value);
    return Number.isNaN(parsed.getTime()) ? "Just now" : parsed.toLocaleString();
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

  $: isModalOpen ? document.body.classList.add("modal-open") : document.body.classList.remove("modal-open");
</script>

<div class="space-y-10 animate-fade-in">
  <header class="flex flex-col md:flex-row md:items-center justify-between gap-6">
    <div class="space-y-2">
      <div class="hero-badge">Profile Details</div>
      <h1 class="hero-title">{savedStream?.name ?? "Saved Stream"}</h1>
      <p class="text-on-surface-variant text-sm max-w-lg leading-relaxed">
        Comprehensive configuration breakdown for this saved profile. Review manifest, headers, and DRM settings.
      </p>
      <a class="nav-pill !inline-flex mt-2" href="/library">
        <Icon icon={arrowLeftIcon} size={0.9} />
        <span>Return to Library</span>
      </a>
    </div>
    {#if savedStream}
      <div class="flex items-center gap-3">
        <Button onclick={() => playStreamFromData(savedStream.stream)} class="!rounded-xl h-12 bg-primary text-white shadow-xl shadow-primary/20">
          <Icon icon={playCircleIcon} size={1} />
          <span class="ml-2 font-bold">Launch Stream</span>
        </Button>
      </div>
    {/if}
  </header>

  {#if !isLoaded}
    <div class="glass-card p-12 text-center">
      <p class="text-on-surface-variant animate-pulse">Loading profile configuration...</p>
    </div>
  {:else if !savedStream}
    <div class="glass-card p-12 text-center space-y-6">
      <div class="w-16 h-16 rounded-full bg-red-500/10 flex items-center justify-center mx-auto">
        <Icon icon={closeIcon} class="text-red-400 text-3xl" />
      </div>
      <div class="space-y-2">
        <h3 class="text-xl font-bold text-on-surface">Profile Not Found</h3>
        <p class="text-on-surface-variant">The requested profile might have been deleted or moved.</p>
      </div>
      <a class="nav-pill !inline-flex" href="/library">
        <Icon icon={arrowLeftIcon} size={0.9} />
        <span>Back to Library</span>
      </a>
    </div>
  {:else}
    <div class="grid grid-cols-1 lg:grid-cols-2 gap-8">
      <!-- Basics -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-2 border-b border-white/5">
          <Icon icon={tuneIcon} class="text-primary text-xl" />
          <h3 class="text-lg font-bold text-on-surface">General Info</h3>
        </div>
        <div class="grid grid-cols-1 gap-4">
          <div class="space-y-1">
            <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">Stream URL</span>
            <p class="text-sm font-mono break-all p-3 bg-black/20 rounded-lg border border-white/5">{getValue(savedStream.stream.streamUrl)}</p>
          </div>
          <div class="grid grid-cols-2 gap-4">
            <div class="space-y-1">
              <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">Manifest Type</span>
              <p class="text-sm font-semibold text-primary">{getValue(savedStream.stream.streamType)}</p>
            </div>
            <div class="space-y-1">
              <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">DRM Scheme</span>
              <p class="text-sm font-semibold text-secondary">{getValue(savedStream.stream.drmScheme)}</p>
            </div>
          </div>
          <div class="space-y-1">
            <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">Last Modified</span>
            <p class="text-sm text-on-surface-variant">{formatTimestamp(savedStream.updatedAt ?? savedStream.createdAt)}</p>
          </div>
        </div>
      </section>

      <!-- Network -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-2 border-b border-white/5">
          <Icon icon={webIcon} class="text-secondary text-xl" />
          <h3 class="text-lg font-bold text-on-surface">Network & Auth</h3>
        </div>
        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <div class="space-y-1">
            <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">Origin</span>
            <p class="text-sm text-on-surface truncate">{getValue(savedStream.stream.origin)}</p>
          </div>
          <div class="space-y-1">
            <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">Referer</span>
            <p class="text-sm text-on-surface truncate">{getValue(savedStream.stream.referer)}</p>
          </div>
          <div class="md:col-span-2 space-y-1">
            <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">User-Agent</span>
            <p class="text-sm text-on-surface truncate">{getValue(savedStream.stream.userAgent)}</p>
          </div>
          <div class="md:col-span-2 space-y-1">
             <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">Custom Headers</span>
             <pre class="text-[11px] p-4 bg-black/30 rounded-xl border border-white/5 overflow-x-auto font-mono text-secondary/80">{getValue(savedStream.stream.requestHeaders)}</pre>
          </div>
        </div>
      </section>

      <!-- DRM -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-2 border-b border-white/5">
          <Icon icon={shieldIcon} class="text-tertiary text-xl" />
          <h3 class="text-lg font-bold text-on-surface">DRM Specification</h3>
        </div>
        <div class="space-y-6">
           <div class="grid grid-cols-1 gap-4">
              <div class="space-y-1">
                <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">ClearKey Value</span>
                <p class="text-sm font-mono text-tertiary">{getValue(savedStream.stream.clearKey)}</p>
              </div>
              <div class="space-y-1">
                <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">License Server</span>
                <p class="text-sm font-mono break-all">{getValue(savedStream.stream.licenseUrl)}</p>
              </div>
           </div>
           <div class="space-y-3">
              <div class="space-y-1">
                <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">License Headers</span>
                <pre class="text-[11px] p-3 bg-black/20 rounded-lg border border-white/5 overflow-x-auto font-mono">{getValue(savedStream.stream.licenseHeaders)}</pre>
              </div>
              <div class="space-y-1">
                <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">Certificate Headers</span>
                <pre class="text-[11px] p-3 bg-black/20 rounded-lg border border-white/5 overflow-x-auto font-mono">{getValue(savedStream.stream.certificateHeaders)}</pre>
              </div>
           </div>
        </div>
      </section>

      <!-- Advanced -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-2 border-b border-white/5">
          <Icon icon={codeIcon} class="text-on-surface-variant text-xl" />
          <h3 class="text-lg font-bold text-on-surface">Engine Configuration</h3>
        </div>
        <div class="space-y-2">
           <span class="text-[10px] uppercase tracking-widest text-on-surface/40 font-bold">Shaka Config Overlay</span>
           <pre class="text-[11px] p-5 bg-black/40 rounded-2xl border border-white/5 overflow-x-auto font-mono text-primary/70 leading-relaxed">{getValue(savedStream.stream.shakaConfig)}</pre>
        </div>
      </section>
    </div>
  {/if}
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
