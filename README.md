# YouTube-Style Ambient Mode / “Ambilight”

> Lightweight, dependency-free JavaScript that paints the same blurred-halo YouTube shows behind its videos – plus a playground to tweak every knob.

## ✨ What’s in this repo?

| folder/file  | purpose                                                                                                                                          |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `index.html` | a full playground UI where you can live-tune blur radius, sampling interval, overscan, position (behind/overlay/below), single-colour mode, etc. |
| `README.md`  | **← you are here** – in-depth explanation of the algorithm + a dead-simple “drop-in” snippet for your own site.                                  |

---

## 1 Algorithm, step-by-step

Below is a plain-English mapping of YouTube’s obfuscated `YH → QNT → OVJ` pipeline to the names you will see in the code. (see the original code below)

| stage                       | YouTube code (symbol)                               | what we do                                                                                                                                                                         |
| --------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. choose source pixels** | storyboard sprite (`cMs.drawToCanvas`)              | capture a down-scaled frame of the live `<video>` into an off-screen canvas (`thumb`). Size = `video * scaleDown`.                                                                 |
| **2. clear & base fill**    | `zQo` – dark backdrop                               | fill the final canvas with `#0f0f0f` so edges stay dark in light scenes.                                                                                                           |
| **3. blur & draw**          | `QNT` – `ctx.filter = blur(px)` then `drawImage`    | set `ctx.filter = blurRadius` and blit `thumb` back over the video area, offset by `blurRadius * overscanMult` so the glow bleeds beneath the bezel.                               |
| **4. cross-fade**           | `OVJ` – copy front→back, fade with `steps()` easing | keep two canvases. On each update, back holds the previous glow, front is newly painted, and we animate their opacities in opposite directions so total brightness stays constant. |
| **5. repeat**               | “continuous” 30 fps _or_ “sampled” every _N_ ms     | continuous mode = redraw every animation frame; sampled = swap+fade once per `sampleInt`.                                                                                          |

### Single-colour variant

_Instead_ of drawing the tiny thumbnail, we average a 16×16 frame to one RGB triple, paint a solid 100×100 block of that colour in the centre and blur it – exactly how YouTube’s **dynamic-single-colour** style works.

---

## 2 Constants

| name           | meaning                                      | default                               |
| -------------- | -------------------------------------------- | ------------------------------------- |
| `blurRadius`   | Gaussian blur in px                          | `100` (very soft, HD big-screen look) |
| `scaleDown`    | thumbnail size ÷ video size                  | `0.01`                                |
| `overscanMult` | glow bleed distance = blur × multiplier      | `10`                                  |
| `opacity`      | final canvas alpha                           | `0.4`                                 |
| `xfadeMs`      | cross-fade duration (sampled)                | `350 ms`                              |
| `sampleInt`    | ms between updates (sampled)                 | `2000 ms`                             |
| `glowScale`    | multiply wrapper scale (simulate fullscreen) | `1`                                   |
| `DARK`         | background color                             | `#000000`                             |

---

## 3 Just-the-basics snippet 🪄

Copy-paste the **HTML + JS** below into any page with a `<video>` element.  
No controls, no UI – **just the engine**.

```html
<!-- HTML (same wrapper & canvases) -->
<style>
  body {
    background: #000;
    margin: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
  }

  .ambient-video-wrapper {
    position: relative;
    display: inline-block;
  }

  video {
    display: block;
    max-width: 100%;
    max-height: 100vh;
    z-index: 2;
    position: relative;
  }

  #ambilight {
    position: absolute;
    pointer-events: none;
    z-index: 1;
  }

  #ambilight canvas {
    position: absolute;
    inset: 0;
    width: 100%;
    height: 100%;
    transform: scale(1.2);
  }
</style>

<div class="ambient-video-wrapper">
  <div id="ambilight">
    <canvas id="ambA"></canvas>
    <canvas id="ambB"></canvas>
  </div>
  <video
    id="vid"
    src="https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4"
    controls
    playsinline
  ></video>
</div>

<script>
  (() => {
    /* CONFIG ---------------------------------------------------------------- */
    const blurRadius = 100; // px
    const scaleDown = 0.01; // % of video
    const overscanMult = 10; // glow bleed = blur * mult
    const opacity = 0.5; // final alpha
    const fps = 30;
    /* ----------------------------------------------------------------------- */

    const video = document.getElementById("vid");
    const layer = document.getElementById("ambilight");
    let frontC = document.getElementById("ambA");
    let backC = document.getElementById("ambB");
    const ctxF = frontC.getContext("2d");
    const ctxB = backC.getContext("2d");
    const thumb = document.createElement("canvas");
    const tCtx = thumb.getContext("2d");
    ctxF.globalAlpha = ctxB.globalAlpha = 1;

    const DARK = "#000000";

    /* ───────────────────── accurately size canvas *and* layer ────────────── */
    function resize() {
      if (!video.videoWidth) return;
      const W = video.videoWidth,
        H = video.videoHeight,
        pad = blurRadius * overscanMult; // <─ same pad in draw()

      // 1. canvases
      [frontC, backC].forEach((c) => {
        c.width = W + pad * 2;
        c.height = H + pad * 2;
      });

      // 2. layer element (so CSS box matches canvas box)
      Object.assign(layer.style, {
        width: `${W + pad * 2}px`,
        height: `${H + pad * 2}px`,
        left: `${-pad}px`,
        top: `${-pad}px`,
      });

      // 3. thumbnail src size
      thumb.width = Math.max(16, (W * scaleDown) | 0);
      thumb.height = Math.max(16, (H * scaleDown) | 0);
    }

    /* ───────────────────────── draw one glow frame ───────────────────────── */
    function paintGlow(dst) {
      // tiny frame
      tCtx.drawImage(video, 0, 0, thumb.width, thumb.height);

      // base + blur
      dst.clearRect(0, 0, dst.canvas.width, dst.canvas.height);
      dst.fillStyle = DARK;
      dst.fillRect(0, 0, dst.canvas.width, dst.canvas.height);
      dst.filter = `blur(${blurRadius}px)`;

      const pad = blurRadius * overscanMult;
      dst.drawImage(
        thumb,
        0,
        0,
        thumb.width,
        thumb.height,
        pad,
        pad,
        video.videoWidth,
        video.videoHeight
      );
      dst.filter = "none";
    }

    /* ───────────────────────── loop @ capped fps ─────────────────────────── */
    const FRAME = 1000 / fps;
    let raf,
      last = 0;
    function loop(now) {
      raf = requestAnimationFrame(loop);
      if (video.paused || now - last < FRAME) return;
      last = now;
      paintGlow(ctxF);
      frontC.style.opacity = opacity;
    }

    /* ───────────────────────── wiring ────────────────────────────────────── */
    video.addEventListener("loadedmetadata", resize);
    window.addEventListener("resize", resize);
    video.addEventListener("play", () => {
      resize();
      loop(performance.now());
    });
    video.addEventListener("pause", () => cancelAnimationFrame(raf));
  })();
</script>
```

### How to switch to “sampled” mode + cross-fade

Replace the continuous `loop()` with:

```js
const xfadeMs = 350,
  sampleInt = 2000;
function swapAndFade() {
  if (video.paused) return;
  // copy old glow to back
  ctxB.clearRect(0, 0, backC.width, backC.height);
  ctxB.drawImage(frontC, 0, 0);
  backC.style.opacity = opacity;

  paintGlow(ctxF); // draw new frame
  frontC.style.opacity = 0; // start hidden

  frontC.animate([{ opacity: 0 }, { opacity: opacity }], {
    duration: xfadeMs,
    fill: "forwards",
  });
  backC.animate([{ opacity: opacity }, { opacity: 0 }], {
    duration: xfadeMs,
    fill: "forwards",
  });
}
video.addEventListener("play", () => {
  resize();
  swapAndFade();
  setInterval(swapAndFade, sampleInt);
});
```

---

## 4 Running the playground locally

```bash
git clone https://github.com/your-handle/youtube-ambient-mode.git
cd youtube-ambient-mode
# you can just double-click index.html …
python3 -m http.server 8000    # …or serve it for CORS-free video playback
```

Open [http://localhost:8000](http://localhost:8000), drag your own MP4/WEBM into the player, and tweak away.

---

## 5 Original Reversed Engineered Code (minified and may be incomplete)

```js
        _.C(vN, "ytd-video-preview", function() {
            if (Cod !== void 0)
                return Cod;
            var S = document.createElement("template");
            _.h(S, '\x3c!--css-build:shady--\x3e\x3c!--css_build_scope:ytd-video-preview--\x3e\x3c!--css_build_styles:video.youtube.src.web.polymer.shared.ui.styles.yt_base_styles.yt.base.styles.css.js--\x3e<div id="video-preview-container" class="style-scope ytd-video-preview">\n  <div id="endorsement" class="style-scope ytd-video-preview"></div>\n  <div id="media-container" class="style-scope ytd-video-preview">\n    <a id="media-container-link" class="yt-simple-endpoint style-scope ytd-video-preview" href$="[[computeHref_(videoPreviewData.navigationEndpoint)]]" data="[[videoPreviewData.navigationEndpoint]]" aria-label$="[[videoPreviewData.accessibilityText]]" on-click="onMediaContainerClick">\n      <div id="thumbnail-container" class="style-scope ytd-video-preview">\n        <ytd-thumbnail data="[[thumbnailData]]" hovered="false" no-rounded-corners="" object-fit="COVER" rich-grid-thumbnail="" width="9999" class="style-scope ytd-video-preview">\n        </ytd-thumbnail>\n      </div>\n      <div id="player-container" class="style-scope ytd-video-preview">\n        <ytd-player id="inline-player" context="WEB_PLAYER_CONTEXT_CONFIG_ID_KEVLAR_INLINE_PREVIEW" class="style-scope ytd-video-preview">\n        </ytd-player>\n      </div>\n      <div id="overlays" class="style-scope ytd-video-preview"></div>\n    </a>\n    <div id="player-controls" class="style-scope ytd-video-preview">\n      <template is="dom-if" if="[[!!playerControlsData]]" class="style-scope ytd-video-preview">\n        <yt-inline-player-controls app-api="[[playerControlsAppApi]]" data="[[playerControlsData]]" options="[[playerControlsOptions]]" class="style-scope ytd-video-preview"></yt-inline-player-controls>\n      </template>\n    </div>\n  </div>\n</div>\n');
            S.content.insertBefore(_.t().content.cloneNode(!0), S.content.firstChild);
            return Cod = S
        }, {
            mode: 2
        });
    } catch (e) {
        _._DumpException(e)
    }
    try {
        var $yo, iiJ, NFX;
        $yo = function(S) {
            return _.y("kevlar_watch_cinematics_invisible") || S.fullscreen && _.y("kevlar_watch_cinematics_invisible_in_fullscreen") || S.theater && !S.fullscreen && _.y("kevlar_watch_cinematics_invisible_in_theater")
        }
        ;
        iiJ = function() {
            var S = document.createElement("canvas")
              , p = S.getContext("2d");
            if (!p)
                throw Error("Wl");
            _.R3(S, {
                position: "absolute",
                width: "100%",
                height: "100%"
            });
            return {
                element: S,
                context: p
            }
        }
        ;
        NFX = function() {
            return !("filter"in CanvasRenderingContext2D.prototype) || _.y("kevlar_watch_cinematics_css_blur")
        }
        ;
        _.BFE = function(S, p, U) {
            return Math.abs(S - p) <= (U || 1E-6)
        }
        ;
        _.qJ5 = function(S, p) {
            return S == p ? !0 : S && p ? S.width == p.width && S.height == p.height : !1
        }
        ;
        _.AMJ = new _.v("notificationActionRenderer");
        var trT = function(S) {
            var p = this;
            this.element = new Image;
            this.failed = this.loaded = !1;
            this.resolver = new _.MI;
            this.element.addEventListener("load", function() {
                p.loaded = !0;
                p.resolver.resolve(p.element)
            });
            this.element.addEventListener("error", function() {
                p.failed = !0
            });
            this.element.src = S
        };
        var cMs = function(S, p) {
            this.image = S;
            this.frame = p
        };
        cMs.prototype.drawToCanvas = function(S, p) {
            var U = this.frame.width / this.frame.columns
              , Y = this.frame.height / this.frame.rows
              , L = p.offsetX
              , I = p.offsetY;
            $yo(p) ? (S.fillStyle = p.fullscreen ? "#000" : "#0f0f0f",
            S.fillRect(L, I, (p == null ? void 0 : p.width) || U, (p == null ? void 0 : p.height) || Y)) : S.drawImage(this.image, this.frame.column * U, this.frame.row * Y, U, Y, L, I, (p == null ? void 0 : p.width) || U, (p == null ? void 0 : p.height) || Y)
        }
        ;
        var SqA = function(S) {
            this.color = S
        };
        SqA.prototype.drawToCanvas = function(S, p) {
            var U = p.offsetX
              , Y = p.offsetY
              , L = p.width
              , I = p.height;
            S.fillStyle = $yo(p) ? p.fullscreen ? "#000" : "#0f0f0f" : this.color;
            S.fillRect(U, Y, L, I)
        }
        ;
        var SE = function(S, p) {
            _.qS.call(this);
            this.playerApi = p;
            this.mosaics = new Map;
            this.colorStore = new Map;
            this.pendingStoryboardIndex = this.currentStoryboardIndex = this.colorStoreTimeInterval = NaN;
            this.currentStoryboardSize = new _.OF(NaN,NaN);
            this.lastUpdateTime = NaN;
            this.paused = !1;
            this.addEventListeners();
            pfT(this, S);
            Uqd(this);
            this.update()
        };
        _.r(SE, _.qS);
        SE.prototype.addEventListeners = function() {
            var S = this
              , p = function() {
                S.update()
            }
              , U = function(L) {
                S.paused || (L.type === "newdata" && (S.mosaics.clear(),
                YqL(S),
                pw(S)),
                Uqd(S),
                S.update())
            }
              , Y = function() {
                S.onPlayerStateChange()
            };
            this.playerApi.addEventListener("onVideoProgress", p);
            this.playerApi.addEventListener("onVideoDataChange", U);
            this.playerApi.addEventListener("onStateChange", Y);
            this.addOnDisposeCallback(function() {
                S.playerApi.removeEventListener("onVideoProgress", p);
                S.playerApi.removeEventListener("onVideoDataChange", U);
                S.playerApi.removeEventListener("onStateChange", Y)
            })
        }
        ;
        var pfT = function(S, p) {
            S.cinematicContainerRenderer !== p && (S.cinematicContainerRenderer = p,
            YqL(S),
            pw(S),
            S.colorStoreUpdateJobId = _.PA.addLowPriorityJob(function() {
                var U;
                if ((U = S.cinematicContainerRenderer.colorStore) != null && U.sampledColors) {
                    U = Infinity;
                    for (var Y = _.d(S.cinematicContainerRenderer.colorStore.sampledColors), L = Y.next(); !L.done; L = Y.next()) {
                        L = L.value;
                        var I = Number(L.key);
                        I !== 0 && I < U && (U = I);
                        I = _.ev(L.value);
                        S.colorStore.set(L.key, I)
                    }
                    S.colorStoreTimeInterval = U
                }
            }))
        }
          , YqL = function(S) {
            S.colorStoreUpdateJobId && (_.PA.cancelJob(S.colorStoreUpdateJobId),
            S.colorStoreUpdateJobId = void 0);
            S.colorStore.clear();
            S.currentStoryboardColor = void 0
        }
          , Lyd = function(S, p) {
            var U;
            return (U = S.getStoryboardFrame(p)) == null ? void 0 : U.url
        };
        SE.prototype.onPlayerStateChange = function() {
            this.update()
        }
        ;
        SE.prototype.isAdPlaying = function() {
            return this.playerApi.getPresentingPlayerType() === 2
        }
        ;
        var Uqd = function(S) {
            var p = S.getStoryboardFrame(0);
            p && (p = new _.OF(p.width / p.columns,p.height / p.rows),
            _.qJ5(S.currentStoryboardSize, p) || (S.currentStoryboardSize = p,
            S.publish("STORYBOARD_SIZE_CHANGED", S.currentStoryboardSize)))
        }
          , M2o = function(S, p) {
            S.currentStoryboardIndex = p;
            S.pendingStoryboardIndex = NaN;
            p = S.getStoryboardFrame(S.currentStoryboardIndex);
            S.currentStoryboard = new cMs(S.mosaics.get(p.url).element,p);
            S.publish("STORYBOARD_CHANGED", S.currentStoryboard);
            S.lastUpdateTime = (0,
            _.AI)()
        };
        SE.prototype.isShorts = function() {
            return this.cinematicContainerRenderer.config.pageType === "CINEMATIC_CONTAINER_PAGE_TYPE_SHORTS"
        }
        ;
        var pw = function(S) {
            S.currentStoryboardIndex = NaN;
            S.pendingStoryboardIndex = NaN;
            S.currentStoryboard && (S.currentStoryboard = void 0,
            S.publish("STORYBOARD_CHANGED", void 0));
            S.lastUpdateTime = NaN
        };
        SE.prototype.update = function() {
            if (!this.paused && this.playerApi.getNumberOfStoryboardLevels() > 0)
                if (this.isAdPlaying() || this.isShorts() && this.playerApi.getProgressState().duration < 15)
                    pw(this);
                else {
                    var S = this.playerApi.getPlayerState(1);
                    if (S === -1 || S === 5 || S === 0)
                        pw(this);
                    else if (isNaN(this.lastUpdateTime) || !((0,
                    _.AI)() < this.lastUpdateTime + this.cinematicContainerRenderer.config.animationConfig.minImageUpdateIntervalMs))
                        if (S = this.playerApi.getCurrentTime() + (this.playerApi.getPlayerState(1) === 2 ? 0 : this.cinematicContainerRenderer.config.animationConfig.crossfadeDurationMs * this.cinematicContainerRenderer.config.animationConfig.crossfadeStartOffset / 1E3),
                        this.cinematicContainerRenderer.presentationStyle === "CINEMATIC_CONTAINER_PRESENTATION_STYLE_DYNAMIC_SINGLE_COLOR")
                            if (this.colorStore.size) {
                                S = "" + Math.round(S * 1E3 / this.colorStoreTimeInterval) * this.colorStoreTimeInterval;
                                var p = this.colorStore.get(S);
                                p ? p !== this.currentStoryboardColor && (this.currentStoryboardColor = this.currentStoryboardColor = p,
                                this.currentStoryboard = new SqA(p),
                                this.publish("STORYBOARD_CHANGED", this.currentStoryboard),
                                this.lastUpdateTime = (0,
                                _.AI)()) : (_.eP(new _.y7("Could not find color for timestamp: " + S,this.cinematicContainerRenderer)),
                                pw(this))
                            } else
                                pw(this);
                        else
                            Iyt(this, S)
                }
        }
        ;
        var Iyt = function(S, p) {
            var U = S.getStoryboardFrameIndex(p);
            if (U !== S.currentStoryboardIndex && U !== S.pendingStoryboardIndex) {
                p = Lyd(S, U);
                var Y = S.mosaics.get(p);
                Y ? Y.loaded && M2o(S, U) : (S.pendingStoryboardIndex = U,
                U = new trT(p),
                S.mosaics.set(p, U),
                U.resolver.promise.then(function() {
                    if (!S.isDisposed() && !S.paused && !isNaN(S.pendingStoryboardIndex)) {
                        var L = Lyd(S, S.pendingStoryboardIndex);
                        if (L) {
                            var I;
                            (I = S.mosaics.get(L)) != null && I.loaded && M2o(S, S.pendingStoryboardIndex)
                        } else
                            pw(S)
                    }
                }))
            }
        };
        SE.prototype.getStoryboardFrameIndex = function(S) {
            var p = this.isShorts() && this.playerApi.getNumberOfStoryboardLevels() > 1 ? 1 : 0;
            return this.playerApi.getStoryboardFrameIndex(S, p)
        }
        ;
        SE.prototype.getStoryboardFrame = function(S) {
            var p = this.isShorts() && this.playerApi.getNumberOfStoryboardLevels() > 1 ? 1 : 0, U, Y;
            return ((Y = (U = this.playerApi).getStoryboardFrame) == null ? void 0 : Y.call(U, S, p)) || null
        }
        ;
        SE.prototype.pause = function() {
            this.lastUpdateTime = NaN;
            this.paused = !0
        }
        ;
        var YH = function(S, p, U, Y) {
            Y = Y === void 0 ? !1 : Y;
            _.hY.call(this);
            this.cinematicContainerRenderer = p;
            this.playerApi = U;
            this.theater = this.fullscreen = !1;
            var L;
            this.ambientLightThemeEnabled = !Y && !!(p == null ? 0 : (L = p.config) == null ? 0 : L.enableInLightTheme);
            this.ambientFullscreenEnabled = Y && _.y("web_cinematic_fullscreen");
            this.container = document.createElement("div");
            S.appendChild(this.container);
            var I;
            if (_.y("web_cinematic_theater_mode") || _.y("web_cinematic_fullscreen") || (p == null ? 0 : (I = p.config) == null ? 0 : I.enableInLightTheme))
                this.ambientV2Container = document.createElement("div"),
                this.container.appendChild(this.ambientV2Container);
            dqE(this);
            S = this.ambientV2Container || this.container;
            this.backCanvas = iiJ();
            this.frontCanvas = iiJ();
            S.appendChild(this.backCanvas.element);
            S.appendChild(this.frontCanvas.element);
            this.storyboardManager = new SE(p,this.playerApi);
            _.Hm(this, this.storyboardManager);
            this.addEventListeners();
            V2J(this) ? TUX(this, 100 + U_(this) * 3 * 2, 100 + U_(this) * 3 * 2) : WyL(this);
            OVJ(this, this.storyboardManager.currentStoryboard)
        };
        _.r(YH, _.hY);
        var dqE = function(S) {
            S.ambientV2Container ? kZL(S) : (_.R3(S.container, {
                position: "absolute",
                top: "0",
                left: "0",
                right: "0",
                bottom: "0",
                "pointer-events": "none",
                transform: "scale(" + HVJ(S) + ", " + DqT(S) + ")"
            }),
            NFX() && _.R3(S.container, "filter", "blur(" + _.pA("cinematic_watch_css_filter_blur_strength", 40) + "px)"))
        }
          , kZL = function(S) {
            if (S.ambientV2Container) {
                var p = S.playerApi.getVideoAspectRatio();
                _.R3(S.container, {
                    "aspect-ratio": "" + p,
                    "max-width": "100%",
                    height: "100%",
                    margin: "0 auto",
                    display: "flex",
                    "flex-direction": "column",
                    "justify-content": "center",
                    "pointer-events": "none"
                });
                _.R3(S.ambientV2Container, {
                    "aspect-ratio": "" + p,
                    position: "relative",
                    "max-height": "100%",
                    "max-width": "100%",
                    "pointer-events": "none",
                    transform: "scale(" + HVJ(S) + ", " + DqT(S) + ")"
                });
                NFX() && _.R3(S.ambientV2Container, "filter", "blur(" + _.pA("cinematic_watch_css_filter_blur_strength", 40) + "px)")
            }
        };
        YH.prototype.setFullscreen = function(S, p) {
            this.fullscreen = S;
            this.theater = !!p;
            dqE(this);
            if (this.ambientFullscreenEnabled || this.ambientLightThemeEnabled)
                this.backCanvas.context.clearRect(0, 0, this.backCanvas.element.width, this.backCanvas.element.height),
                S = this.storyboardManager.currentStoryboard,
                zQo(this),
                S && QNT(this, S)
        }
        ;
        YH.prototype.addEventListeners = function() {
            var S = this
              , p = this.storyboardManager.subscribe("STORYBOARD_CHANGED", function(L) {
                OVJ(S, L)
            })
              , U = this.storyboardManager.subscribe("STORYBOARD_SIZE_CHANGED", function() {
                WyL(S)
            });
            this.addOnDisposeCallback(function() {
                S.storyboardManager.unsubscribeByKey(p);
                S.storyboardManager.unsubscribeByKey(U)
            });
            if (this.ambientV2Container) {
                var Y = function() {
                    kZL(S)
                };
                this.playerApi.addEventListener("onVideoDataChange", Y);
                this.addOnDisposeCallback(function() {
                    S.playerApi.removeEventListener("onVideoDataChange", Y)
                })
            }
        }
        ;
        var WyL = function(S) {
            if (!V2J(S)) {
                var p = S.storyboardManager.currentStoryboardSize;
                isNaN(p.width) || isNaN(p.height) || TUX(S, Number(p.width) + U_(S) * 3 * 2, Number(p.height) + U_(S) * 3 * 2)
            }
        }
          , TUX = function(S, p, U) {
            S.backCanvas.element.width = p;
            S.backCanvas.element.height = U;
            S.frontCanvas.element.width = p;
            S.frontCanvas.element.height = U
        }
          , V2J = function(S) {
            return S.cinematicContainerRenderer.presentationStyle === "CINEMATIC_CONTAINER_PRESENTATION_STYLE_DYNAMIC_SINGLE_COLOR"
        }
          , OVJ = function(S, p, U) {
            U = U === void 0 ? !1 : U;
            var Y = S.frontCanvas.element.getAnimations()[0];
            Y ? (Y.pause(),
            S.backCanvas.context.globalAlpha = Number(getComputedStyle(S.frontCanvas.element).opacity),
            S.frontCanvas.element.style.opacity = "0",
            Y.finish()) : S.backCanvas.context.globalAlpha = 1;
            S.backCanvas.context.drawImage(S.frontCanvas.element, 0, 0, S.backCanvas.element.width, S.backCanvas.element.height);
            zQo(S);
            p && QNT(S, p);
            p = p ? S.cinematicContainerRenderer.config.animationConfig.crossfadeDurationMs : _.pA("cinematic_watch_fade_out_duration", 500);
            Y = _.pA("cinematic_watch_transition_frame_rate") / 1E3;
            var L = {};
            Y && (L = {
                easing: "steps(" + Math.round(p * Y) + ")"
            });
            (U === void 0 ? 0 : U) || S.frontCanvas.element.animate([{
                opacity: 0
            }, {
                opacity: 1
            }], Object.assign({}, {
                duration: p,
                iterations: 1
            }, L));
            S.frontCanvas.element.style.opacity = "1"
        }
          , zQo = function(S) {
            var p = S.ambientLightThemeEnabled
              , U = S.ambientLightThemeEnabled || S.ambientFullscreenEnabled && !_.y("web_cinematic_fullscreen_v2");
            S.frontCanvas.context.fillStyle = S.theater && p || S.fullscreen && U ? "#000" : "#0f0f0f";
            NFX() || (S.frontCanvas.context.filter = "blur(0)");
            S.frontCanvas.context.fillRect(0, 0, S.frontCanvas.element.width, S.frontCanvas.element.height)
        }
          , QNT = function(S, p) {
            NFX() || (S.frontCanvas.context.filter = "blur(" + U_(S) + "px)");
            S.frontCanvas.context.globalAlpha = _.pA("cinematic_watch_effect_opacity", .4);
            var U = {
                offsetX: U_(S) * 3,
                offsetY: U_(S) * 3,
                theater: S.theater,
                fullscreen: S.fullscreen
            };
            V2J(S) && (U.width = 100,
            U.height = 100);
            p.drawToCanvas(S.frontCanvas.context, U);
            S.frontCanvas.context.globalAlpha = 1
        }
          , U_ = function(S) {
            var p;
            return (p = S.cinematicContainerRenderer.config.blurStrength) != null ? p : 5
        }
          , HVJ = function(S) {
            var p, U;
            if ((S.fullscreen || S.theater) && ((p = S.cinematicContainerRenderer.config) == null ? 0 : (U = p.watchFullscreenConfig) == null ? 0 : U.colorSourceWidthMultiplier))
                return S.cinematicContainerRenderer.config.watchFullscreenConfig.colorSourceWidthMultiplier;
            var Y;
            return (Y = S.cinematicContainerRenderer.config.colorSourceWidthMultiplier) != null ? Y : S.cinematicContainerRenderer.config.colorSourceSizeMultiplier
        }
          , DqT = function(S) {
            var p, U;
            if ((S.fullscreen || S.theater) && ((p = S.cinematicContainerRenderer.config) == null ? 0 : (U = p.watchFullscreenConfig) == null ? 0 : U.colorSourceHeightMultiplier))
                return S.cinematicContainerRenderer.config.watchFullscreenConfig.colorSourceHeightMultiplier;
            var Y;
            return (Y = S.cinematicContainerRenderer.config.colorSourceHeightMultiplier) != null ? Y : S.cinematicContainerRenderer.config.colorSourceSizeMultiplier
        };
        YH.prototype.disposeInternal = function() {
            _.hY.prototype.disposeInternal.call(this);
            this.container.remove()
        }
        ;
        YH.prototype.clear = function() {
            OVJ(this, void 0, !0)
        }
        ;
        YH.prototype.pause = function() {
            this.storyboardManager.pause()
        }
        ;
        var nIX;
        nIX = _.WD(function() {
            var S, p, U = !((p = (S = document.createElement("canvas")).getContext) == null || !p.call(S, "2d")), Y;
            S = !((Y = CSS) == null || !Y.supports("filter: blur(0)"));
            Y = !!Element.prototype.animate && !!Element.prototype.getAnimations;
            p = _.y("web_cinematic_fullscreen") || _.y("web_cinematic_theater_mode") || _.y("web_cinematic_light_theme") || !1;
            var L;
            return U && S && Y && (!p || !((L = CSS) == null || !L.supports("aspect-ratio: 1 / 1")))
        });
        _.Lw = function(S, p) {
            _.qS.call(this);
            this.container = S;
            this.playerApi = p;
            this.fullscreen = this.theater = this.settingEnabled = this.isDarkModeEnabled = this.wasAllowed = !1;
            this.prefersReducedMotionQuery = JbJ(this);
            this.update()
        }
        ;
        _.r(_.Lw, _.qS);
        _.ybt = function(S) {
            var p = document.documentElement.hasAttribute("dark");
            S.isDarkModeEnabled = p;
            S.update()
        }
        ;
        _.Kyi = function(S, p) {
            S.settingEnabled = p;
            S.update()
        }
        ;
        _.Lw.prototype.setFullscreen = function(S, p) {
            this.fullscreen = S;
            this.theater = !!p;
            this.update()
        }
        ;
        _.bVY = function(S, p) {
            S.cinematicContainerRenderer = p;
            S.cinematicContainerRenderer && (S.cinematicsVe = _.bZ(S.isShorts() ? 227858 : 159022),
            _.pa(_.HX(), S.cinematicsVe),
            S.loggingClientData = {
                watchCinematicContainerData: {
                    presentationStyle: S.cinematicContainerRenderer.presentationStyle
                }
            });
            S.update()
        }
        ;
        _.Lw.prototype.isShorts = function() {
            var S, p;
            return ((S = this.cinematicContainerRenderer) == null ? void 0 : (p = S.config) == null ? void 0 : p.pageType) === "CINEMATIC_CONTAINER_PAGE_TYPE_SHORTS"
        }
        ;
        _.Lw.prototype.isAllowed = function() {
            var S;
            if (S = nIX()) {
                var p, U, Y;
                S = ((U = this.cinematicContainerRenderer) == null ? void 0 : U.presentationStyle) === "CINEMATIC_CONTAINER_PRESENTATION_STYLE_DYNAMIC_SINGLE_COLOR" && !((Y = this.cinematicContainerRenderer) == null || !Y.colorStore) || ((p = this.cinematicContainerRenderer) == null ? void 0 : p.presentationStyle) === "CINEMATIC_CONTAINER_PRESENTATION_STYLE_DYNAMIC_BLURRED"
            }
            if (S)
                if (_.y("web_cinematics_pausing")) {
                    var L, I;
                    S = this.isDarkModeEnabled || !!((L = this.cinematicContainerRenderer) == null ? 0 : (I = L.config) == null ? 0 : I.enableInLightTheme)
                } else {
                    var V, W;
                    L = !!((V = this.cinematicContainerRenderer) == null ? 0 : (W = V.config) == null ? 0 : W.enableInLightTheme) && (_.g7("INNERTUBE_CLIENT_NAME") === "MWEB" || this.fullscreen || this.theater);
                    S = this.isDarkModeEnabled || L
                }
            if (V = S) {
                var O, D;
                V = !((D = (O = this.prefersReducedMotionQuery) == null ? void 0 : O.matches) != null && D)
            }
            return V
        }
        ;
        _.jNL = function(S) {
            (S = S.currentCinematicEffect) != null && (S = S.storyboardManager,
            S.paused = !1,
            Uqd(S),
            S.update())
        }
        ;
        _.Lw.prototype.isEnabled = function() {
            return this.isAllowed() && this.settingEnabled
        }
        ;
        var JbJ = function(S) {
            if (!_.y("web_watch_cinematics_preferred_reduced_motion_default_disabled") && window.matchMedia) {
                var p = window.matchMedia("(prefers-reduced-motion: reduce)")
                  , U = function() {
                    S.update()
                };
                p.addListener(U);
                S.addOnDisposeCallback(function() {
                    p.removeListener(U)
                });
                return p
            }
        };
        _.Lw.prototype.update = function() {
            this.isAllowed() !== this.wasAllowed && (this.wasAllowed = this.isAllowed(),
            this.publish("CINEMATICS_ALLOWED_CHANGED", this.wasAllowed));
            if (this.isEnabled()) {
                var S = this.cinematicContainerRenderer;
                this.currentCinematicEffect || (this.currentCinematicEffect = new YH(this.container,S,this.playerApi,this.isDarkModeEnabled),
                _.Hm(this, this.currentCinematicEffect));
                this.currentCinematicEffect.setFullscreen(this.fullscreen, this.theater);
                var p = this.currentCinematicEffect;
                p.cinematicContainerRenderer !== S && (p.cinematicContainerRenderer = S,
                pfT(p.storyboardManager, S),
                dqE(p));
                S = _.D9();
                _.zI(0, 194, !0);
                S.save();
                S = _.X$();
                this.cinematicsVe && S && _.fp(S, [this.cinematicsVe], this.loggingClientData)
            } else
                this.currentCinematicEffect && (S = _.X$(),
                this.cinematicsVe && S && _.Cp(S, [this.cinematicsVe], !1, this.loggingClientData),
                _.t9(this.currentCinematicEffect),
                this.currentCinematicEffect = void 0)
        }
        ;
        _.$u.Object.defineProperties(_.Lw.prototype, {
            TEST_ONLY: {
                configurable: !0,
                enumerable: !0,
                get: function() {}
            }
        });
```

---

## 6 License

MIT – do whatever you like, attribution appreciated.
Portions reverse-engineer behaviour visible in YouTube; no proprietary code is reproduced.

Enjoy the glow! 🌟
