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
  import { tick } from "svelte";
  import { openUrl } from "@tauri-apps/plugin-opener";
  import JSON5 from "json5";
  import playCircleIcon from "@iconify-icons/mdi/play-circle";
  import aboutCircleIcon from "@iconify-icons/mdi/about-circle";
  import webIcon from "@iconify-icons/mdi/web";
  import facebookIcon from "@iconify-icons/mdi/facebook";
  import githubIcon from "@iconify-icons/mdi/github";
  import closeIcon from "@iconify-icons/mdi/close";
  import refreshIcon from "@iconify-icons/mdi/refresh";
  import VideoPlayer from "$lib/components/VideoPlayer.svelte";
  import Dialog from "$lib/components/Dialog.svelte";
  import parseCurl from "parse-curl";

  let isModalOpen,
    isAboutOpen = false;

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

  $: isModalOpen
    ? document.body.classList.add("modal-open")
    : document.body.classList.remove("modal-open");
</script>

<div class="app-container">
  <div class="grid grid-cols-12 gap-3 mb-4 fw-input">
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

  <div class="flex items-center my-6">
    <h3 class="text-base font-semibold text-on-body">HEADERS</h3>
    <div class="flex-grow border-t border-outline-variant ml-3"></div>
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
        label="User-Agent"
        id="userAgent"
        placeholder=""
        bind:value={formData.userAgent}
      />
      <span class="block text-on-surface mt-2 text-sm"
        >Value for the User-Agent header</span
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
        >Additional headers for the request in key: value format. One per line.</span
      >
    </div>
  </div>
  <!-- /.grid -->

  <div class="flex items-center my-6">
    <h3 class="text-base font-semibold text-on-body">DRM</h3>
    <div class="flex-grow border-t border-outline-variant ml-3"></div>
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

  <div class="flex items-center my-6">
    <h3 class="text-base font-semibold text-on-body">ADVANCED</h3>
    <div class="flex-grow border-t border-outline-variant ml-3"></div>
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

  <FAB
    title="Reset Form"
    style="position: fixed; bottom: 5%; left: 20px;z-index:10;"
    color="tertiary"
    elevation="normal"
    onclick={resetFormData}
    icon={refreshIcon}
  />

  <FAB
    title="About"
    style="position: fixed; bottom: 5%; left: 80px;z-index:10;"
    color="secondary"
    elevation="normal"
    onclick={() => {
      isAboutOpen = true;
    }}
    icon={aboutCircleIcon}
  />

  <FAB
    title="Play Stream"
    style="position: fixed; bottom: 5%; right: 20px;z-index:10;"
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

<div class="about-modal">
  <Dialog headline="Media Stream Player" bind:open={isAboutOpen} icon={false}>
    {#snippet children()}
      <p class="mb-3">
        &copy; {new Date().getFullYear()}. All rights reserved.
      </p>
      <!-- /.mb-3 -->

      <p class="mb-4">Made with 💗 by Miraz Mac.</p>

      <p>
        <strong>Media Stream Player</strong> does not host media. Users are responsible
        for the content they access and must ensure streams and DRM licenses comply
        with the law.
      </p>
    {/snippet}
    {#snippet buttons()}
      <Button
        iconType="full"
        onclick={async (e) => {
          e.preventDefault();
          await openUrl("https://mirazmac.com");
        }}
        title="Website"
      >
        <Icon icon={webIcon} size={0.9} />
      </Button>

      <Button
        iconType="full"
        style="background:#3b5998"
        onclick={async (e) => {
          e.preventDefault();
          await openUrl("https://fb.com/mirazmac");
        }}
        title="Facebook"
      >
        <Icon icon={facebookIcon} size={0.9} />
      </Button>

      <Button
        iconType="full"
        style="background:#111"
        onclick={async (e) => {
          e.preventDefault();
          await openUrl("https://github.com/MirazMac");
        }}
        title="GitHub"
      >
        <Icon icon={githubIcon} size={0.9} />
      </Button>
    {/snippet}
  </Dialog>
</div>

<Snackbar class="shaka-snack holder" bind:this={snackbar} />
