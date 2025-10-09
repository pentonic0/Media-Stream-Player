<script>
  import { onMount, onDestroy } from "svelte";

  import { Snackbar } from "m3-svelte";

  // @ts-ignore
  import shaka from "shaka-player/dist/shaka-player.ui.js";
  import { parseHeaders } from "$lib";

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

  /**
   * @type {Snackbar}
   */
  let snackbar;

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
    snackbar.show({
      message: getErrorName(err.code) + " " + err.message,
      closable: false,
    });
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
        base: "rgba(255,255,255,.2)",
        buffered: "rgba(255,255,255,.4)",
        played: "rgb(255,0,0)",
      },

      volumeBarColors: {
        base: "rgba(255, 255, 255, 0.54)",
        level: "rgb(255, 255, 255)",
      },

      controlPanelElements: [
        "play_pause",

        "skip_to_live",

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
          //ignoreMinBufferTime: true,
          ignoreSuggestedPresentationDelay: true,
          //updatePeriod: 10,
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
        rebufferingGoal: 5,
        bufferingGoal: 15,
        //lowLatencyMode: true,
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

    if (spinner) {
      spinner.setAttribute("style", "display:flex!important");
    }

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
      streamType =
        stream.streamType === "application/dash+xml" ? "dash" : "hls";
      mimeType = stream.streamType;
    }

    if (streamType === "dash") {
      player.configure("streaming.lowLatencyMode", false);
    }

    const additionalHeaders = parseHeaders(stream.requestHeaders);
    const licenseHeaders = parseHeaders(stream.licenseHeaders);
    const certificateHeaders = parseHeaders(stream.certificateHeaders);

    // Make an headers object with the necessary headers
    // Accurate referer and origin must be set otherwise some servers will reject the request
    const headers = {
      referer:
        stream.referer.trim().length > 0
          ? stream.referer.trim()
          : streamUrlObj.origin + "/",
      origin:
        stream.origin.trim().length > 0
          ? stream.origin.trim()
          : streamUrlObj.origin,
      ...additionalHeaders,
    };

    if (stream.userAgent.trim().length) {
      headers["user-agent"] = stream.userAgent.trim();
    }

    if (stream.cookie.trim().length) {
      headers["cookie"] = stream.cookie.trim();
    }

    // Handle inline clearkey DRM
    if (
      stream.drmScheme === "clearkey_inline" &&
      stream.clearKey.trim().length > 0
    ) {
      const parts = stream.clearKey.split(":");

      if (parts.length === 2) {
        const kid = parts[0].trim();
        const kvalue = parts[1].trim();

        const clearkeyDRM = {
          clearKeys: {},
        };

        clearkeyDRM.clearKeys[kid] = kvalue;

        player.configure("drm", clearkeyDRM);
      }
    }

    if (
      stream.drmScheme === "com.widevine.alpha" ||
      stream.drmScheme === "com.microsoft.playready"
    ) {
      const widevineConfig = {
        servers: {
          [stream.drmScheme]: stream.licenseUrl.trim(),
        },
      };

      if (stream.certificateUrl.trim().length) {
        widevineConfig.advanced = {
          [stream.drmScheme]: {
            serverCertificate: stream.certificateUrl.trim(),
          },
        };
      }

      player.configure("drm", widevineConfig);
    }

    if (stream.drmScheme === "org.w3.clearkey") {
      const clearKeyConfig = {
        servers: {
          "org.w3.clearkey": stream.licenseUrl.trim(),
        },
      };

      player.configure("drm", clearKeyConfig);
    }

    const networkingEngine = player.getNetworkingEngine();

    // Register a request filter to set headers for specific requests
    // @ts-ignore
    networkingEngine.registerRequestFilter((type, request) => {
      // By default all requests will have the custom request headers set
      request.headers = {
        ...request.headers,
        ...headers,
      };

      // Can be overridden for license and certificate requests
      if (type == shaka.net.NetworkingEngine.RequestType.LICENSE) {
        request.headers = {
          ...request.headers,
          ...licenseHeaders,
        };

        // If content type is missing for clearkey server license request use json as default
        // as many servers expect that
        if (
          stream.drmScheme === "org.w3.clearkey" &&
          !request.headers["content-type"]
        ) {
          request.headers["content-type"] = "application/json";
        }
      }

      if (type == shaka.net.NetworkingEngine.RequestType.SERVER_CERTIFICATE) {
        request.headers = {
          ...request.headers,
          ...certificateHeaders,
        };
      }
    });

    if (stream.shakaConfig.trim().length) {
      try {
        const additionalConfig = JSON.parse(stream.shakaConfig);
        player.configure(additionalConfig);
      } catch (err) {
        console.error(err);
      }
    }

    player
      .load(stream.streamUrl, null, mimeType)
      .then(() => {
        if (spinner) {
          spinner.removeAttribute("style");
        }
      })
      .catch(onErrorEvent);
  };

  onDestroy(() => {
    if (ui) {
      ui.destroy();
    }

    if (player) {
      player.destroy();
    }
  });

  function handleVolumeChange(event) {
    localStorage.setItem("player_volume", event.target.volume);
  }

  function handleVolumeControl(event) {
    if (
      !event.target.closest(".shaka-mute-button, .shaka-volume-bar-container")
    ) {
      return;
    }
    const volumeChange = 0.06;

    if (event.deltaY < 0) {
      // Scrolling up increases volume
      video.volume = Math.min(1, video.volume + volumeChange);
    } else {
      // Scrolling down decreases volume
      video.volume = Math.max(0, video.volume - volumeChange);
    }
  }
</script>

<Snackbar class="shaka-snack holder text-center" bind:this={snackbar} />

<div
  class="!m-0 w-full liv-theme youtube-theme"
  bind:this={container}
  on:wheel={handleVolumeControl}
>
  <!-- svelte-ignore a11y-media-has-caption -->
  <video
    on:volumechange={handleVolumeChange}
    bind:this={video}
    {...$$restProps}
    autoplay
    style="object-fit:contain;min-height:60vh"
    class="w-full h-full"
  ></video>
</div>
