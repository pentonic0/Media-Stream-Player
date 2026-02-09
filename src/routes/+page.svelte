<script>
  import { onMount, tick } from "svelte";
  import Icon from "@iconify/svelte";
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
  import { showToast } from "$lib/toast.svelte.js";
  import parseCurl from "parse-curl";

  let isModalOpen = $state(false);

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
  let formData = $state({ ...defaultFormData });

  let streamTypes = [
    { value: "auto", text: "Auto-Detect" },
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

  /** @type {Record<string, string>} */
  let errors = $state({});
  let streamHistory = $state([]);
  let savedStreams = $state([]);
  let savedStreamName = $state("");

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

  /** @param {keyof typeof rules} name */
  const validateField = (name) => {
    const value = formData[name];
    const rule = rules[name];
    if (rule) {
      const error = rule(value);
      if (error) {
        errors[name] = error;
      } else {
        delete errors[name];
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
        showToast("cURL command detected. Autofilled parameters.");
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
        validateField("streamUrl");
      } catch (error) {
        showToast("Invalid CURL command. " + error);
      }
      return;
    }

    const decodedPaste = decodeURI(pastedText);
    if (decodedPaste.includes("|")) {
      event.preventDefault();
      const [url, search] = decodedPaste.split("|");
      const nsPlayerURL = new URL("https://google.com?" + search.trim());
      formData.streamUrl = url;
      validateField("streamUrl");
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
      showToast("NS Player URL detected. " + (autofilled ? "Autofilled parameters." : "No supported parameters found."));
      return;
    }
    setTimeout(() => validateField("streamUrl"), 0);
  };

  const handleSubmit = (event) => {
    if (event) event.preventDefault();
    /** @type {(keyof typeof rules)[]} */
    const ruleKeys = /** @type {any} */ (Object.keys(rules));
    for (const name of ruleKeys) {
      validateField(name);
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
    errors = {};
  };

  const saveStream = () => {
    const name = savedStreamName.trim();
    if (!name.length) {
      showToast("Provide a name to save this stream.");
      return;
    }
    const urlError = rules.streamUrl(formData.streamUrl);
    if (urlError) {
      showToast(urlError);
      return;
    }
    const payload = getStreamPayload(formData);
    updateSavedStreams([
      { id: createId(), name, createdAt: new Date().toISOString(), stream: payload },
      ...savedStreams,
    ]);
    showToast("Stream saved for later.");
    savedStreamName = "";
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
      <div class="hero-badge bg-primary/10 text-primary border border-primary/20 px-3 py-1 rounded-full text-[10px] font-bold uppercase tracking-wider inline-block">Stream Workspace</div>
      <h1 class="text-4xl font-black text-white tracking-tight">New Session</h1>
      <p class="text-on-surface-variant text-sm max-w-lg leading-relaxed">
        Configure your stream parameters below. Support for <span class="text-white font-medium">DASH, HLS</span>, and various <span class="text-white font-medium">DRM schemes</span> with advanced header overrides.
      </p>
    </div>
    <div class="flex items-center gap-3">
      <button
        onclick={resetFormData}
        class="flex items-center gap-2 px-5 py-2.5 rounded-xl border border-white/10 text-on-surface-variant hover:text-white hover:bg-white/5 transition-all text-sm font-semibold"
      >
        <Icon icon={refreshIcon} />
        Clear Form
      </button>
      <button
        onclick={handleSubmit}
        class="flex items-center gap-2 px-6 py-2.5 rounded-xl bg-primary text-white shadow-lg shadow-primary/25 hover:scale-[1.02] active:scale-[0.98] transition-all text-sm font-bold"
      >
        <Icon icon={playCircleIcon} class="text-lg" />
        Launch Player
      </button>
    </div>
  </header>

  <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">
    <div class="lg:col-span-2 space-y-8">
      <!-- Main Config -->
      <section class="glass-card p-8 space-y-6 relative overflow-hidden">
        <div class="absolute top-0 right-0 w-32 h-32 bg-primary/5 rounded-full blur-3xl -mr-16 -mt-16"></div>

        <div class="flex items-center gap-3 pb-4 border-b border-white/5 relative z-10">
          <div class="w-10 h-10 rounded-xl bg-primary/10 flex items-center justify-center border border-primary/10">
            <Icon icon={tuneIcon} class="text-primary text-xl" />
          </div>
          <div>
            <h3 class="text-lg font-bold text-white leading-none">Base Configuration</h3>
            <p class="text-[11px] text-on-surface-variant mt-1">Core stream details and format</p>
          </div>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-4 gap-6 relative z-10">
          <div class="md:col-span-3 space-y-2">
            <label for="streamUrl" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">Stream Manifest URL</label>
            <div class="relative group">
              <input
                id="streamUrl"
                type="text"
                autocomplete="off"
                placeholder="https://example.com/manifest.mpd"
                onpaste={handlePaste}
                bind:value={formData.streamUrl}
                oninput={() => validateField('streamUrl')}
                class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-3 text-sm text-white focus:outline-none focus:border-primary/50 focus:bg-primary/5 transition-all {errors.streamUrl ? 'border-red-500/50' : ''}"
              />
              {#if errors.streamUrl}
                <p class="text-[10px] text-red-400 mt-1 ml-1">{errors.streamUrl}</p>
              {/if}
            </div>
          </div>
          <div class="space-y-2">
            <label for="streamType" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">Format</label>
            <select
              id="streamType"
              bind:value={formData.streamType}
              class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-3 text-sm text-white focus:outline-none focus:border-primary/50 focus:bg-primary/5 transition-all appearance-none"
            >
              {#each streamTypes as type}
                <option value={type.value}>{type.text}</option>
              {/each}
            </select>
          </div>
        </div>
      </section>

      <!-- Headers Config -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-4 border-b border-white/5">
          <div class="w-10 h-10 rounded-xl bg-secondary/10 flex items-center justify-center border border-secondary/10">
            <Icon icon={webIcon} class="text-secondary text-xl" />
          </div>
          <div>
            <h3 class="text-lg font-bold text-white leading-none">Network Headers</h3>
            <p class="text-[11px] text-on-surface-variant mt-1">Bypass restrictions and CORS</p>
          </div>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <div class="space-y-1.5">
            <label for="origin" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">Origin</label>
            <input id="origin" type="text" bind:value={formData.origin} placeholder="https://site.com" class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-secondary/50 focus:bg-secondary/5 transition-all" />
          </div>
          <div class="space-y-1.5">
            <label for="referer" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">Referer</label>
            <input id="referer" type="text" bind:value={formData.referer} placeholder="https://site.com/player" class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-secondary/50 focus:bg-secondary/5 transition-all" />
          </div>
          <div class="space-y-1.5">
            <label for="cookie" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">Cookie</label>
            <input id="cookie" type="text" bind:value={formData.cookie} placeholder="session=..." class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-secondary/50 focus:bg-secondary/5 transition-all" />
          </div>
          <div class="space-y-1.5">
            <label for="userAgent" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">User-Agent</label>
            <input id="userAgent" type="text" bind:value={formData.userAgent} placeholder="Mozilla/5.0..." class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-secondary/50 focus:bg-secondary/5 transition-all" />
          </div>
          <div class="md:col-span-2 space-y-1.5">
            <label for="requestHeaders" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">Custom Request Headers</label>
            <textarea
              id="requestHeaders"
              placeholder="X-Custom-Header: Value"
              bind:value={formData.requestHeaders}
              class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-3 text-sm text-white focus:outline-none focus:border-secondary/50 focus:bg-secondary/5 transition-all min-h-[100px] font-mono text-xs"
            ></textarea>
          </div>
        </div>
      </section>

      <!-- DRM Config -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-4 border-b border-white/5">
          <div class="w-10 h-10 rounded-xl bg-pink-500/10 flex items-center justify-center border border-pink-500/10">
            <Icon icon={shieldIcon} class="text-pink-400 text-xl" />
          </div>
          <div>
            <h3 class="text-lg font-bold text-white leading-none">Content Protection</h3>
            <p class="text-[11px] text-on-surface-variant mt-1">Manage DRM and license servers</p>
          </div>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
          <div class="space-y-1.5">
            <label for="drmScheme" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">DRM Scheme</label>
            <select
              id="drmScheme"
              bind:value={formData.drmScheme}
              onchange={() => formData.drmScheme === "clearkey_inline" && tick().then(() => document.getElementById("clearKey")?.focus())}
              class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-pink-500/50 focus:bg-pink-500/5 transition-all appearance-none"
            >
              {#each drmSchemes as scheme}
                <option value={scheme.value}>{scheme.text}</option>
              {/each}
            </select>
          </div>

          {#if formData.drmScheme === "clearkey_inline"}
            <div class="space-y-1.5">
              <label for="clearKey" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">ClearKey (kid:key)</label>
              <input id="clearKey" type="text" bind:value={formData.clearKey} placeholder="deadbeef...:deadbeef..." class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-pink-500/50 focus:bg-pink-500/5 transition-all" />
            </div>
          {/if}

          {#if !["none", "clearkey_inline"].includes(formData.drmScheme)}
            <div class="md:col-span-2 grid grid-cols-1 md:grid-cols-2 gap-6">
              <div class="space-y-1.5">
                <label for="licenseUrl" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">License Server URL</label>
                <input id="licenseUrl" type="text" bind:value={formData.licenseUrl} class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-pink-500/50 focus:bg-pink-500/5 transition-all" />
              </div>
              <div class="space-y-1.5">
                <label for="certificateUrl" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">Certificate URL (Optional)</label>
                <input id="certificateUrl" type="text" bind:value={formData.certificateUrl} class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-pink-500/50 focus:bg-pink-500/5 transition-all" />
              </div>
              <div class="space-y-1.5">
                <label for="licenseHeaders" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">License Headers</label>
                <textarea id="licenseHeaders" bind:value={formData.licenseHeaders} class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-3 text-sm text-white focus:outline-none focus:border-pink-500/50 focus:bg-pink-500/5 transition-all min-h-[80px] font-mono text-xs"></textarea>
              </div>
              <div class="space-y-1.5">
                <label for="certificateHeaders" class="text-[11px] font-bold text-on-surface-variant uppercase tracking-wider ml-1">Certificate Headers</label>
                <textarea id="certificateHeaders" bind:value={formData.certificateHeaders} class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-3 text-sm text-white focus:outline-none focus:border-pink-500/50 focus:bg-pink-500/5 transition-all min-h-[80px] font-mono text-xs"></textarea>
              </div>
            </div>
          {/if}
        </div>
      </section>

      <!-- Advanced -->
      <section class="glass-card p-8 space-y-6">
        <div class="flex items-center gap-3 pb-4 border-b border-white/5">
          <div class="w-10 h-10 rounded-xl bg-white/5 flex items-center justify-center border border-white/10">
            <Icon icon={codeIcon} class="text-white text-xl" />
          </div>
          <div>
            <h3 class="text-lg font-bold text-white leading-none">Advanced Player Config</h3>
            <p class="text-[11px] text-on-surface-variant mt-1">Shaka Player JSON/JSON5 overrides</p>
          </div>
        </div>
        <div class="space-y-1.5">
           <textarea
            id="shakaConfig"
            placeholder="{ '{ "streaming": { "bufferingGoal": 60 } }' }"
            bind:value={formData.shakaConfig}
            oninput={() => validateField('shakaConfig')}
            class="w-full bg-black/20 border border-white/5 rounded-xl px-4 py-4 text-sm text-white focus:outline-none focus:border-white/20 focus:bg-white/5 transition-all min-h-[120px] font-mono text-xs {errors.shakaConfig ? 'border-red-500/50' : ''}"
          ></textarea>
          {#if errors.shakaConfig}
            <p class="text-[10px] text-red-400 mt-1 ml-1">{errors.shakaConfig}</p>
          {/if}
        </div>
      </section>
    </div>

    <aside class="space-y-8">
      <!-- Quick Save -->
      <div class="relative overflow-hidden glass-panel rounded-[2.5rem] p-7 space-y-6 border border-white/10 shadow-2xl">
        <div class="absolute -left-10 -bottom-10 w-32 h-32 bg-primary/10 rounded-full blur-3xl"></div>

        <div class="flex items-center gap-3 relative z-10">
          <div class="w-8 h-8 rounded-lg bg-primary/20 flex items-center justify-center">
            <Icon icon={contentSaveIcon} class="text-primary text-lg" />
          </div>
          <h3 class="font-bold text-white">Save Profile</h3>
        </div>

        <div class="space-y-4 relative z-10">
          <div class="space-y-1.5">
            <label for="savedStreamName" class="text-[10px] font-bold text-on-surface-variant uppercase tracking-widest ml-1">Profile Name</label>
            <input
              id="savedStreamName"
              type="text"
              placeholder="E.g. My Live TV"
              bind:value={savedStreamName}
              class="w-full bg-white/5 border border-white/5 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-primary/50 transition-all"
            />
          </div>
          <button
            onclick={saveStream}
            class="w-full py-3 rounded-xl bg-white/5 hover:bg-white/10 text-white text-xs font-bold border border-white/10 transition-all flex items-center justify-center gap-2"
          >
            <Icon icon={contentSaveIcon} />
            Store Permanently
          </button>
        </div>
      </div>

      <!-- Recent Saved -->
      <div class="space-y-5">
        <div class="flex items-center justify-between px-2">
          <h3 class="text-[10px] font-black uppercase tracking-[0.2em] text-on-surface-variant">Recent Saved</h3>
          <a href="/library" class="text-[10px] font-bold text-primary hover:text-primary/80 transition-colors uppercase tracking-widest">View All</a>
        </div>

        <div class="space-y-3">
          {#if savedStreams.length}
            {#each savedStreams.slice(0, 3) as item}
              <div class="glass-card p-4 group relative overflow-hidden hover:border-primary/30 transition-all">
                <div class="flex items-center justify-between mb-2">
                  <span class="text-sm font-bold text-white truncate pr-8">{item.name}</span>
                  <button
                    onclick={() => playStreamFromData(item.stream)}
                    class="absolute top-3 right-3 w-8 h-8 rounded-full bg-primary text-white flex items-center justify-center opacity-0 group-hover:opacity-100 transition-all shadow-lg shadow-primary/30 scale-90 group-hover:scale-100"
                  >
                    <Icon icon={playCircleIcon} />
                  </button>
                </div>
                <p class="text-[10px] text-on-surface-variant truncate mb-3 opacity-60 font-mono">{item.stream.streamUrl}</p>
                <div class="flex items-center justify-between border-t border-white/5 pt-3">
                  <span class="text-[9px] text-on-surface/40 font-bold uppercase tracking-tighter">{formatTimestamp(item.updatedAt ?? item.createdAt)}</span>
                  <a href={`/saved/${item.id}`} class="text-[10px] font-black text-secondary hover:underline uppercase tracking-tighter">Details</a>
                </div>
              </div>
            {/each}
          {:else}
            <div class="p-8 text-center glass-card border-dashed border-white/10 opacity-50">
              <Icon icon={libraryIcon} class="mx-auto text-2xl mb-2 text-on-surface-variant" />
              <p class="text-[10px] text-on-surface-variant uppercase font-bold tracking-widest">No profiles yet</p>
            </div>
          {/if}
        </div>
      </div>
    </aside>
  </div>
</div>

<button
  onclick={handleSubmit}
  class="fixed bottom-8 right-8 z-[100] flex items-center gap-3 px-6 py-4 rounded-2xl bg-primary text-white shadow-2xl shadow-primary/40 hover:scale-105 active:scale-95 transition-all group"
>
  <div class="w-6 h-6 rounded-lg bg-white/20 flex items-center justify-center group-hover:rotate-12 transition-transform">
    <Icon icon={playCircleIcon} class="text-lg" />
  </div>
  <span class="font-black text-sm uppercase tracking-wider">Play Now</span>
</button>

<div class="player-modal">
  <Dialog
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


<style>
  textarea {
    resize: vertical;
  }
</style>
