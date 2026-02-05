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
  import historyIcon from "@iconify-icons/mdi/history";
  import contentSaveIcon from "@iconify-icons/mdi/content-save";
  import pencilIcon from "@iconify-icons/mdi/pencil";
  import trashIcon from "@iconify-icons/mdi/trash-can-outline";
  import closeIcon from "@iconify-icons/mdi/close";
  import refreshIcon from "@iconify-icons/mdi/refresh";
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
  let editingSavedId = null;

  const STORAGE_KEYS = {
    history: "msp_stream_history",
    saved: "msp_saved_streams",
  };
  const HISTORY_LIMIT = 12;

  // Define rules here
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

    /**
     *
     * @param value {String}
     */
    shakaConfig: (value) => {
      if (!value.toString().trim()) return null;

      try {
        JSON5.parse(value);
      } catch (e) {
        return "Must be valid JSON/JSON5";
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

  const getStreamLabel = (stream) => {
    try {
      const url = new URL(stream.streamUrl);
      return url.hostname || stream.streamUrl;
    } catch (error) {
      return stream.streamUrl || "Unknown stream";
    }
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

  /**
   *
   * @param id {String}
   */
  const resetTextAreaHeight = (id) => {
    const textarea = document.getElementById(id);

    if (!textarea) {
      return;
    }

    setTimeout(() => {
      const spaceEvent = new InputEvent("input", {
        bubbles: true,
        cancelable: true,
        inputType: "insertText",
        data: "",
      });
      textarea.dispatchEvent(spaceEvent);
    }, 0);
  };

  /**
   *
   * @param event {InputEvent|Object}
   */
  const validateField = (event) => {
    const input = /** @type {HTMLInputElement | HTMLTextAreaElement} */ (
      event.target
    );
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

  /**
   *
   * @param event {ClipboardEvent}
   */
  const handlePaste = (event) => {
    const pastedText = event.clipboardData.getData("text").trim();
    if (pastedText.startsWith("curl")) {
      event.preventDefault();

      try {
        const parsedData = parseCurl(pastedText);

        if (!parsedData.url) {
          throw new Error("Only cURL bash command is supported.");
        }

        snackbar.show({
          message: "cURL command detected. Autofilled parameters.",
        });

        formData.streamUrl = parsedData.url;

        let headerText = "";

        for (const key in parsedData.header) {
          const headerName = key.trim().toLowerCase();
          const value = parsedData.header[key];

          if (["cookie", "set-cookie"].includes(headerName)) {
            formData.cookie = value;
            continue;
          }

          if (["origin", "referer"].includes(headerName)) {
            formData[headerName] = value;
            continue;
          }

          if (headerName === "user-agent") {
            formData.userAgent = value;
            continue;
          }

          headerText += headerName + " : " + value + "\n";
        }

        formData.requestHeaders = headerText;

        // Dispatch an event to adjust textarea's height
        resetTextAreaHeight("requestHeaders");

        validateField(event);
      } catch (error) {
        snackbar.show({
          message: "Invalid CURL command. " + error,
        });
      }

      return;
    }

    // Try to detect NS Player formatted URLs (Hacky approach will improve later)

    const decodedPaste = decodeURI(pastedText);

    // Try to detect NS Player formatted URLs
    if (decodedPaste.includes("|")) {
      event.preventDefault();
      const [url, search] = decodedPaste.split("|");

      // Build a dummy URL to parse the searchParams
      const nsPlayerURL = new URL("https://google.com?" + search.trim());

      formData.streamUrl = url;

      // Manually trigger validation
      validateField(event);

      const drmScheme = nsPlayerURL.searchParams.get("drmScheme");
      const drmLicense = nsPlayerURL.searchParams.get("drmLicense");

      let autofilled = false;

      ["origin", "userAgent", "referer", "referrer", "cookie"].forEach(
        (key) => {
          const value = nsPlayerURL.searchParams.get(key);
          if (value) {
            formData[key === "referrer" ? "referer" : key] = value;
            autofilled = true;
          }
        }
      );

      switch (drmScheme) {
        case "clearkey":
          // If a : is present it's inline otherwise it's a server
          if (drmLicense.includes(":")) {
            formData.drmScheme = "clearkey_inline";
            formData.clearKey = drmLicense;
          } else {
            formData.drmScheme = "org.w3.clearkey";
            formData.licenseUrl = drmLicense;
          }
          autofilled = true;
          break;
        default:
          break;
      }

      snackbar.show({
        message:
          "NS Player URL detected. " +
          (autofilled
            ? "Autofilled parameters."
            : "No supported paramters found."),
      });

      return;
    }

    // Validate field as a fallback
    validateField(event);
  };

  const handleSubmit = (event) => {
    event.preventDefault();

    for (const name in formData) {
      validateField({ target: { id: name, value: formData[name] } });
    }

    if (Object.keys(errors).length !== 0) {
      const firstKey = Object.keys(errors)[0];
      document.getElementById(firstKey).focus();
      return;
    }

    updateHistory(formData);
    isModalOpen = true;
  };

  const handleDrmSchemeChange = () => {
    if (formData.drmScheme === "clearkey_inline") {
      tick().then(() => {
        setTimeout(() => {
          document.getElementById("clearKey").focus();
        }, 0);
      });
    }
  };

  const handleHeadersPlaceholder = (event) => {
    if (event.target.value.trim().length === 0) {
      event.target.placeholder =
        "Authorization: Bearer\nX-Custom-Header: Value";
    } else {
      event.target.placeholder = "";
    }
  };

  const handleHeadersPlaceholderBlur = (event) => {
    event.target.placeholder = "";
  };

  const resetFormData = () => {
    formData = { ...defaultFormData };
    [
      "requestHeaders",
      "shakaConfig",
      "licenseHeaders",
      "certificateHeaders",
    ].forEach((id) => {
      resetTextAreaHeight(id);
    });

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
    if (editingSavedId) {
      updateSavedStreams(
        savedStreams.map((item) =>
          item.id === editingSavedId
            ? {
                ...item,
                name,
                updatedAt: new Date().toISOString(),
                stream: payload,
              }
            : item
        )
      );
      snackbar?.show({ message: "Saved stream updated." });
    } else {
      updateSavedStreams([
        {
          id: createId(),
          name,
          createdAt: new Date().toISOString(),
          stream: payload,
        },
        ...savedStreams,
      ]);
      snackbar?.show({ message: "Stream saved for later." });
    }

    savedStreamName = "";
    editingSavedId = null;
  };

  const editSavedStream = (item) => {
    editingSavedId = item.id;
    savedStreamName = item.name;
    applyStreamToForm(item.stream);
  };

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

<div class="app-shell space-y-6">
  <header class="panel-card app-header">
    <div>
      <p class="text-xs uppercase tracking-[0.25em] text-on-surface">
        Stream workspace
      </p>
      <h2 class="text-2xl font-semibold text-on-body">Media Stream Player</h2>
      <p class="text-sm text-on-surface">
        Build, save, and launch streams with a modern workspace built for quick
        playback.
      </p>
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
      <div class="flex flex-wrap gap-2">
        <Button variant="outlined" onclick={resetFormData}>
          <Icon icon={refreshIcon} size={0.9} />
          <span class="ml-1">Reset</span>
        </Button>
        <Button onclick={handleSubmit}>
          <Icon icon={playCircleIcon} size={0.9} />
          <span class="ml-1">Play stream</span>
        </Button>
      </div>
    </div>
  </header>

  <div class="grid gap-6 xl:grid-cols-[minmax(0,2.2fr)_minmax(0,1fr)]">
    <div class="space-y-6">
      <div class="panel-card panel-card--elevated space-y-6">
        <div class="grid grid-cols-12 gap-3 fw-input">
          <div class="md:col-span-8 col-span-7">
            <TextField
              autocomplete="off"
              label="Stream URL"
              id="streamUrl"
              onfocus={(e) => {
                e.currentTarget.placeholder = e.currentTarget.value.length
                  ? ""
                  : "Paste NS Player URL or cURL command here for autofill";
              }}
              onblur={(e) => {
                e.currentTarget.placeholder = "";
              }}
              onpaste={handlePaste}
              class="w-full"
              bind:value={formData.streamUrl}
              oninput={validateField}
              error={errors.streamUrl}
            />

            <span
              class={{
                "text-error": errors.streamUrl,

                "text-on-surface": !errors.streamUrl,
                "mt-2 text-sm block": true,
              }}
            >
              {#if errors.streamUrl}
                {errors.streamUrl}
              {:else}
                The URL to the media stream
              {/if}
            </span>
          </div>

          <div class="md:col-span-4 col-span-5">
            <Select
              label="Type"
              options={streamTypes}
              bind:value={formData.streamType}
            />
            <span></span>

            <span class="block text-on-surface mt-3 text-sm"
              >Leave to auto if unsure.</span
            >
          </div>
        </div>
        <!-- ./grid -->

        <div class="section-title">
          <h3 class="text-base font-semibold text-on-body">Headers</h3>
          <div class="section-divider"></div>
        </div>
        <!-- ./flex -->

        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-2 gap-3 fw-input">
          <div>
            <TextField
              autocomplete="off"
              label="Cookie"
              id="cookie"
              bind:value={formData.cookie}
            />
            <span class="block text-on-surface mt-2 text-sm"
              >Value for the Cookie header</span
            >
          </div>
          <!-- ./mb-3 -->

          <div>
            <TextField
              autocomplete="off"
              label="Origin"
              id="origin"
              bind:value={formData.origin}
            />
            <span class="block text-on-surface mt-2 text-sm"
              >Value for the Origin header</span
            >
          </div>
          <!-- ./mb-3 -->

          <div>
            <TextField
              autocomplete="off"
              label="Referer"
              id="referer"
              bind:value={formData.referer}
            />
            <span class="block text-on-surface mt-2 text-sm"
              >Value for the Referer header</span
            >
          </div>
          <!-- ./mb-3 -->

          <div>
            <TextField
              autocomplete="off"
              label="User-Agent (Optional)"
              id="userAgent"
              placeholder=""
              bind:value={formData.userAgent}
            />
            <span class="block text-on-surface mt-2 text-sm"
              >Optional override for the User-Agent header</span
            >
          </div>
          <!-- ./mb-3 -->

          <div class="col-span-1 md:col-span-2 mt-3">
            <TextFieldMultiline
              autocomplete="off"
              label="Additional Headers"
              id="requestHeaders"
              placeholder=""
              onfocus={handleHeadersPlaceholder}
              onblur={handleHeadersPlaceholderBlur}
              bind:value={formData.requestHeaders}
            />
            <span class="block text-on-surface mt-2 text-sm"
              >Additional headers for the request in key: value format. One per
              line.</span
            >
          </div>
        </div>
        <!-- /.grid -->

        <div class="section-title">
          <h3 class="text-base font-semibold text-on-body">DRM</h3>
          <div class="section-divider"></div>
        </div>
        <!-- ./flex -->

        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-2 gap-3 fw-input">
          <div>
            <Select
              label="DRM Scheme"
              id="drmScheme"
              onchange={handleDrmSchemeChange}
              options={drmSchemes}
              bind:value={formData.drmScheme}
            />
            <span class="block text-on-surface mt-3 text-sm"
              >Choose the DRM Scheme</span
            >
          </div>
          <!-- ./mb-3 -->

          {#if formData.drmScheme === "clearkey_inline"}
            <div>
              <TextField
                autocomplete="off"
                label="ClearKeyID:Key"
                id="clearKey"
                bind:value={formData.clearKey}
              />
              <span class="block text-on-surface mt-2 text-sm"
                >Clearkey in kid:key format</span
              >
            </div>
            <!-- ./mb-3 -->
          {/if}

          {#if !["none", "clearkey_inline"].includes(formData.drmScheme)}
            <div>
              <TextField
                autocomplete="off"
                label="License URL"
                id="licenseUrl"
                bind:value={formData.licenseUrl}
              />
              <span class="block text-on-surface mt-2 text-sm"
                >The license server URL</span
              >
            </div>

            {#if formData.drmScheme === "com.widevine.alpha" || formData.drmScheme === "com.microsoft.playready"}
              <div>
                <TextField
                  autocomplete="off"
                  label="Certificate URL"
                  id="certificateUrl"
                  bind:value={formData.certificateUrl}
                />
                <span class="block text-on-surface mt-2 text-sm"
                  >The certificate URL</span
                >
              </div>

              <div></div>
            {/if}

            <div>
              <TextFieldMultiline
                autocomplete="off"
                label="License Headers"
                id="licenseHeaders"
                placeholder=""
                onfocus={handleHeadersPlaceholder}
                onblur={handleHeadersPlaceholderBlur}
                bind:value={formData.licenseHeaders}
              />
              <span class="block text-on-surface mt-2 text-sm"
                >Headers for license request in key: value format</span
              >
            </div>

            {#if formData.drmScheme === "com.widevine.alpha" || formData.drmScheme === "com.microsoft.playready"}
              <div>
                <TextFieldMultiline
                  autocomplete="off"
                  label="Certificate Headers"
                  id="certificateHeaders"
                  placeholder=""
                  onfocus={handleHeadersPlaceholder}
                  onblur={handleHeadersPlaceholderBlur}
                  bind:value={formData.certificateHeaders}
                />
                <span class="block text-on-surface mt-2 text-sm"
                  >Headers for certificate request in key: value format</span
                >
              </div>
            {/if}
          {/if}
        </div>
        <!-- /.grid -->

        <div class="section-title">
          <h3 class="text-base font-semibold text-on-body">Advanced</h3>
          <div class="section-divider"></div>
        </div>
        <!-- ./flex -->

        <div class="fw-input">
          <TextFieldMultiline
            autocomplete="off"
            label="Shaka Player Config"
            id="shakaConfig"
            placeholder=""
            oninput={validateField}
            bind:value={formData.shakaConfig}
            error={errors.shakaConfig}
          />
          <span
            class={{
              "text-error": errors.shakaConfig,

              "text-on-surface": !errors.shakaConfig,
              "mt-2 text-sm block": true,
            }}
          >
            {#if errors.shakaConfig}
              {errors.shakaConfig}
            {:else}
              Additional Shaka player configuration as JSON/JSON5 object
            {/if}
          </span>
        </div>
      </div>
    </div>

    <aside class="space-y-6">
      <div class="panel-card panel-card--glass space-y-4">
        <div class="flex items-center justify-between gap-3">
          <div class="flex items-center gap-2">
            <Icon icon={bookmarkIcon} size={1} />
            <h3 class="text-base font-semibold text-on-body">Saved Streams</h3>
          </div>
          {#if savedStreams.length}
            <span class="text-xs text-on-surface"
              >{savedStreams.length} saved</span
            >
          {/if}
        </div>

        <div class="grid gap-3">
          <TextField
            autocomplete="off"
            label="Save current stream as"
            id="savedStreamName"
            bind:value={savedStreamName}
          />
          <div class="flex flex-wrap gap-2">
            <Button onclick={saveStream}>
              <Icon icon={contentSaveIcon} size={0.9} />
              <span class="ml-1">
                {editingSavedId ? "Update saved stream" : "Save stream"}
              </span>
            </Button>
            {#if editingSavedId}
              <Button
                onclick={() => {
                  editingSavedId = null;
                  savedStreamName = "";
                }}
                variant="outlined"
              >
                <Icon icon={closeIcon} size={0.9} />
                <span class="ml-1">Cancel</span>
              </Button>
            {/if}
          </div>
        </div>

        {#if savedStreams.length}
          <div class="space-y-3">
            {#each savedStreams as item}
              <div class="saved-item">
                <div class="flex items-start justify-between gap-3">
                  <div>
                    <p class="text-sm font-semibold text-on-body">
                      {item.name}
                    </p>
                    <p class="text-xs text-on-surface break-all">
                      {item.stream.streamUrl}
                    </p>
                    <p class="text-xs text-on-surface mt-1">
                      Updated {formatTimestamp(item.updatedAt ?? item.createdAt)}
                    </p>
                  </div>
                  <Button
                    iconType="full"
                    title="Play saved stream"
                    onclick={() => playStreamFromData(item.stream)}
                  >
                    <Icon icon={playCircleIcon} size={0.9} />
                  </Button>
                </div>
                <div class="flex flex-wrap gap-2">
                  <Button
                    variant="outlined"
                    onclick={() => editSavedStream(item)}
                  >
                    <Icon icon={pencilIcon} size={0.8} />
                    <span class="ml-1">Edit</span>
                  </Button>
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
            Save stream setups you use often. They will appear here.
          </p>
        {/if}
      </div>

      <div class="panel-card panel-card--glass space-y-4">
        <div class="flex items-center justify-between gap-3">
          <div class="flex items-center gap-2">
            <Icon icon={historyIcon} size={1} />
            <h3 class="text-base font-semibold text-on-body">Stream History</h3>
          </div>
          {#if streamHistory.length}
            <Button variant="outlined" onclick={clearHistory}>
              Clear
            </Button>
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
                      {getStreamLabel(item.stream)}
                    </p>
                    <p class="text-xs text-on-surface break-all">
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
      </div>
    </aside>
  </div>

  <FAB
    title="Play Stream"
    style="position: fixed; bottom: 4%; right: 28px;z-index:10;"
    color="primary"
    elevation="normal"
    onclick={handleSubmit}
    icon={playCircleIcon}
    text="Play"
  />
</div>
<!-- /.container -->

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
        style="position: absolute;
    top: 4px;
    right: 10px;"
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
