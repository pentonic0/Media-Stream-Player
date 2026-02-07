<script>
  import { onMount, onDestroy } from "svelte";

  // @ts-ignore
  import shaka from "shaka-player/dist/shaka-player.ui.js";
  import { parseHeaders } from "$lib";
  import JSON5 from "json5";
  import { TauriFetchLoader } from "$lib/TauriFetchLoader.js";
  import { SkipToLiveButtonFactory } from "$lib/ShakaLiveButton.js";

  // Register the button with Shaka UI
  shaka.ui.Controls.registerElement(
    "skip_to_live", // Reference name for UI config
    new SkipToLiveButtonFactory()
  );

  /**
   * @type {StreamFormData}
   */
  export let stream;

  let player, ui, video, container;

  /**
   *
   * @param {Object} err
   */
  function onErrorEvent(err) {
    console.error(err);

    /**
     * @param {Number} code
     */
    const getErrorName = (code) => {
      return Object.keys(shaka.util.Error.Code).find(
        (name) => shaka.util.Error.Code[name] === code
      );
    };

    const errName = getErrorName(err.code);
    let heading = `${errName} (${err.message})`;
    let message = "";

    if (shaka.util.Error.Code.BAD_HTTP_STATUS === err.code) {
      message = `Request failed. HTTP Status Code ${err.data[1]}.`;
    }

    if (shaka.util.Error.Code.NO_LICENSE_SERVER_GIVEN === err.code) {
      message = `No license server was given for the key system: ${err.data[0]}`;
    }

    const spinner = container.querySelector(".shaka-spinner-container");
    if (spinner) {
      spinner.innerHTML = `
      <div class="flex justify-center items-center text-center flex-col gap-4 px-8 py-10 glass-panel rounded-3xl border border-red-500/20 shadow-2xl" style="max-width:85%">
        <div class="w-16 h-16 rounded-full bg-red-500/10 flex items-center justify-center text-red-500 mb-2">
          <svg xmlns="http://www.w3.org/2000/svg" width="32" height="32" viewBox="0 0 24 24"><path fill="currentColor" d="M12 17q.425 0 .713-.288T13 16t-.288-.712T12 15t-.712.288T11 16t.288.713T12 17m-1-4h2V7h-2zm1 9q-2.075 0-3.9-.788t-3.175-2.137T2.788 15.9T2 12t.788-3.9t2.137-3.175T8.1 2.788T12 2t3.9.788t3.175 2.137T21.213 8.1T22 12t-.788 3.9t-2.137 3.175t-3.175 2.138T12 22m0-2q3.35 0 5.675-2.325T20 12t-2.325-5.675T12 4T6.325 6.325T4 12t2.325 5.675T12 20m0-8"/></svg>
        </div>
        <div class="space-y-1">
          <h3 class="text-lg font-black text-white uppercase tracking-tight">${errName}</h3>
          <p class="text-sm text-on-surface-variant font-medium">${message || err.message}</p>
        </div>
      </div>
      `;
    }
  }

  onMount(async () => {
    shaka.net.NetworkingEngine.registerScheme(
      "http",
      TauriFetchLoader.parse,
      shaka.net.NetworkingEngine.PluginPriority.PREFERRED,
      false
    );
    shaka.net.NetworkingEngine.registerScheme(
      "https",
      TauriFetchLoader.parse,
      shaka.net.NetworkingEngine.PluginPriority.PREFERRED,
      false
    );
    shaka.net.NetworkingEngine.registerScheme(
      "blob",
      TauriFetchLoader.parse,
      shaka.net.NetworkingEngine.PluginPriority.PREFERRED,
      false
    );

    player = new shaka.Player();
    ui = new shaka.ui.Overlay(player, container, video);

    const config = {
      preferDocumentPictureInPicture: false,
      singleClickForPlayAndPause: false,
      addSeekBar: true,
      addBigPlayButton: false,

      seekBarColors: {
        base: "rgba(255,255,255,.15)",
        buffered: "rgba(255,255,255,.25)",
        played: "rgb(99, 102, 241)", // Indigo 500 to match theme
      },

      volumeBarColors: {
        base: "rgba(255, 255, 255, 0.3)",
        level: "rgb(255, 255, 255)",
      },

      controlPanelElements: [
        "play_pause",
        "time_and_duration",
        "mute",
        "volume",
        "spacer",
        "captions",
        "overflow_menu",
        "picture_in_picture",
        "fullscreen",
      ],
    };
    ui.configure(config);

    player.configure({
      manifest: {
        dash: {
          autoCorrectDrift: true,
          ignoreSuggestedPresentationDelay: true,
        },
        hls: {
          liveSegmentsDelay: 3,
        },
      },
      abr: {
        enabled: true,
        preferNetworkInformationBandwidth: true,
      },
      streaming: {
        observeQualityChanges: true,
        preferNativeDash: true,
        inaccurateManifestTolerance: 0,
        rebufferingGoal: 5,
        bufferingGoal: 15,
        segmentPrefetchLimit: 3,
      },
    });

    player.attach(video);

    const volume = localStorage.getItem("player_volume");
    if (volume) {
      video.volume = volume;
      video.dispatchEvent(new CustomEvent("volumechange"));
    }

    loadStream();
  });

  const loadStream = () => {
    const spinner = container.querySelector(".shaka-spinner-container");
    if (spinner) spinner.setAttribute("style", "display:flex!important");

    let mimeType = null;
    let streamType = "progressive";
    const streamUrlObj = new URL(stream.streamUrl);

    if (stream.streamType === "auto") {
      if (streamUrlObj.pathname.includes(".m3u")) {
        streamType = "hls";
      } else if (streamUrlObj.pathname.includes(".mpd")) {
        streamType = "dash";
      }
    } else {
      streamType = stream.streamType === "application/dash+xml" ? "dash" : "hls";
      mimeType = stream.streamType;
    }

    if (streamType === "dash") player.configure("streaming.lowLatencyMode", false);

    const additionalHeaders = parseHeaders(stream.requestHeaders);
    const licenseHeaders = parseHeaders(stream.licenseHeaders);
    const certificateHeaders = parseHeaders(stream.certificateHeaders);

    const headers = {
      referer: stream.referer.trim() || streamUrlObj.origin + "/",
      origin: stream.origin.trim() || streamUrlObj.origin,
      ...additionalHeaders,
    };

    if (stream.userAgent.trim()) headers["user-agent"] = stream.userAgent.trim();
    if (stream.cookie.trim()) headers["cookie"] = stream.cookie.trim();

    if (stream.drmScheme === "clearkey_inline" && stream.clearKey.trim()) {
      const parts = stream.clearKey.split(":");
      if (parts.length === 2) {
        const clearkeyDRM = { clearKeys: {} };
        clearkeyDRM.clearKeys[parts[0].trim()] = parts[1].trim();
        player.configure("drm", clearkeyDRM);
      }
    }

    if (["com.widevine.alpha", "com.microsoft.playready"].includes(stream.drmScheme)) {
      const drmConfig = { servers: { [stream.drmScheme]: stream.licenseUrl.trim() } };
      if (stream.certificateUrl.trim()) {
        drmConfig.advanced = { [stream.drmScheme]: { serverCertificate: stream.certificateUrl.trim() } };
      }
      player.configure("drm", drmConfig);
    }

    if (stream.drmScheme === "org.w3.clearkey") {
      player.configure("drm", { servers: { "org.w3.clearkey": stream.licenseUrl.trim() } });
    }

    const networkingEngine = player.getNetworkingEngine();
    networkingEngine.registerRequestFilter((type, request) => {
      request.headers = { ...request.headers, ...headers };
      if (type == shaka.net.NetworkingEngine.RequestType.LICENSE) {
        request.headers = { ...request.headers, ...licenseHeaders };
        if (stream.drmScheme === "org.w3.clearkey" && !request.headers["content-type"]) {
          request.headers["content-type"] = "application/json";
        }
      }
      if (type == shaka.net.NetworkingEngine.RequestType.SERVER_CERTIFICATE) {
        request.headers = { ...request.headers, ...certificateHeaders };
      }
    });

    if (stream.shakaConfig.trim()) {
      try {
        const additionalConfig = JSON5.parse(stream.shakaConfig);
        player.configure(additionalConfig);
      } catch (err) {
        console.error(err);
      }
    }

    player.load(stream.streamUrl, null, mimeType).then(() => {
      if (spinner) spinner.removeAttribute("style");
    }).catch(onErrorEvent);
  };

  onDestroy(() => {
    if (ui) ui.destroy();
    if (player) player.destroy();
  });

  function handleVolumeChange(event) {
    localStorage.setItem("player_volume", event.target.volume.toString());
  }

  function handleVolumeControl(event) {
    if (!event.target.closest(".shaka-mute-button, .shaka-volume-bar-container")) return;
    const volumeChange = 0.06;
    if (event.deltaY < 0) video.volume = Math.min(1, video.volume + volumeChange);
    else video.volume = Math.max(0, video.volume - volumeChange);
  }
</script>

<div
  class="!m-0 w-full liv-theme youtube-theme glass-panel"
  bind:this={container}
  on:wheel={handleVolumeControl}
>
  <!-- svelte-ignore a11y-media-has-caption -->
  <video
    on:volumechange={handleVolumeChange}
    bind:this={video}
    {...$$restProps}
    autoplay
    class="w-full h-full aspect-video bg-black"
  ></video>
</div>

<style>
  :global(.shaka-video-container) {
    background: black !important;
  }
  :global(.shaka-controls-container) {
    background: linear-gradient(to top, rgba(0,0,0,0.8) 0%, transparent 100%) !important;
  }
  :global(.shaka-range-container) {
    height: 4px !important;
    transition: height 0.2s ease !important;
  }
  :global(.shaka-range-container:hover) {
    height: 6px !important;
  }
</style>
