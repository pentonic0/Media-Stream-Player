<script>
  import {
    TextField,
    Select,
    FAB,
    Button,
    Icon,
    Snackbar,
    TextFieldMultiline,
  } from "m3-svelte";
  import { onMount, tick } from "svelte";
  import JSON5 from "json5";
  import playCircleIcon from "@iconify-icons/mdi/play-circle";
  import bookmarkIcon from "@iconify-icons/mdi/bookmark";
  import contentSaveIcon from "@iconify-icons/mdi/content-save";
  import closeIcon from "@iconify-icons/mdi/close";
  import refreshIcon from "@iconify-icons/mdi/refresh";
  import libraryIcon from "@iconify-icons/mdi/library-shelves";
  import infoIcon from "@iconify-icons/mdi/information-outline";
  import tuneIcon from "@iconify-icons/mdi/tune-variant";
  import shieldIcon from "@iconify-icons/mdi/shield-key-outline";
  import webIcon from "@iconify-icons/mdi/web";
  import codeIcon from "@iconify-icons/mdi/xml";

  import VideoPlayer from "$lib/components/VideoPlayer.svelte";
  import Dialog from "$lib/components/Dialog.svelte";
  import parseCurl from "parse-curl";

  let isModalOpen;

  /**
   * @type {Snackbar}
   */
  let snackbar;

  let defaultFormData = {
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

  let streamTypes = [
    { value: "auto", text: "Auto" },
    { value: "application/vnd.apple.mpegurl", text: "HLS" },
    { value: "application/dash+xml", text: "DASH" },
  ];

  const drmSchemes = [
    { value: "none", text: "N/A" },
    { value: "clearkey_inline", text: "ClearKey (Inline)" },
    { value: "org.w3.clearkey", text: "ClearKey (Server)" },
    { value: "com.widevine.alpha", text: "Widevine" },
    { value: "com.microsoft.playready", text: "PlayReady" },
  ];

  let errors = {};
  let streamHistory = [];
  let savedStreams = [];
  let savedStreamName = "";

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
    shakaConfig: (value) => {
      if (!value.toString().trim()) return null;
      try { JSON5.parse(value); } catch (e) { return "Must be valid JSON/JSON5"; }
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
    errors = {};
    [
      "requestHeaders",
      "shakaConfig",
      "licenseHeaders",
      "certificateHeaders",
    ].forEach((id) => resetTextAreaHeight(id));
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

  const resetTextAreaHeight = (id) => {
    const textarea = document.getElementById(id);
    if (!textarea) return;
    setTimeout(() => {
      const spaceEvent = new InputEvent("input", { bubbles: true, cancelable: true, inputType: "insertText", data: "" });
      textarea.dispatchEvent(spaceEvent);
    }, 0);
  };

  const validateField = (event) => {
    const input = /** @type {HTMLInputElement | HTMLTextAreaElement} */ (event.target);
    const name = input.id;
    const value = formData[name];
    const rule = rules[name];
    if (rule) {
      const error = rule(value);
      if (error) {
        errors = { ...errors, [name]: error };
      } else {
        const { [name]: removed, ...rest } = errors;
        errors = rest;
      }
    }
  };

  const handlePaste = (event) => {
    const pastedText = event.clipboardData.getData("text").trim();
    if (pastedText.startsWith("curl")) {
      event.preventDefault();
      try {
        const parsedData = parseCurl(pastedText);
        if (!parsedData.url) throw new Error("Only cURL bash command is supported.");
        snackbar.show({ message: "cURL command detected. Autofilled parameters." });
        formData.streamUrl = parsedData.url;
        let headerText = "";
        for (const key in parsedData.header) {
          const headerName = key.trim().toLowerCase();
          const value = parsedData.header[key];
          if (["cookie", "set-cookie"].includes(headerName)) { formData.cookie = value; continue; }
          if (["origin", "referer"].includes(headerName)) { formData[headerName] = value; continue; }
          if (headerName === "user-agent") { formData.userAgent = value; continue; }
          headerText += headerName + " : " + value + "\n";
        }
        formData.requestHeaders = headerText;
        resetTextAreaHeight("requestHeaders");
        validateField(event);
      } catch (error) {
        snackbar.show({ message: "Invalid CURL command. " + error });
      }
      return;
    }

    const decodedPaste = decodeURI(pastedText);
    if (decodedPaste.includes("|")) {
      event.preventDefault();
      const [url, search] = decodedPaste.split("|");
      const nsPlayerURL = new URL("https://google.com?" + search.trim());
      formData.streamUrl = url;
      validateField(event);
      const drmScheme = nsPlayerURL.searchParams.get("drmScheme");
      const drmLicense = nsPlayerURL.searchParams.get("drmLicense");
      let autofilled = false;
      ["origin", "userAgent", "referer", "referrer", "cookie"].forEach((key) => {
        const value = nsPlayerURL.searchParams.get(key);
        if (value) {
          formData[key === "referrer" ? "referer" : key] = value;
          autofilled = true;
        }
      });
      if (drmScheme === "clearkey") {
        if (drmLicense.includes(":")) {
          formData.drmScheme = "clearkey_inline";
          formData.clearKey = drmLicense;
        } else {
          formData.drmScheme = "org.w3.clearkey";
          formData.licenseUrl = drmLicense;
        }
        autofilled = true;
      }
      snackbar.show({ message: "NS Player URL detected. " + (autofilled ? "Autofilled parameters." : "No supported paramters found.") });
      return;
    }
    validateField(event);
  };

  const handleSubmit = (event) => {
    if (event) event.preventDefault();
    for (const name in formData) {
      validateField({ target: { id: name, value: formData[name] } });
    }
    if (Object.keys(errors).length !== 0) {
      const firstKey = Object.keys(errors)[0];
      document.getElementById(firstKey)?.focus();
      return;
    }
    updateHistory(formData);
    isModalOpen = true;
  };

  const resetFormData = () => {
    formData = { ...defaultFormData };
    [
      "requestHeaders",
      "shakaConfig",
      "licenseHeaders",
      "certificateHeaders",
    ].forEach((id) => resetTextAreaHeight(id));
    errors = {};
  };

  const saveStream = () => {
    const name = savedStreamName.trim();
    if (!name.length) {
      snackbar?.show({ message: "Provide a name to save this stream." });
      return;
    }
    const urlError = rules.streamUrl(formData.streamUrl);
    if (urlError) {
      snackbar?.show({ message: urlError });
      return;
    }
    const payload = getStreamPayload(formData);
    updateSavedStreams([
      { id: createId(), name, createdAt: new Date().toISOString(), stream: payload },
      ...savedStreams,
    ]);
    snackbar?.show({ message: "Stream saved for later." });
    savedStreamName = "";
  };

  $: isModalOpen ? document.body.classList.add("modal-open") : document.body.classList.remove("modal-open");
</script>

<div class="space-y-10 animate-fade-in">
  <header class="flex flex-col md:flex-row md:items-center justify-between gap-6">
    <div class="space-y-2">
      <div class="hero-badge">Stream Workspace</div>
      <h1 class="hero-title">New Session</h1>
      <p class="text-on-surface-variant text-sm max-w-lg leading-relaxed">
        Configure your stream parameters below. Support for DASH, HLS, and various DRM schemes with advanced header overrides.
      </p>
    </div>
    <div class="flex items-center gap-3">
      <Button variant="outlined" onclick={resetFormData} class="!rounded-xl h-11">
        <Icon icon={refreshIcon} size={0.9} />
        <span class="ml-2">Clear Form</span>
      </Button>
      <Button onclick={handleSubmit} class="!rounded-xl h-11 bg-primary text-white shadow-lg shadow-primary/20">
        <Icon icon={playCircleIcon} size={1} />
        <span class="ml-2 font-bold">Launch Player</span>
      </Button>
    </div>
  </header>

  <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
    <div class="lg:col-span-2 space-y-8">
      <!-- Main Config -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-2 border-b border-white/5">
          <div class="w-8 h-8 rounded-lg bg-primary/10 flex items-center justify-center">
            <Icon icon={tuneIcon} class="text-primary text-lg" />
          </div>
          <h3 class="text-lg font-bold text-on-surface">Base Configuration</h3>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-4 gap-6">
          <div class="md:col-span-3 space-y-2">
            <TextField
              autocomplete="off"
              label="Stream Manifest URL"
              id="streamUrl"
              onpaste={handlePaste}
              class="w-full !m-0"
              bind:value={formData.streamUrl}
              oninput={validateField}
              error={errors.streamUrl}
            />
            <p class="text-[11px] text-on-surface-variant px-1 italic">
              Supports .m3u8, .mpd, or progressive MP4.
            </p>
          </div>
          <div class="space-y-2">
            <Select
              label="Format"
              options={streamTypes}
              bind:value={formData.streamType}
              class="!m-0"
            />
          </div>
        </div>
      </section>

      <!-- Headers Config -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-2 border-b border-white/5">
          <div class="w-8 h-8 rounded-lg bg-secondary/10 flex items-center justify-center">
            <Icon icon={webIcon} class="text-secondary text-lg" />
          </div>
          <h3 class="text-lg font-bold text-on-surface">Network Headers</h3>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <TextField autocomplete="off" label="Origin" id="origin" bind:value={formData.origin} />
          <TextField autocomplete="off" label="Referer" id="referer" bind:value={formData.referer} />
          <TextField autocomplete="off" label="Cookie" id="cookie" bind:value={formData.cookie} />
          <TextField autocomplete="off" label="User-Agent Override" id="userAgent" bind:value={formData.userAgent} />
          <div class="md:col-span-2">
            <TextFieldMultiline
              autocomplete="off"
              label="Additional Request Headers"
              id="requestHeaders"
              placeholder="Key: Value (one per line)"
              bind:value={formData.requestHeaders}
            />
          </div>
        </div>
      </section>

      <!-- DRM Config -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-2 border-b border-white/5">
          <div class="w-8 h-8 rounded-lg bg-tertiary/10 flex items-center justify-center">
            <Icon icon={shieldIcon} class="text-tertiary text-lg" />
          </div>
          <h3 class="text-lg font-bold text-on-surface">Content Protection (DRM)</h3>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <Select
            label="DRM Scheme"
            id="drmScheme"
            onchange={() => formData.drmScheme === "clearkey_inline" && tick().then(() => document.getElementById("clearKey")?.focus())}
            options={drmSchemes}
            bind:value={formData.drmScheme}
          />

          {#if formData.drmScheme === "clearkey_inline"}
            <TextField autocomplete="off" label="ClearKey (kid:key)" id="clearKey" bind:value={formData.clearKey} />
          {/if}

          {#if !["none", "clearkey_inline"].includes(formData.drmScheme)}
            <div class="md:col-span-2 grid grid-cols-1 md:grid-cols-2 gap-6">
              <TextField autocomplete="off" label="License Server URL" id="licenseUrl" bind:value={formData.licenseUrl} />
              <TextField autocomplete="off" label="Certificate URL (Optional)" id="certificateUrl" bind:value={formData.certificateUrl} />
              <TextFieldMultiline label="License Headers" id="licenseHeaders" bind:value={formData.licenseHeaders} />
              <TextFieldMultiline label="Certificate Headers" id="certificateHeaders" bind:value={formData.certificateHeaders} />
            </div>
          {/if}
        </div>
      </section>

      <!-- Advanced -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-2 border-b border-white/5">
          <div class="w-8 h-8 rounded-lg bg-white/5 flex items-center justify-center">
            <Icon icon={codeIcon} class="text-on-surface-variant text-lg" />
          </div>
          <h3 class="text-lg font-bold text-on-surface">Advanced Shaka Config</h3>
        </div>
        <TextFieldMultiline
          autocomplete="off"
          label="JSON/JSON5 Configuration"
          id="shakaConfig"
          bind:value={formData.shakaConfig}
          oninput={validateField}
          error={errors.shakaConfig}
        />
      </section>
    </div>

    <aside class="space-y-8">
      <!-- Quick Save -->
      <div class="glass-panel rounded-[2rem] p-6 space-y-6 border border-white/5 shadow-2xl">
        <div class="flex items-center gap-3">
          <Icon icon={contentSaveIcon} class="text-primary text-xl" />
          <h3 class="font-bold text-on-surface">Snapshot</h3>
        </div>
        <div class="space-y-4">
          <TextField
            autocomplete="off"
            label="Profile Name"
            id="savedStreamName"
            bind:value={savedStreamName}
            class="!m-0"
          />
          <Button onclick={saveStream} class="w-full !rounded-xl h-11 bg-white/5 text-on-surface hover:bg-white/10 border border-white/10">
            <Icon icon={contentSaveIcon} size={0.9} />
            <span class="ml-2">Save Profile</span>
          </Button>
        </div>
      </div>

      <!-- Recent Saved -->
      <div class="space-y-4">
        <div class="flex items-center justify-between px-2">
          <h3 class="text-sm font-bold uppercase tracking-widest text-on-surface-variant">Recent Saved</h3>
          <a href="/library" class="text-xs font-semibold text-primary hover:underline">View All</a>
        </div>

        <div class="space-y-3">
          {#if savedStreams.length}
            {#each savedStreams.slice(0, 3) as item}
              <div class="glass-card p-4 group relative overflow-hidden">
                <div class="flex items-center justify-between mb-2">
                  <span class="text-sm font-bold text-on-surface truncate pr-8">{item.name}</span>
                  <button
                    onclick={() => playStreamFromData(item.stream)}
                    class="absolute top-3 right-3 w-8 h-8 rounded-full bg-primary text-white flex items-center justify-center opacity-0 group-hover:opacity-100 transition-opacity shadow-lg shadow-primary/30"
                  >
                    <Icon icon={playCircleIcon} size={0.8} />
                  </button>
                </div>
                <p class="text-[10px] text-on-surface-variant truncate mb-2">{item.stream.streamUrl}</p>
                <div class="flex items-center justify-between">
                  <span class="text-[9px] text-on-surface/40 uppercase tracking-tighter">{formatTimestamp(item.updatedAt ?? item.createdAt)}</span>
                  <a href={`/saved/${item.id}`} class="text-[10px] font-bold text-secondary hover:text-primary transition-colors">Details</a>
                </div>
              </div>
            {/each}
          {:else}
            <div class="p-8 text-center glass-card border-dashed border-white/10">
              <p class="text-xs text-on-surface-variant italic">No saved profiles yet.</p>
            </div>
          {/if}
        </div>
      </div>
    </aside>
  </div>
</div>

<FAB
  title="Play Stream"
  style="position: fixed; bottom: 32px; right: 32px; z-index: 100;"
  color="primary"
  elevation="normal"
  onclick={handleSubmit}
  icon={playCircleIcon}
  text="Play Now"
  class="!rounded-2xl shadow-2xl shadow-primary/40 h-14 px-6"
/>

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

<style>
  :global(.m3-text-field .m3-container),
  :global(.m3-select .m3-container),
  :global(.m3-text-field-multiline .m3-container) {
    background: rgba(0,0,0,0.2) !important;
    border: 1px solid rgba(255,255,255,0.05) !important;
    transition: all 0.3s ease !important;
  }
  :global(.m3-text-field .m3-container:focus-within),
  :global(.m3-select .m3-container:focus-within),
  :global(.m3-text-field-multiline .m3-container:focus-within) {
    border-color: rgba(var(--m3-scheme-primary), 0.5) !important;
    background: rgba(var(--m3-scheme-primary), 0.05) !important;
  }
</style>
